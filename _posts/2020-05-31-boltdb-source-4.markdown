---
layout: post
title:  "boltdb源码阅读5-Bucket和Cursor"
date:   2020-05-31 12:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

boltdb的所有操作均通过bucket来进行，bucket可以理解为mysql里面的表，我们要选定表之后才能进行各种增删改查操作，boltdb也是一样的，所有的key value是基于bucket的，下面是它的经典用法：
{% highlight go %}
db.Update(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket")) //首先先得到bucket
	err := b.Put([]byte("answer"), []byte("42")) //基于bucket才能设值和取值
	return err
})
{% endhighlight %}

所以本文来学习一下bucket内部是怎么运作的，每个Bucket是一颗完整的B+树，本文也可以看做boltdb是怎么在完整的B+树上做操作的，注意上文主要学习了在一个B+树节点中怎么做增删改查的

来看一下Bucket的定义：
{% highlight go %}
type Bucket struct {
	*bucket //下面有定义，来指明根节点所在的页号
	tx       *Tx                // 与事务绑定
	buckets  map[string]*Bucket // 子bucket的缓存
	page     *page              // inline page， 如果一个bucket比较小的话
	rootNode *node              // B+树根节点
	nodes    map[pgid]*node     // 页号到节点的缓存
	FillPercent float64 //决定B+树节点要不要分裂
}

type bucket struct {
	root     pgid   // 根节点页号
	sequence uint64 // monotonically incrementing, used by NextSequence()
}
{% endhighlight %}

接下来就是看一下Bucket的两个主要操作了，一个是Put()，一个是Get(),是的，boltdb操作相当之简单，主要只有这两个， 来看一下Put()函数：
{% highlight go %}
func (b *Bucket) Put(key []byte, value []byte) error {
	//首先是做一系列的检查，来看输入合不合法
	if b.tx.db == nil {
		return ErrTxClosed
	} else if !b.Writable() {
		return ErrTxNotWritable
	} else if len(key) == 0 {
		return ErrKeyRequired
	} else if len(key) > MaxKeySize {
		return ErrKeyTooLarge
	} else if int64(len(value)) > MaxValueSize {
		return ErrValueTooLarge
	}

	// 将游标移到适当的位置
	c := b.Cursor()
	k, _, flags := c.seek(key) //查找key

	// 如果value是bucket的话则返回错误
	if bytes.Equal(key, k) && (flags&bucketLeafFlag) != 0 {
		return ErrIncompatibleValue
	}

	// 调用node的put()函数来插入数据，node的put()函数上文已经介绍过
	key = cloneBytes(key)
	c.node().put(key, key, value, 0, 0)

	return nil
}
{% endhighlight %}

Get()函数更为简单：
{% highlight go %}
func (b *Bucket) Get(key []byte) []byte {
	k, v, flags := b.Cursor().seek(key) //利用游标来寻找key

	// 如果value是bucket类型的话就返回空
	if (flags & bucketLeafFlag) != 0 {
		return nil
	}

	// 如果key不相等的话，那就是还没有对应的元素
	if !bytes.Equal(key, k) {
		return nil
	}
	return v
}
{% endhighlight %}
上面两段代码均很简单，简单的让人以为它不干实事，事实上，它确实也就是个空架子，主要都是通过游标来寻找到对应节点，然后做增删改查的，那么有必要看下游标是具体怎么操作的。


先来看下Cursor的定义：
{% highlight go %}
type Cursor struct {
	bucket *Bucket
	stack  []elemRef
}
{% endhighlight %}
定义很简单，只有两个元素，一个是对应的Bucket,还有一个则是查找过程中对应的元素栈。

