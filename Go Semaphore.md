# Go Semaphore

[toc]

## 基本概念

Semaphore 是 Golang 的 mutex 实现的基础，Semaphore semacquire 保证只有 (*addr) 个 Goroutine 获取 Semaphore 成功，其他的 Goroutine 再调用 semacquire 就会直接被调度，等待着其他的 Goroutine 调用 Semrelease 函数复活。

有趣的是，当调用 require 的时候，(*addr) 会立刻减一，直到变成 0 之后，就不会再减为负数了。即使有再多的 G 在等待着信号量，`(*addr)` 也是 0。

当调用 release 之后，(*addr) 会自动自增，唤醒排在队列里面的 G，G 被唤醒之后，再对 `(*addr)` 递减，所以只要有 G 在 treap 中等待，那么 `(*addr)` 就一直保持着 0，直到所有的 G 都从 treap 中唤醒，这个时候 release 才会使得 `(*addr)` 成为正数，直到还原为初始值。

值得注意的是，当 G 被唤醒之后，并不是直接就返回，而是需要再次抢夺 `(*addr)`，因为很可能调用 release 和它刚刚被调度被唤醒中间有一段时间，其他的 G 在这段时间里抢夺了信号量。只有 release 的参数加上 handoff 的时候，release 会对 sodog.ticket 设置为 1，同时直接递减 (*addr) 的值，G 被调度唤醒之后，才能直接获得信号量。

```
//go:linkname sync_runtime_Semacquire sync.runtime_Semacquire
func sync_runtime_Semacquire(addr *uint32) {
	semacquire1(addr, false, semaBlockProfile, 0)
}

//go:linkname poll_runtime_Semacquire internal/poll.runtime_Semacquire
func poll_runtime_Semacquire(addr *uint32) {
	semacquire1(addr, false, semaBlockProfile, 0)
}

//go:linkname sync_runtime_Semrelease sync.runtime_Semrelease
func sync_runtime_Semrelease(addr *uint32, handoff bool, skipframes int) {
	semrelease1(addr, handoff, skipframes)
}

//go:linkname sync_runtime_SemacquireMutex sync.runtime_SemacquireMutex
func sync_runtime_SemacquireMutex(addr *uint32, lifo bool, skipframes int) {
	semacquire1(addr, lifo, semaBlockProfile|semaMutexProfile, skipframes)
}

//go:linkname poll_runtime_Semrelease internal/poll.runtime_Semrelease
func poll_runtime_Semrelease(addr *uint32) {
	semrelease(addr)
}

```

## 数据结构

### semaRoot

其实 Semaphore 实现原理很简单：

- 首先利用 atomic 来判断 `(*addr)` 当前的数值，如果当前 `(*addr)` 还大于 0，那么直接返回继续执行；

- 如果已经为 0 了，那么就找到 addr 映射的 semaRoot，先自增等待数 semaRoot.nwait。很多个 addr 可能映射到 semtable 数组中同一个 semaRoot。

- semaRoot 里面有个节点 sudog 类型的 treap 树，这个 treap 树的根就是 semaRoot.treap 变量。

- 遍历这个 treap 树，这个 treap 树是二叉搜索树，节点是按照 addr 地址大小来排序的。

- 在这个 treap 树中去找值为 addr 的节点，这个节点是所有被阻塞在 (addr) 上的 Goroutine 组合，把自己入队这个链表。
- gopark 自己当前的 G，等待着唤醒即可。

每次调用 semrelease1 的时候，过程相反。

我们先来看看数据结构：

```
type semaRoot struct {
    lock  mutex
    treap *sudog // root of balanced tree of unique waiters.
    nwait uint32 // Number of waiters. Read w/o the lock.
}

// Prime to not correlate with any user patterns.
const semTabSize = 251

var semtable [semTabSize]struct {
    root semaRoot
    pad  [sys.CacheLineSize - unsafe.Sizeof(semaRoot{})]byte
}

func semroot(addr *uint32) *semaRoot {
    return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}

```

### sudog 数据结构

sudog 是 G 结构的包装，用于连接多个等待某个条件的 Goroutine。prev/next 是双向链表的指针。

