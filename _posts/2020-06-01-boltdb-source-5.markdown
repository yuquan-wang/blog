---
layout: post
title:  "boltdb源码阅读6-Transaction"
date:   2020-06-01 12:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

本文主要探索boltdb是怎么实现事务(transaction)的，首先来看一下事务的常见用法：
{% highlight go %}
// Start a writable transaction.
tx, err := db.Begin(true)
if err != nil {
    return err
}
defer tx.Rollback()

// Use the transaction...
_, err := tx.CreateBucket([]byte("MyBucket"))
if err != nil {
    return err
}

// Commit the transaction and check for error.
if err := tx.Commit(); err != nil {
    return err
}
{% endhighlight %}

可以看到，由db.Begin()开始一个事务，最后需要调用commit()或者Rollback()函数来提交或者撤销。来看一下db.Begin()函数具体干了啥
{% highlight go %}
func (db *DB) Begin(writable bool) (*Tx, error) {
	if writable {
		return db.beginRWTx()
	}
	return db.beginTx()
}
{% endhighlight %}
根据传入的参数不同，返回一个可写或者只读的事务，因为只读的事务相对来说不会太复杂且不会修改数据，故这里重点研究可写事务，即db.beginRWTx():
{% highlight go %}
func (db *DB) beginRWTx() (*Tx, error) {
	// 如果数据库是以只读方式打开的，那么就不应该有可写的事务产生
	if db.readOnly {
		return nil, ErrDatabaseReadOnly
	}

	// 获取写锁，保证同一时刻只有一个线程在写
	db.rwlock.Lock()

	// 同事需要锁住metadata
	db.metalock.Lock()
	defer db.metalock.Unlock()

	// 如果数据库还没打开，则返回错误
	if !db.opened {
		db.rwlock.Unlock()
		return nil, ErrDatabaseNotOpen
	}

	// 创建一个事务
	t := &Tx{writable: true}
	t.init(db) //初始化事务
	db.rwtx = t

	// 释放已经结束的所有可读事务的页面， 这里先找出最小的事务id
	var minid txid = 0xFFFFFFFFFFFFFFFF
	for _, t := range db.txs {
		if t.meta.txid < minid {
			minid = t.meta.txid
		}
	}
	if minid > 0 {
		db.freelist.release(minid - 1) //在这里做释放工作
	}

	return t, nil
}
{% endhighlight %}
基本上所有工作都被封装在t.init(db)里的，在研究具体实现前，先来看下事务的定义：
{% highlight go %}
type Tx struct {
	writable       bool //本次事务是否可写
	managed        bool //暂时不用管
	db             *DB //对应的数据库对象
	meta           *meta //meta data指针
	root           Bucket //根bucket
	pages          map[pgid]*page //pageid 和Page的缓存
	stats          TxStats //用来统计状态用
	commitHandlers []func() //暂时不用管
	WriteFlag int //暂时不用管
}
{% endhighlight %}
然后再来看t.init(db)里都是怎么设置这些变量的：
{% highlight go %}
func (tx *Tx) init(db *DB) {
	tx.db = db //首先设置db对象
	tx.pages = nil //缓存刚初始化的时候肯定为空

	tx.meta = &meta{}
	db.meta().copy(tx.meta) //把db.meta复制到tx.meta里，不能直接用等号，因为会被修改

	// Copy over the root bucket.
	tx.root = newBucket(tx)
	tx.root.bucket = &bucket{}
	*tx.root.bucket = tx.meta.root //root bucket需设置为meta里面记录的root

	// 如果是可写事务，则事务id需要+1
	if tx.writable {
		tx.pages = make(map[pgid]*page)
		tx.meta.txid += txid(1)
	}
}
{% endhighlight %}
事务的初始化过程基本如上所示，也没啥黑科技，在事务初始化之后，就是一系列读取数据库的操作，在这些操作之后，用户需要调用commit()函数来提交这些操作，这个函数需要保证数据库的ACID特性，即：
- atomicity: 原子性
- consistentcy: 一致性
- isolation: 隔离性
- duration:持久性

