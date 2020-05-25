---
layout: post
title:  "boltdb源码阅读3-Page"
date:   2020-05-22 05:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

上文介绍了boltdb在初始化的时候会构造4个页,其中第0和第1个为meta页， 第2个为空闲列表页，第3个为B+树叶子节点页（可以看成是数据页），如下图所示， 但具体每种页的布局怎么样，还有待在此篇文章中进行学习。
![png1]({{ site.baseurl }}/assets/img/boltdb/db-init.png){: .center-image }

来看一个Page的定义
{% highlight go %}
type page struct {
	id       pgid  //定义该页属于逻辑上的第几页
	flags    uint16 //表示是哪种页面
	count    uint16 //存储了多少个数据
	overflow uint32 //表示是否超过一页，需要和其他的页连在一起表示一个完整的数据结构
	ptr      uintptr //具体数据存储开始的地方
}
{% endhighlight %}
首先可以看到page里面用flags来区分不同的页面，来看一下都有哪些页面：
{% highlight go %}
const (
	branchPageFlag   = 0x01 //B+树非叶子节点对应的页
	leafPageFlag     = 0x02 //B+树叶子节点对应的页
	metaPageFlag     = 0x04 //mata页
	freelistPageFlag = 0x10 //空闲列表页
)
{% endhighlight %}
一共分为4种页面，其中metaPageFlag对应之前说的第0页和第1页freelistPageFlag对应第2页，leafPageFlag对应第3页，因为初始化时只构造了B+树叶子节点，并没有构造非叶子节点，故暂时没有页与branchPageFlag对应。

下面我们一个一个来解析不同的页面，首先来看meta页（注意代码在db.go里）：
{% highlight go %}
type meta struct {
	magic    uint32 //神奇的魔法数字， 不重要
	version  uint32 //版本，也不重要
	pageSize uint32 //页的大小，在上一篇文章中提到过，是存储数据库页大小的地方，重要
	flags    uint32 //暂时好像不重要
	root     bucket //根bucket
	freelist pgid //空闲页表在第几页
	pgid     pgid //数据库一共有多少页
	txid     txid //最新的事务id
	checksum uint64 //不重要
}
{% endhighlight %}
meta的结构如上所示，主要记录了当前数据库页大小，根bucket，空闲页表和最新事务id等信息，来看一下它是怎么获取的：
{% highlight go %}
// meta returns a pointer to the metadata section of the page.
func (p *page) meta() *meta {
	return (*meta)(unsafe.Pointer(&p.ptr))
}
{% endhighlight %}
可以看到是直接把p.ptr(也就是数据部分)直接转成meta结构，那么meta页的结构也就比较清晰了， 如下图所示：
![png2]({{ site.baseurl }}/assets/img/boltdb/meta-layout.png){: .center-image }

接下来看一个freelist 页：
{% highlight go %}
type freelist struct {
	ids     []pgid          // 所有空闲的且可被分配的页号
	pending map[txid][]pgid // 马上从事务txid可以被释放的页号
	cache   map[pgid]bool   // 缓存，用来加速，暂时不管
}
{% endhighlight %}
freelist页暂时就先分析到这，只要知道它是用来记录有哪些页是可用的就是，之后会单独有一篇文章来探索。


接下来就是B+树对应的两种页了，首先来看branchPage
{% highlight go %}
type branchPageElement struct {
	pos   uint32 //真正存储数据的地方和当前元素的位置差
	ksize uint32 //key的大小，因为是非叶子节点，没有value的大小
	pgid  pgid //page id
}
{% endhighlight %}
下面是获取key的方法：
{% highlight go %}
func (n *branchPageElement) key() []byte {
	buf := (*[maxAllocSize]byte)(unsafe.Pointer(n))
	return (*[maxAllocSize]byte)(unsafe.Pointer(&buf[n.pos]))[:n.ksize]
}
{% endhighlight %}
可以看到，它是把当前元素的地址拿出来，然后直接取pos位置的数据，大小为size.
注意这里的页和B+树里的节点一一对应，也就是说一页应该包含多个节点（因为B+树非叶子节点内也会存储多个key）

{% highlight go %}
// 获取非叶子节点中的某个元素，直接根据p.ptr进行指针操作，获取第index个
func (p *page) branchPageElement(index uint16) *branchPageElement {
	return &((*[0x7FFFFFF]branchPageElement)(unsafe.Pointer(&p.ptr)))[index]
}

// 获取非叶子节点中的所有元素
func (p *page) branchPageElements() []branchPageElement {
	if p.count == 0 {
		return nil
	}
	return ((*[0x7FFFFFF]branchPageElement)(unsafe.Pointer(&p.ptr)))[:]
}
{% endhighlight %}

这样一来，非叶子节点的布局大概也就清晰明了了， 如下图所示：
![png1]({{ site.baseurl }}/assets/img/boltdb/db-branch-page.png){: .center-image }
注意所有的key其实是存在一起的，可以根据前面的pos来快速查找。


叶子节点的布局也非常类似，只不过不同点在于多了一个value.
{% highlight go %}
type leafPageElement struct {
	flags uint32 //叶子节点需要有flag表示是不是嵌套的Bucket
	pos   uint32
	ksize uint32
	vsize uint32
}
{% endhighlight %}
获取叶子节点内的元素页都和上面类似，下面直接给出叶子节点的布局：
![png1]({{ site.baseurl }}/assets/img/boltdb/db-leaf-page.png){: .center-image }

这样一来，几种页面的布局我们都清楚了，下篇文章我们来学习一个node.go,之前已经说过若干遍，node是page在内存中的表示，node和page是一一对应的关系。