在 treap 中，prev 还代表着其左子树，next 代表着其右子树，parent 是其父节点。waitlink 是阻塞在同一个信号上的队列单链表指针，waittail 是队列尾节点，ticket 是 treap 的随机数，elem 是 treap 的 value（信号量地址），acquiretime 与 releasetime 代表着阻塞的时间。

```
type sudog struct {
	g *g

	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // data element (may point to stack)

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}

```

## semacquire 过程

过程如上一个小节所述：

- 首先利用 cansemacquire 来判断 addr 当前的值，如果还大于 0，那么减一，直接返回当前的 G；如果已经为 0 了，那么就得接着执行 semacquire 函数
- 初始化 sodug 结构体，ticket 是用于 treap 的随机数，保障 treap 大体平衡。acquiretime 是 G 进入调度的时间，对应的 releasetime 被出队的时间。
- root.nwait 自增
- 获取一个锁，这个 mutex 是一个比较底层的实现，是 runtime 专用的一种锁，这个锁实现的临界区通常非常小。如果拿不到当前线程可能会被 futexsleep 一小段时间。
- 拿到锁之后，再次判断当前的 addr 的值，如果已经变成了大于 0，直接解锁，然后返回。
- 在 treap 树中插入节点，入队
- goparkunlock 进行调度，调度开始之前的时候会自动解锁。这样临界区结束。
- 调度回来之后，判断是否真的可以返回，而不是误返回。误返回会重新入队并进入调度。
- s.ticket 代表特殊对待当前的 G，代表着调用 release 函数的时候使用了 handoff 参数，这时候就不需要再利用 cansemacquire 去抢夺信号量了。
- cansemacquire 是其他的 Goroutine 调用了 semrelease

```
func semroot(addr *uint32) *semaRoot {
	return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}

func cansemacquire(addr *uint32) bool {
	for {
		v := atomic.Load(addr)
		if v == 0 {
			return false
		}
		if atomic.Cas(addr, v, v-1) {
			return true
		}
	}
}

func semacquire(addr *uint32) {
	semacquire1(addr, false, 0, 0)
}

func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags, skipframes int) {
	gp := getg()
	if gp != gp.m.curg {
		throw("semacquire not on the G stack")
	}

	// Easy case.
	if cansemacquire(addr) {
		return
	}

	s := acquireSudog()
	root := semroot(addr)
	t0 := int64(0)
	s.releasetime = 0
	s.acquiretime = 0
	s.ticket = 0
	
	for {
		lock(&root.lock)
		// Add ourselves to nwait to disable "easy case" in semrelease.
		atomic.Xadd(&root.nwait, 1)
		// Check cansemacquire to avoid missed wakeup.
		if cansemacquire(addr) {
			atomic.Xadd(&root.nwait, -1)
			unlock(&root.lock)
			break
		}
		// Any semrelease after the cansemacquire knows we're waiting
		// (we set nwait above), so go to sleep.
		root.queue(addr, s, lifo)
		goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4+skipframes) // 开始调度
		if s.ticket != 0 || cansemacquire(addr) { // 调度返回
			break
		}
	}

	releaseSudog(s)
}

```

## semrelease 过程

- 对 addr 自增
- 首先判断 root.nwait 是否是 0，如果是的话，就是说明当前整个 root 都没有 G 等待
- 获取锁，然后再次检查
- treap 找到 addr 对应的节点，然后进行出队
- 如果设置了 handoff 参数，那么直接递减 (*addr) 信号量，不需要取出的 G 再进行抢夺。
- readyWithTime 赋值 s.releasetime，然后唤醒对应的 G

```
func semrelease(addr *uint32) {
	semrelease1(addr, false, 0)
}

func semrelease1(addr *uint32, handoff bool, skipframes int) {
	root := semroot(addr)
	atomic.Xadd(addr, 1)

	if atomic.Load(&root.nwait) == 0 {
		return
	}

	// Harder case: search for a waiter and wake it.
	lock(&root.lock)
	if atomic.Load(&root.nwait) == 0 {
		// The count is already consumed by another goroutine,
		// so no need to wake up another goroutine.
		unlock(&root.lock)
		return
	}
	s, t0 := root.dequeue(addr)
	if s != nil {
		atomic.Xadd(&root.nwait, -1)
	}
	unlock(&root.lock)
	if s != nil { // May be slow, so unlock first
		if s.ticket != 0 {
			throw("corrupted semaphore ticket")
		}
		if handoff && cansemacquire(addr) {
			s.ticket = 1
		}
		readyWithTime(s, 5+skipframes)
	}
}

func readyWithTime(s *sudog, traceskip int) {
	if s.releasetime != 0 {
		s.releasetime = cputicks()
	}
	goready(s.g, traceskip)
}
```

