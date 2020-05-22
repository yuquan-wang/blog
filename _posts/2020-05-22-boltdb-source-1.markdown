---
layout: post
title:  "boltdb源码阅读2-DB"
date:   2020-05-22 01:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

boltdb创建或者打开数据库的方式非常简单，如下：
{% highlight go %}
func main() {
   db, err := bolt.Open("my.db", 0600, nil) //打开一个数据库，注意boltdb是单文件数据库
   if err != nil {
      log.Fatal(err)
   }
   defer db.Close()
   ...
}
{% endhighlight %}

我们直接通过bolt.Open函数来看一下具体实现(省去部分细节)：
{% highlight go %}
func Open(path string, mode os.FileMode, options *Options) (*DB, error) {
	var db = &DB{opened: true} //首先创建数据库对象，设置opened为true

	// 做一些基本的初始化工作
	if options == nil {
		options = DefaultOptions
	}
	//省去其他初始化的设置

	// 设置db的路径文件，注意boltdb是单文件数据库
	db.path = path
	var err error
	if db.file, err = os.OpenFile(db.path, flag|os.O_CREATE, mode); err != nil {
    //打开数据库文件，若无文件存在则创建文件
		_ = db.close()
		return nil, err
	}

	// 加文件锁，防止多个进程同时对数据库进行写操作
	if err := flock(db, mode, !db.readOnly, options.Timeout); err != nil {
		_ = db.close()
		return nil, err
	}

	// 对数据库进行初始化
	if info, err := db.file.Stat(); err != nil {
		return nil, err
	} else if info.Size() == 0 {
		// 若发现数据库文件没有初始化过，调用init()方法进行初始化
		if err := db.init(); err != nil {
			return nil, err
		}
	} else {
		// 若发现数据库文件已经初始化过了，则读取第0个页的数据（第0页和第1页为mata页，用来存一些类似页大小的数据，之后会详细介绍）
		var buf [0x1000]byte
		if _, err := db.file.ReadAt(buf[:], 0); err == nil {
			m := db.pageInBuffer(buf[:], 0).meta()
			if err := m.validate(); err != nil {
				// 若validate出错，则将页大小设置会系统默认页大小
				db.pageSize = os.Getpagesize()
			} else {
        //设置页大小为meta页里面的数据
				db.pageSize = int(m.pageSize)
			}
		}
	}

	// 初始化用来分配页面的pool
	db.pagePool = sync.Pool{
		New: func() interface{} {
			return make([]byte, db.pageSize)
		},
	}

	// mmap(内存映射)数据文件，boltdb用mmap来读取文件来优化性能(比一般文件读取少一次copy操作)，但是用一般的文件写操作来修改磁盘页面
	if err := db.mmap(options.InitialMmapSize); err != nil {
		_ = db.close()
		return nil, err
	}

	// 设置页面空闲列表
	db.freelist = newFreelist()
	db.freelist.read(db.page(db.meta().freelist))

	// Mark the database as opened and return.
	return db, nil
}
{% endhighlight %}

具体每一步都已经详细标注出来，其中有些函数值得细看一下，首先来看一下flock()函数：
{% highlight go %}
// flock acquires an advisory lock on a file descriptor.
func flock(db *DB, mode os.FileMode, exclusive bool, timeout time.Duration) error {
	var t time.Time
	for {
		// If we're beyond our timeout then return an error.
		// This can only occur after we've attempted a flock once.
		if t.IsZero() {
			t = time.Now()
		} else if timeout > 0 && time.Since(t) > timeout {
			return ErrTimeout
		}
		flag := syscall.LOCK_SH  //首先设置为共享锁
		if exclusive {
			flag = syscall.LOCK_EX //在此种情况下设置为排他锁
		}

		// Otherwise attempt to obtain an exclusive lock.
		err := syscall.Flock(int(db.file.Fd()), flag|syscall.LOCK_NB)
		if err == nil {
			return nil
		} else if err != syscall.EWOULDBLOCK {
			return err
		}

		// Wait for a bit and try again.
		time.Sleep(50 * time.Millisecond)
	}
}
{% endhighlight %}
flock()函数也比较简单，就是对数据库文件上个锁，如果以非readonly模式打开数据库，则对数据库文件加上一个排他锁，这样其他线程/进程如果也要对文件进行改动的话，就会被block住，除非等当前改动结果或者等timeout.

然后值得一看的是init()函数，让我们来看看具体初始化工作中设置了写啥：
{% highlight go %}
// init creates a new database file and initializes its meta pages.
func (db *DB) init() error {
	// 首先设置页大小
	db.pageSize = os.Getpagesize()

	// 然后生成两个meta页，这两个meta页是控制背后不同版本B+树的关键
	buf := make([]byte, db.pageSize*4)
	for i := 0; i < 2; i++ {
		p := db.pageInBuffer(buf[:], pgid(i))
		p.id = pgid(i)
		p.flags = metaPageFlag

		// Initialize the meta page.
		m := p.meta()
		m.magic = magic
		m.version = version
		m.pageSize = uint32(db.pageSize)
		m.freelist = 2 //说明空闲页表在第2页（第0和第1页为meta页）
		m.root = bucket{root: 3} //表示最上层的bucket在page 3(数据页从第三页开始，仅仅是初始化的时候，之后空闲页可以不在第二页，数据页就可以从第二页开始)
		m.pgid = 4 //这边的pgid表示目前一共有多少页
		m.txid = txid(i)
		m.checksum = m.sum64()
	}

	// 在第2页（0 based）创建空闲页表
	p := db.pageInBuffer(buf[:], pgid(2))
	p.id = pgid(2)
	p.flags = freelistPageFlag
	p.count = 0

	// 在第3页（0 based）创建数据页面(B+树叶子节点)
	p = db.pageInBuffer(buf[:], pgid(3))
	p.id = pgid(3)
	p.flags = leafPageFlag
	p.count = 0

	// 把数据写到磁盘上
	if _, err := db.ops.writeAt(buf, 0); err != nil {
		return err
	}
	if err := fdatasync(db); err != nil {
		return err
	}

	return nil
}
{% endhighlight %}
具体的逻辑已在文中注释出来，初始化工作主要是做了如下操作：
- 设置页大小为系统默认页面大小（一般是4k大小）
- 设置两个meta页，之后会讲为什么需要两个meta页，和无锁MVCC有关
- 设置空闲页表
- 设置数据页(B+树叶子节点，数据页会构成一个完整的B+树)

用图来表示的话就如下图所示：
![png1]({{ site.baseurl }}/assets/img/boltdb/db-init.png){: .center-image }

下一篇文章便是学习具体每一个页面是如何布局的。