来看一下具体commit 函数里做了啥（删除了一些收集状态的代码，因其不是重点）：
{% highlight go %}
func (tx *Tx) Commit() error {
	//首先是做一系列的检查操作看事务可不可以提交
	_assert(!tx.managed, "managed tx commit not allowed")
	if tx.db == nil {
		return ErrTxClosed
	} else if !tx.writable {
		return ErrTxNotWritable
	}

	// 对B+树里面进行过删除操作的节点，调用rebalance()函数，进行树的平衡调整
	// 主要是怕删除过的节点内元素太少，会把该节点合并到其他节点
	tx.root.rebalance()

	// 然后对根节点调用分裂函数，主要是怕上面rebalance()之后，有的节点有太多元素，需要进行合理分裂
	if err := tx.root.spill(); err != nil {
		tx.rollback() //分裂失败则进行回滚操作
		return err
	}

	// Free the old root bucket.
	tx.meta.root.root = tx.root.root
	opgid := tx.meta.pgid

	// 释放freelist并且重新申请新的页
	tx.db.freelist.free(tx.meta.txid, tx.db.page(tx.meta.freelist))
	p, err := tx.allocate((tx.db.freelist.size() / tx.db.pageSize) + 1)
	if err != nil {
		tx.rollback()
		return err
	}
	//先把新的freelist写到刚申请的p里面
	if err := tx.db.freelist.write(p); err != nil {
		tx.rollback() //写出错就进行回滚操作
		return err
	}
	tx.meta.freelist = p.id //设置内存里的meta.freelist为新p的id

	// 如果需要增长数据库大小
	if tx.meta.pgid > opgid {
		if err := tx.db.grow(int(tx.meta.pgid+1) * tx.db.pageSize); err != nil {
			tx.rollback() //数据库增加大小失败，则回滚
			return err
		}
	}

	// 把B+ 树分裂产生的脏页写入磁盘，这里其实内存中是有多个版本的B+树的
	if err := tx.write(); err != nil {
		tx.rollback() //写入错误则进行回滚
		return err
	}

	// 最后把meta写入到磁盘，这是最关键的，如果这一步失败，那么相当于就没有写入成功，因为之后的操作还会读取老的metadata来读取数据
	if err := tx.writeMeta(); err != nil {
		tx.rollback() //写入失败则进行回滚
		return err
	}

	// 关闭事务
	tx.close()

	return nil
}

{% endhighlight %}

总结一下：
在事务的commit()函数里面，一共做了以下事情：
- 树的rebalance,主要是检查删除过的节点，如果所含元素过少，则并到其他节点
- 树的分裂，上述步骤之后加上节点的增删改查，有的节点所含元素数量过多，需要进行分裂来满足B+树的要求
- 先把freelist写到新的页上
- 在把分裂得到的脏页写入磁盘
- 最后把meta页写入磁盘，只有这步结束后，才算事务成功提交，不然的话，之后的操作还是读取老metadata,而上述的步骤均会被回滚。

来看看ACID的几个特性是怎么保证的：
- atomicity: 原子性，只有meta页写入成功才算成功，不然就会回滚
- consistentcy: 一致性，某个特定时刻只有一个写事务会操作数据。
- isolation: 隔离性，boltdb 支持多个读事务与一个写事务同时执行，写事务提交时会释放旧的 page，分配新的 page，只要确保分配的新 page 不会是其他读事务使用到的就能实现 Isolation。在写事务提交时，释放的老 page 有可能还会被其他读事务访问到，不能立即用于下次分配，所以放在 freelist.pending 中， 只有确保没有读事务会用到时，才将相应的 pending page 放入 freelist.ids 中用于分配， 也就是说读写事务时间不会相互影响。
- duration:持久性 //改动写入磁盘后将一直保持

至于回滚操作，基本上都是freelist的操作：
{% highlight go %}
func (tx *Tx) rollback() {
	if tx.db == nil {
		return
	}
	if tx.writable {
		tx.db.freelist.rollback(tx.meta.txid)
		tx.db.freelist.reload(tx.db.page(tx.db.meta().freelist))
	}
	tx.close()
}
{% endhighlight %}
具体做了什么等下篇文章探索了freelist在进行解释。