## treap queue 入队过程

sudog 按照地址 hash 到 251 个 bucket 中的其中一个，每一个 bucket 都是一棵 treap。而相同 addr 上的 sudog 会形成一个链表。

为啥同一个地址的 sudog 不需要展开放在 treap 中呢？显然，sudog 唤醒的时候，block 在同一个 addr 上的 goroutine，说明都是加的同一把锁，这些 goroutine 被唤醒肯定是一起被唤醒的，相同地址的 g 并不需要查找才能找到，只要决定是先进队列的被唤醒(fifo)还是后进队列的被唤醒(lifo)就可以了。

### treap 树插入

要理解 treap 树相对于普通二叉搜索树的优点，我们需要先看看普通二叉搜索树的插入过程：

```
int BSTreeNodeInsertR(BSTreeNode **tree,DataType x) //搜索树的插入
{
    if(*tree == NULL)
    {
        *tree = BuyTreeNode(x);
        return 0;
    }

    if ((*tree)->_data > x)
        return BSTreeNodeInsertR(&(*tree)->_left,x);
    else if ((*tree)->_data < x)
        return BSTreeNodeInsertR(&(*tree)->_right,x);
    else
        return -1;
}	

```

我们可以看到，这段代码从根 root 开始递归查找，直到叶子节点，然后把自己挂到叶子节点的左孩子或者右孩子。这种插入方法很容易造成二叉树的不平衡，有的地方深度为 10，有的地方深度为 2。

AVL 树通过左旋或者右旋，可以实现不破坏二叉树的特征基础上减小局部的深度。但是 AVL 树要求过于严格，它不允许任何左右子树的深度差值超过 1，这样虽然查询特别快速，但是插入操作会进行很多左旋右旋的动作，效率比较低，

红黑树采用节点黑、红的特点实现近似的 AVL 树，效率比较高，但是实现上过于繁琐。

跳表是另一种实现快速查找的数据结构，采用的是随机建立索引的方式，实现快速查找链表。

Golang 中大量使用的是 treap 树，这个树在二叉搜索树的基础上，在每个节点上加入随机 ticket。它的原理很简单，对于随机插入的二叉堆，大概率下二叉堆是近似平衡的。

我们可以利用这个规则，先找到需要插入的叶子节点，然后赋予一个随机数，利用这个随机数调整最小堆。调整最小堆的过程，并不是普通的直接替换父子节点即可，而是进行左旋与右旋操作，保障二叉树的性质：

```
func (root *semaRoot) queue(addr *uint32, s *sudog, lifo bool) {
	s.g = getg()
	s.elem = unsafe.Pointer(addr)
	s.next = nil
	s.prev = nil

	var last *sudog
	pt := &root.treap
	for t := *pt; t != nil; t = *pt {
		last = t
		if uintptr(unsafe.Pointer(addr)) < uintptr(t.elem) {
			pt = &t.prev // 左子树
		} else {
			pt = &t.next // 右子树
		}
	}

	s.ticket = fastrand() | 1    // 随机数的生成
	s.parent = last
	*pt = s

	// Rotate up into tree according to ticket (priority).
	for s.parent != nil && s.parent.ticket > s.ticket { // 最小二叉堆的调整
		if s.parent.prev == s {
			root.rotateRight(s.parent)  // 右旋
		} else {
			root.rotateLeft(s.parent)  // 左旋
		}
	}
}

```

左旋和右旋的代码也比较简单：

