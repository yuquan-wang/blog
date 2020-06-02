---
layout: post
title:  "boltdb源码阅读7-Freelist"
date:   2020-06-02 12:49:45 +0200
categories: 分布式 database
---
github 地址：[boltdb](https://github.com/boltdb/bolt)

本文主要探索boltdb是怎么处理freelist的相关逻辑的，整个类的代码并不多，我们可以直接来一行一行解读源码：
{% highlight go %}
type freelist struct {
	ids     []pgid          // 所有可用来被分配的页号
	pending map[txid][]pgid // 马上就要被事务txid释放的页号，为了保证隔离性，需要事务结束后才能真正移到上面的ids,表示真正可被分配
	cache   map[pgid]bool   // 缓存，用来快速查询某个page是不是可用或者即将可用
}

// newFreelist 返回一个空的freelist
func newFreelist() *freelist {
	return &freelist{
		pending: make(map[txid][]pgid),
		cache:   make(map[pgid]bool),
	}
}

// size 返回序列化之后page的大小，即页头大小+n个元素的大小
func (f *freelist) size() int {
	n := f.count()
	if n >= 0xFFFF {
		// The first element will be used to store the count. See freelist.write.
		n++
	}
	return pageHeaderSize + (int(unsafe.Sizeof(pgid(0))) * n)
}

// count 返回有多少可用的页（ids）和即将可用的页(pending)
func (f *freelist) count() int {
	return f.free_count() + f.pending_count()
}

// free_count 返回有多少可用的页
func (f *freelist) free_count() int {
	return len(f.ids)
}

// pending_count 返回有多少即将可用的页
func (f *freelist) pending_count() int {
	var count int
	for _, list := range f.pending {
		count += len(list)
	}
	return count
}

// copyall 将所有可用和即将可用的页排好序复制到dst中
// f.count returns the minimum length required for dst.
func (f *freelist) copyall(dst []pgid) {
	m := make(pgids, 0, f.pending_count())
	for _, list := range f.pending {
		m = append(m, list...)
	}
	sort.Sort(m)
	mergepgids(dst, f.ids, m)
}

// allocate 分配n个连续的页，返回第一个页的页号
func (f *freelist) allocate(n int) pgid {
	if len(f.ids) == 0 {
		return 0
	}

	var initial, previd pgid
	for i, id := range f.ids {
		if id <= 1 {
			panic(fmt.Sprintf("invalid page allocation: %d", id))
		}

		// Reset initial page if this is not contiguous.
		if previd == 0 || id-previd != 1 {
			initial = id
		}

		// 如果找到满足条件的：即连续的n个页
		if (id-initial)+1 == pgid(n) {
			if (i + 1) == n {
				f.ids = f.ids[i+1:]
			} else {
				copy(f.ids[i-n+1:], f.ids[i+1:])
				f.ids = f.ids[:len(f.ids)-n]//从可用页中删除
			}

			// 从缓存中删除
			for i := pgid(0); i < pgid(n); i++ {
				delete(f.cache, initial+i)
			}

			return initial //返回第一个页的页号
		}

		previd = id
	}
	return 0
}

// free 把某个事务中的page p标为可用的
func (f *freelist) free(txid txid, p *page) {
	if p.id <= 1 {
		panic(fmt.Sprintf("cannot free page 0 or 1: %d", p.id))
	}

	// Free page and all its overflow pages.
	var ids = f.pending[txid]
	for id := p.id; id <= p.id+pgid(p.overflow); id++ {
		// Verify that page is not already free.
		if f.cache[id] {
			panic(fmt.Sprintf("page %d already freed", id))
		}

		// 加到freelist的ids和cache中
		ids = append(ids, id)
		f.cache[id] = true
	}
	f.pending[txid] = ids
}

// release 把某个事务中即将释放的页变成彻底可用的，加入到f.ids
func (f *freelist) release(txid txid) {
	m := make(pgids, 0)
	for tid, ids := range f.pending {
		if tid <= txid {
			// Move transaction's pending pages to the available freelist.
			// Don't remove from the cache since the page is still free.
			m = append(m, ids...)
			delete(f.pending, tid)
		}
	}
	sort.Sort(m)
	f.ids = pgids(f.ids).merge(m)
}

// rollback 把给定事务即将释放的页给删掉，页就是说那些即将释放的页会像原来一直存在下去，之后不会变成可用的，数据页就不会被覆盖
func (f *freelist) rollback(txid txid) {
	// Remove page ids from cache.
	for _, id := range f.pending[txid] {
		delete(f.cache, id)
	}

	// Remove pages from pending list.
	delete(f.pending, txid)
}

// freed 返回某个页是不是可用可分配的
func (f *freelist) freed(pgid pgid) bool {
	return f.cache[pgid]
}

// read 反序列化，用page来填充freelist
func (f *freelist) read(p *page) {
	// If the page.count is at the max uint16 value (64k) then it's considered
	// an overflow and the size of the freelist is stored as the first element.
	idx, count := 0, int(p.count)
	if count == 0xFFFF {
		idx = 1
		count = int(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0])
	}

	// Copy the list of page ids from the freelist.
	if count == 0 {
		f.ids = nil
	} else {
		ids := ((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[idx:count]
		f.ids = make([]pgid, len(ids))
		copy(f.ids, ids)

		// Make sure they're sorted.
		sort.Sort(pgids(f.ids))
	}

	// Rebuild the page cache.
	f.reindex()
}

// write 序列化，从freelist写到page
func (f *freelist) write(p *page) error {
	// Combine the old free pgids and pgids waiting on an open transaction.

	// Update the header flag.
	p.flags |= freelistPageFlag

	// The page.count can only hold up to 64k elements so if we overflow that
	// number then we handle it by putting the size in the first element.
	lenids := f.count()
	if lenids == 0 {
		p.count = uint16(lenids)
	} else if lenids < 0xFFFF {
		p.count = uint16(lenids)
		f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[:])
	} else {
		p.count = 0xFFFF
		((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[0] = pgid(lenids)
		f.copyall(((*[maxAllocSize]pgid)(unsafe.Pointer(&p.ptr)))[1:])
	}

	return nil
}

// reload 从磁盘中重新加载freelist
func (f *freelist) reload(p *page) {
	f.read(p)

	// Build a cache of only pending pages.
	pcache := make(map[pgid]bool)
	for _, pendingIDs := range f.pending {
		for _, pendingID := range pendingIDs {
			pcache[pendingID] = true
		}
	}

	// Check each page in the freelist and build a new available freelist
	// with any pages not in the pending lists.
	var a []pgid
	for _, id := range f.ids {
		if !pcache[id] {
			a = append(a, id)
		}
	}
	f.ids = a

	// Once the available list is rebuilt then rebuild the free cache so that
	// it includes the available and pending free pages.
	f.reindex()
}

// reindex 重新构建缓存
func (f *freelist) reindex() {
	f.cache = make(map[pgid]bool, len(f.ids))
	for _, id := range f.ids {
		f.cache[id] = true
	}
	for _, pendingIDs := range f.pending {
		for _, pendingID := range pendingIDs {
			f.cache[pendingID] = true
		}
	}
}
{% endhighlight %}

整个freelist的代码还是比较清晰易懂的，那么我们重新来看一下上篇文章留下的问题， 在事务回滚的时候，都发生了些什么，下面是回滚代码：
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
可以看到，主要是读写事务的时候会有点特殊逻辑（废话，只读事务又不修改数据），主要是freelist调用rollback(txid)，然后用磁盘数据把freelist给重设了。

{% highlight go %}
func (f *freelist) rollback(txid txid) {
	// Remove page ids from cache.
	for _, id := range f.pending[txid] {
		delete(f.cache, id)
	}

	// Remove pages from pending list.
	delete(f.pending, txid)
}
{% endhighlight %}
rollback()的代码如上，主要就是把这个事务即将释放的页（f.pending）给从cache 和pending里删掉了，也就是说这个事务对应的页不会被删掉，上面的数据会一直留在磁盘上，因为当rollback()被调用时，可以肯定的是meta页肯定没有被覆盖，这样之后的读写还是会从老的meta页读取数据，然后读取原来的B+树的数据，本次事务的数据也就不会反应到磁盘上，保证了原子性。
