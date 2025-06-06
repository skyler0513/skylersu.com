---
title: Go语言中的垃圾回收
date: 2020-01-01 17:37:50
lastmod: 2020-01-01 17:37:50
cover: https://img.sysummery.top/gogc.jpg
avatar: https://img.sysummery.top/coffee.jpg
authorlink: https:www.skylersu.com
categories:
  - 后端
tags:
  - go
---
垃圾回收（英语：Garbage Collection，缩写为GC），在计算机科学中是一种自动的存储器管理机制。当一个计算机上的动态存储器不再需要时，就应该予以释放，以让出存储器，这种存储器资源管理，称为垃圾回收。垃圾回收器可以让程序员减轻许多负担，也减少程序员犯错的机会。
<!--more-->
## 垃圾回收算法
### 引用计数法
每个单元维护一个域，保存其他单元对其的引用。当引用为0时就可以回收了。PHP就是使用的这种机制。

引用计数是渐进式的，内存管理与用户程序交织在一起，不会出现STW，但是这种方式解决不了互相引用的问题。比如
```
a.b = b;
b.a = a;
```
a和b二者互相引用，并且没有其他的引用了，引用计数法并不会把他们当做垃圾回收。

### 标记清扫法
是一种自动内存管理基于追踪的垃圾回收算法。垃圾回收的时候首先会STW。然后从根节点开始追踪每一个可达的对象，标记完之后就去回收没有被标记的几点然后进行回收。

那么标记清扫法能不能解决循环引用的问题呢？是可以的
![](https://img.sysummery.top/biaojishanchu.jpg)

虽然object4与object5互相引用，但是从根节点起是没有到达他们的路径的，因此他们两个不会被标记可达，在清扫阶段会被回收。

标记清扫法最大的缺点就是GC的时候会STW。

### 分代收集
分代垃圾回收算法将对象按生命周期长短存放到堆上的两个（或者更多）区域，这些区域就是分代（generation）。对于新生代的区域的垃圾回收频率要明显高于老年代区域。

分配对象的时候从新生代里面分配，如果后面发现对象的生命周期较长，则将其移到老年代，这个过程叫做 promote。随着不断 promote，最后新生代的大小在整个堆的占用比例不会特别大。收集的时候集中主要精力在新生代就会相对来说效率更高，STW 时间也会更短。

java的GC就是分代收集。

### 三色标记法
![](https://img.sysummery.top/sansebiaojifa.gif)

三色标记算法是对标记阶段的改进，原理如下：

1. 起初所有对象都是白色。
2. 从根出发扫描所有可达对象，标记为灰色，放入待处理队列。
3. 从队列取出灰色对象，将其引用对象标记为灰色放入队列，自身标记为黑色。
4. 重复 3，直到灰色对象队列为空。此时白色对象即为垃圾，进行回收。

步骤1、2、3的时候需要STW, 不然会出现问题。假如不STW，如下图，灰色的B节点引用了白色的C节点。当C节点还没有变灰的时候，B和C之间的引用断了并且已经变黑的A节点又引用了C。因此C不会变成灰色的，还是白色。等到回收的时候会把C回收了，但是C明明被黑色的A引用着，就出问题。因此三色球算法在标记的过程中必须要STW。
![](https://img.sysummery.top/3sbjfqx.jpg)


## go语言的垃圾回收算法
go原因的垃圾回收算法是基于三色标记法的，并在此基础上加入了写屏障缩短了整个过程的STW时间。

![](https://img.sysummery.top/gc.png)

具体的过程是

1. 首先从 root 开始遍历，root 包括全局指针和 goroutine 栈上的指针。
2. 从 root 开始遍历，标记为灰色。遍历灰色队列。
3. re-scan 全局指针和栈。因为 mark 和用户程序是并行的，所以在过程 1 的时候可能会有新的对象分配，这个时候就需要通过写屏障（write barrier）记录下来。re-scan 再完成检查一下。

SWT的阶段

1. 第一个是 GC 将要开始的时候，这个时候主要是一些准备工作，比如 enable write barrier。
2. 第二个过程就是上面提到的 re-scan 过程。如果这个时候没有 stw，那么 mark 将无休止。

什么时候触发gc

1. 阈值：默认内存扩大一倍，启动gc
2. 定期：默认2min触发一次
3. 手动：runtime.gc()