```
func (root *semaRoot) rotateLeft(x *sudog) {
	// p -> (x a (y b c))
	p := x.parent
	a, y := x.prev, x.next
	b, c := y.prev, y.next

	y.prev = x
	x.parent = y
	y.next = c      // 个人认为是不必要的步骤
	if c != nil {
		c.parent = y
	}
	
	x.prev = a
	if a != nil {
		a.parent = x
	}
	x.next = b       // 个人认为是不必要的步骤
	if b != nil {
		b.parent = x
	}

	y.parent = p
	if p == nil {
		root.treap = y
	} else if p.prev == x {
		p.prev = y
	} else {
		if p.next != x {
			throw("semaRoot rotateLeft")
		}
		p.next = y
	}
}

func (root *semaRoot) rotateRight(y *sudog) {
	// p -> (y (x a b) c)
	p := y.parent
	x, c := y.prev, y.next
	a, b := x.prev, x.next

	x.prev = a
	if a != nil {
		a.parent = x
	}
	x.next = y
	
	y.parent = x
	y.prev = b
	if b != nil {
		b.parent = y
	}
	y.next = c
	if c != nil {
		c.parent = y
	}

	x.parent = p
	if p == nil {
		root.treap = x
	} else if p.prev == y {
		p.prev = x
	} else {
		if p.next != y {
			throw("semaRoot rotateRight")
		}
		p.next = x
	}
}
```

### sodog 链表的插入

当 treap 树中已经存在了节点，这个时候就需要更新等待的队列。

插入队尾比较简单，只需要更新前一个元素的 waitlink 即可。

但是插入队首比较麻烦，因为队首元素要替代之前的队首成为 treap 节点，之前的元素

```
func (root *semaRoot) queue(addr *uint32, s *sudog, lifo bool) {
    ...
    if t.elem == unsafe.Pointer(addr) {
			// Already have addr in list.
			if lifo {                       
				// 插入链表队首
				
				// 继承之前队首的属性
				*pt = s
				s.ticket = t.ticket
				s.acquiretime = t.acquiretime
				s.parent = t.parent
				s.prev = t.prev
				s.next = t.next
				
				// 更新 treap 节点的左右节点
				if s.prev != nil {
					s.prev.parent = s
				}
				if s.next != nil {
					s.next.parent = s
				}
				
				// Add t first in s's wait list.
				s.waitlink = t
				s.waittail = t.waittail
				if s.waittail == nil {
					s.waittail = t
				}
				
				// 重置之前队首节点的属性
				t.parent = nil
				t.prev = nil
				t.next = nil
				t.waittail = nil
			} else {
				// 插入链表队尾
				if t.waittail == nil {
					t.waitlink = s
				} else {
					t.waittail.waitlink = s
				}
				t.waittail = s
				s.waitlink = nil
			}
			return
		}
		...
}
```

## treap 出队过程


```
func (root *semaRoot) dequeue(addr *uint32) (found *sudog, now int64) {
	ps := &root.treap
	s := *ps
	for ; s != nil; s = *ps {
		if s.elem == unsafe.Pointer(addr) {
			goto Found
		}
		if uintptr(unsafe.Pointer(addr)) < uintptr(s.elem) {
			ps = &s.prev
		} else {
			ps = &s.next
		}
	}
	return nil, 0

Found:
	now = int64(0)
	if s.acquiretime != 0 {
		now = cputicks()
	}
	if t := s.waitlink; t != nil {
		// 需要用 t 来替代 s 在 treap 的节点
		*ps = t
		t.ticket = s.ticket
		t.parent = s.parent
		t.prev = s.prev
		
		if t.prev != nil {
			t.prev.parent = t
		}
		t.next = s.next
		if t.next != nil {
			t.next.parent = t
		}
		
		if t.waitlink != nil {
			t.waittail = s.waittail
		} else {
			t.waittail = nil
		}
		
		t.acquiretime = now
		s.waitlink = nil
		s.waittail = nil
	} else {
		// 在 treap 中删除 addr 节点
		
		// 先调整左右子树
		// 注意这里是 for 循环，只要 s 还有左右子树就不断的调整
		// 左右子树，谁的 ticket 小，谁做父节点
		for s.next != nil || s.prev != nil {
			if s.next == nil || s.prev != nil && s.prev.ticket < s.next.ticket {
				root.rotateRight(s)
			} else {
				root.rotateLeft(s)
			}
		}
		
		// 删除 s 节点，s 现在是叶子节点
		if s.parent != nil {
			if s.parent.prev == s {
				s.parent.prev = nil
			} else {
				s.parent.next = nil
			}
		} else {
			root.treap = nil
		}
	}
	s.parent = nil
	s.elem = nil
	s.next = nil
	s.prev = nil
	s.ticket = 0
	return s, now
}

```·

