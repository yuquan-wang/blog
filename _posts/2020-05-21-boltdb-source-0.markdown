---
layout: post
title:  "boltdb源码阅读1-简介"
date:   2020-05-21 01:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

> Package bolt implements a low-level key/value store in pure Go. It supports
fully serializable transactions, ACID semantics, and lock-free MVCC with
multiple readers and a single writer. Bolt can be used for projects that
want a simple data store without the need to add large dependencies such as
Postgres or MySQL.
Bolt is a single-level, zero-copy, B+tree data store. This means that Bolt is
optimized for fast read access and does not require recovery in the event of a
system crash. Transactions which have not finished committing will simply be
rolled back in the event of a crash.
The design of Bolt is based on Howard Chu's LMDB database project.

如上面引用所述, boltdb是用pure实现的非常精简的key-value数据库，说他精简是因为相比mysql等其他数据库， 它缺少了很多功能，但很多时候我们并不需要那些功能， 而boltdb所提供的完全序列化事务，ACID支持及无锁的MVCC支持，往往对使用者来说已经足够，更为重要的事，它的代码非常简洁，适合用来学习具体数据库是怎么实现的。

boltdb的使用方式相当之简单，因为是个key-value数据库，基本上只有get和set的操作，下面是读写数据库的一个样例：
{% highlight go %}
func main() {
   db, err := bolt.Open("my.db", 0600, nil) //打开一个数据库，注意boltdb是单文件数据库
   if err != nil {
      log.Fatal(err)
   }
   defer db.Close()

   err = db.Update(func(tx *bolt.Tx) error {
      b, err := tx.CreateBucket([]byte("MyBucket"))
      if err != nil {
         return err
      }
      _ = b.Put([]byte("key"), []byte("value")) // 设置key和value
      return nil
   })

   err = db.View(func(tx *bolt.Tx) error {
      b := tx.Bucket([]byte("MyBucket"))
      v:= b.Get([]byte("key")) //读取key和value
      fmt.Printf("The key is: %s\n", v)
      return nil
   })
}
{% endhighlight %}

以上展示了boltdb的基本用法，1）打开或者创建数据库 2）设置key-value 3)根据key获得value,用法是相当之简单。

来看一下源码的目录结构（除去平台相关的代码和测试文件）:
- db.go: 程序的入口，对整个db的封装
- node.go: B+树节点的封装，与page一一对应
- page.go: 存储B+树节点的磁盘页，与B+树节点一一对应
- freelist.go: 空闲页， 用来表示哪些页可以被直接再次利用
- bucket.go: 桶，和mysql里面的表概念类似，boltdb的key和value都存在桶下面
- cursor.go: B+树查找时会用到
- tx.go: 对事务的封装

基本上就只有上述核心代码，简洁但又实现了数据库的基本需求，且性能良好（体现在很多项目以它为基础进行二次开发），故值得一学，接下来的几篇博客将会一一解析上述文件。
