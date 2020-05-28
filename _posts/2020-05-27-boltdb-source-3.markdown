---
layout: post
title:  "boltdb源码阅读4-Node"
date:   2020-05-27 05:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

上文介绍了所有页面的布局，本篇文章主要介绍和页一一对应的Node.Node用来表示B+树中的节点。
一句话来概括的话，就是Page是Node在磁盘的表示，Node是page在内存的表示。

来看一下Node的定义：
{% highlight go %}
// node represents an in-memory, deserialized page.
type node struct {
	bucket     *Bucket //是哪个Bucket下面的B+树节点
	isLeaf     bool //是不是叶子节点
	unbalanced bool //是不是不平衡的，在commit时候rebalance时会用到
	spilled    bool //有没有分裂过，在commit时候分裂时会用到
	key        []byte //节点的key
	pgid       pgid //对应的页号
	parent     *node //父节点
	children   nodes //子节点
	inodes     inodes //该节点内存的数据
}
{% endhighlight %}
其中inodes表示当前节点内存的数据：
{% highlight go %}
type inode struct {
	flags uint32 //节点类型，与bucketLeafFlag有关，判断leaf node是不是新的bucket的起点
	pgid  pgid //页号
	key   []byte
	value []byte
}

type inodes []inode
{% endhighlight %}
首先来看一下我们一直在强调的说法，Page和Node之间是序列化和反序列化的关系，来看一下具体实现：
Page到Node的转换：
{% highlight go %}
// read函数从page中读取内容初始化Node
func (n *node) read(p *page) {
	n.pgid = p.id //设置Node的页号
	n.isLeaf = ((p.flags & leafPageFlag) != 0) //设置是不是叶子节点
	n.inodes = make(inodes, int(p.count)) //先根据页里面的元素数量来构造inodes数组

	for i := 0; i < int(p.count); i++ { //然后一个个对inode进行赋值
		inode := &n.inodes[i]
		if n.isLeaf { //需要判断是不是叶子节点，因为叶子节点和非叶子节点构造不同
			elem := p.leafPageElement(uint16(i)) //从页中读取第i个元素
			inode.flags = elem.flags //将第i个元素赋值给inode
			inode.key = elem.key()
			inode.value = elem.value()
		} else {
			elem := p.branchPageElement(uint16(i)) //同上
			inode.pgid = elem.pgid
			inode.key = elem.key()
		}
		_assert(len(inode.key) > 0, "read: zero-length inode key")
	}

	if len(n.inodes) > 0 { //节点的key为其中最小的Key
		n.key = n.inodes[0].key
		_assert(len(n.key) > 0, "read: zero-length node key")
	} else {
		n.key = nil
	}
}