Bucket的Put()和Get()函数，都主要依赖Cursor的seek()函数，所以来看看这里面发生了啥：
{% highlight go %}
func (c *Cursor) Seek(seek []byte) (key []byte, value []byte) {
	k, v, flags := c.seek(seek)

	// 如果元素栈栈顶超出了这页能表示的范围，则返回next()
	if ref := &c.stack[len(c.stack)-1]; ref.index >= ref.count() {
		k, v, flags = c.next()
	}

	if k == nil { //key不能为空
		return nil, nil
	} else if (flags & uint32(bucketLeafFlag)) != 0 { //value不能为bucket
		return k, nil
	}
	return k, v
}
{% endhighlight %}
可以看到，所有的事情基本上都有seek()函数处理了：
{% highlight go %}
// seek moves the cursor to a given key and returns it.
// If the key does not exist then the next key is used.
func (c *Cursor) seek(seek []byte) (key []byte, value []byte, flags uint32) {
	_assert(c.bucket.tx.db != nil, "tx closed")

	// 首先先清空栈，从头开始查询
	c.stack = c.stack[:0]
	c.search(seek, c.bucket.root) //通过search函数进行查找，并将中间遍历过的元素放入栈中
	ref := &c.stack[len(c.stack)-1] //栈顶便是最后找到的key value

	// If the cursor is pointing to the end of page/node then return nil.
	if ref.index >= ref.count() {
		return nil, nil, 0
	}

	return c.keyValue()
}
{% endhighlight %}
感觉马上就要接近真相了，这里已经出现了元素栈顶的用法，显而易见，search()函数承包了所有的逻辑：
{% highlight go %}
// 使用二分查找在页或者节点中进行查找
func (c *Cursor) search(key []byte, pgid pgid) {
	p, n := c.bucket.pageNode(pgid) //首先获取页号
	if p != nil && (p.flags&(branchPageFlag|leafPageFlag)) == 0 {
		panic(fmt.Sprintf("invalid page type: %d: %x", p.id, p.flags))
	}
	e := elemRef{page: p, node: n}
	c.stack = append(c.stack, e) //将当前元素放入元素栈中

	// 如果是个叶子节点的话
	if e.isLeaf() {
		c.nsearch(key)
		return
	}

	if n != nil { //对于非叶子节点来说，如果page已经反序列化成node了，就直接在node里面查找
		c.searchNode(key, n)
		return
	}
	c.searchPage(key, p) //不然的话就通过Page来查找
}
{% endhighlight %}


上面出现了三个重要的函数：
- nsearch()
- searchNode()
- searchPage()

我们一个个来解析，首先是第一个nsearch(),是在叶子节点中进行的搜索：
{% highlight go %}
// nsearch 从栈顶元素中搜索叶子节点
func (c *Cursor) nsearch(key []byte) {
	e := &c.stack[len(c.stack)-1]
	p, n := e.page, e.node

	// 如果页已经反序列化成节点
	if n != nil {
		index := sort.Search(len(n.inodes), func(i int) bool {
			return bytes.Compare(n.inodes[i].key, key) != -1
		})
		e.index = index
		return
	}

	// 不然的话直接搜索Page
	inodes := p.leafPageElements()
	index := sort.Search(int(p.count), func(i int) bool {
		return bytes.Compare(inodes[i].key(), key) != -1
	})
	e.index = index
}
{% endhighlight %}

然后来看第二个searchNode(),是在B+树非叶子节点中进行的搜索（前提是Page已经反序列化成Node了）：
{% highlight go %}
func (c *Cursor) searchNode(key []byte, n *node) {
	var exact bool
	index := sort.Search(len(n.inodes), func(i int) bool { //二分查找
		ret := bytes.Compare(n.inodes[i].key, key)
		if ret == 0 {
			exact = true
		}
		return ret != -1
	})
	if !exact && index > 0 {
		index--
	}
	c.stack[len(c.stack)-1].index = index //设置栈顶元素的下标

	// 然后递归调用search函数来进行查找，主要是search()并不是searchNode和searchPage
	c.search(key, n.inodes[index].pgid)
}
{% endhighlight %}

最后来看第三个函数searchPage(), 是在B+树非叶子节点中进行的搜索（前提是Page并没有被反序列化成Node）：
{% highlight go %}
func (c *Cursor) searchPage(key []byte, p *page) {
	// Binary search for the correct range.
	inodes := p.branchPageElements()

	var exact bool
	index := sort.Search(int(p.count), func(i int) bool { //二分查找
		ret := bytes.Compare(inodes[i].key(), key)
		if ret == 0 {
			exact = true
		}
		return ret != -1
	})
	if !exact && index > 0 {
		index--
	}
	c.stack[len(c.stack)-1].index = index

	// 与searchNode()类似
	c.search(key, inodes[index].pgid)
}
{% endhighlight %}
searchPage()和searchNode()函数异常相似，只不过一个是从内存中的inodes里面查找，一个是直接从磁盘中进行查找。

这样整个查找的思路就都清晰了，不管是叶子节点还是非叶子节点，都是通过二分查找来做的，区别只是在于是直接查找Node，还是说去查找Page.

下面文章讲一下事务这个概念，来看看boltdb是怎么保持ACID的，我个人是冲着学习这个来看的源码。