{% endhighlight %}
接下来是Node到Page的转换，思路基本一致：
{% highlight go %}
// write 把元素写到一个或多个页中
func (n *node) write(p *page) {
	// Initialize page.
	if n.isLeaf { //首先设置该页的flag表示是哪种节点
		p.flags |= leafPageFlag
	} else {
		p.flags |= branchPageFlag
	}

	if len(n.inodes) >= 0xFFFF {
		panic(fmt.Sprintf("inode overflow: %d (pgid=%d)", len(n.inodes), p.id))
	}
	p.count = uint16(len(n.inodes)) //设置count为节点数量

	// Stop here if there are no items to write.
	if p.count == 0 {
		return
	}

	// Loop over each item and write it to the page.
	// 定位到真正开始存数据的地方，需要便宜n.pageElementSize()*len(n.inodes),因为一开始要存那么多个数据的meta data
	b := (*[maxAllocSize]byte)(unsafe.Pointer(&p.ptr))[n.pageElementSize()*len(n.inodes):]
	for i, item := range n.inodes {
		_assert(len(item.key) > 0, "write: zero-length inode key")

		// Write the page element.
		if n.isLeaf {
			elem := p.leafPageElement(uint16(i)) //获取到第i个元素
			elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem))) //计算数据和meta之间的位置差
			elem.flags = item.flags //设置flag
			elem.ksize = uint32(len(item.key)) //设置key的大小
			elem.vsize = uint32(len(item.value)) //设置value的大小
		} else {
			elem := p.branchPageElement(uint16(i)) //与上面类似
			elem.pos = uint32(uintptr(unsafe.Pointer(&b[0])) - uintptr(unsafe.Pointer(elem)))
			elem.ksize = uint32(len(item.key))
			elem.pgid = item.pgid
			_assert(elem.pgid != p.id, "write: circular dependency occurred")
		}

		// If the length of key+value is larger than the max allocation size
		// then we need to reallocate the byte array pointer.
		//
		// See: https://github.com/boltdb/bolt/pull/335
		klen, vlen := len(item.key), len(item.value)
		if len(b) < klen+vlen {
			b = (*[maxAllocSize]byte)(unsafe.Pointer(&b[0]))[:]
		}

		// 然后开始把数据写到b数组， b数组一直保持当前写入的位置
		copy(b[0:], item.key)
		b = b[klen:]
		copy(b[0:], item.value)
		b = b[vlen:]
	}

	// DEBUG ONLY: n.dump()
}
{% endhighlight %}


讲完了node和page之间的相互转换，来看一下B+树节点的一些基本操作，包括增删改查，之后整个数据库的操作就是建立在node的这些基本操作之上的。

增或者改：
{% highlight go %}
// put inserts a key/value.
func (n *node) put(oldKey, newKey, value []byte, pgid pgid, flags uint32) {
	if pgid >= n.bucket.tx.meta.pgid {
		panic(fmt.Sprintf("pgid (%d) above high water mark (%d)", pgid, n.bucket.tx.meta.pgid))
	} else if len(oldKey) <= 0 {
		panic("put: zero-length old key")
	} else if len(newKey) <= 0 {
		panic("put: zero-length new key")
	}

	// 首先二分查找出该插入的位置
	index := sort.Search(len(n.inodes), func(i int) bool { return bytes.Compare(n.inodes[i].key, oldKey) != -1 })

	// Add capacity and shift nodes if we don't have an exact match and need to insert.
	exact := (len(n.inodes) > 0 && index < len(n.inodes) && bytes.Equal(n.inodes[index].key, oldKey))
	if !exact { //判断是不是正好找到了key,是的话就是改，不是的话就是增
		n.inodes = append(n.inodes, inode{})
		copy(n.inodes[index+1:], n.inodes[index:])
	}

	inode := &n.inodes[index]//然后设置key和value
	inode.flags = flags
	inode.key = newKey
	inode.value = value
	inode.pgid = pgid
	_assert(len(inode.key) > 0, "put: zero-length inode key")
}
{% endhighlight %}

删：
{% highlight go %}
// del removes a key from the node.
func (n *node) del(key []byte) {
	// 先二分查找的需要删除元素的下标
	index := sort.Search(len(n.inodes), func(i int) bool { return bytes.Compare(n.inodes[i].key, key) != -1 })

	// Exit if the key isn't found.
	if index >= len(n.inodes) || !bytes.Equal(n.inodes[index].key, key) { //没找到就直接退出
		return
	}

	// Delete inode from the node.
	n.inodes = append(n.inodes[:index], n.inodes[index+1:]...) //删除Index元素

	// Mark the node as needing rebalancing.
	n.unbalanced = true //删除元素之后需要标记为unbalanced，然后在commit过程时会看需不需要和隔壁合并起来
}
{% endhighlight %}

查：查的话其实都嵌套在上述操作里了，主要体现在其中的二分查找了，是增删改的基础。

上面讲述了在一个B+树节点里的增删查改，但是要注意的是，对整个数据库而言，我们进行的是对整颗B+树的增删改查，对Boltdb来讲，所有增删改查都是在bucket基础之上的，下篇文章将讲解bucket.
