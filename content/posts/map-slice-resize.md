---
title: 切片和字段的扩容
date: 2023-03-02T17:09:51+08:00
lastmod: 2023-03-02T17:09:51+08:00
cover: https://img.sysummery.top/golang-map.png
avatar: https://img.sysummery.top/coffee.jpg
authorlink: https:www.skylersu.com
categories:
  - 后端
tags:
  - go
---
Map和Slice在使用的时候如果容量不够了会自动扩容，这点对于我们来说还是很方便的。今天想研究一下他们的扩容机制

<!--more-->


### Map扩容
代码位置：`GoSDK/src/runtime/map.go`
触发扩容的条件有2个

1. 超过了负载因子，这个是是6.5
2. 溢出桶的数量 >= 桶的数量

有关超出负载因子的判断在函数`overLoadFactor`中
```go
const (
	bucketCntBits = 3 //bucketCnt是根据bucketCntBits计算的，是2的倍数初始值是8
	bucketCnt     = 1 << bucketCntBits // 相当于2^3 = 8

	// 这两个值决定了为啥负载因子是6.5而不是其他的数
	loadFactorNum = 13
	loadFactorDen = 2
)
// bucketShift 返回 1<<b，为了优化代码生成。
func bucketShift(b uint8) uintptr {
	// 通过掩码处理移位数量，可以省略溢出检查。
	return uintptr(1) << (b & (goarch.PtrSize*8 - 1))
}

// overLoadFactor 报告将 count 个项放置在 1<<B 个桶中是否超过负载因子。
func overLoadFactor(count int, B uint8) bool {
	// 如果 count 大于每个桶能容纳的元素数量（bucketCnt），并且
	// count 大于负载因子允许的最大元素数量（loadFactorNum*(bucketShift(B)/loadFactorDen)），
	// 则返回 true，表示超过负载因子。
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```
解释一下`uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)`这行代码，假设：
```go
x = loadFactorNum*(bucketShift(B)/loadFactorDen)
```
做一下数学变换,把`loadFactorNum`移到左面
```go
x/loadFactorNum = bucketShift(B)/loadFactorDen
```
再做一次变换
```go
x * loadFactorDen = loadFactorNum * bucketShift(B)
```
再把loadFactorDen移到右边
```go
x = (loadFactorNum * bucketShift(B)) / loadFactorDen
```
最终得到
```go
x = bucketShift(B) * 6.5
```
因此当桶的平均键值对个数超过6.5的时候就是**超过了负载因子**

有关溢出桶数量的判断在函数`tooManyOverflowBuckets`中

```go
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// 如果阈值过低，我们会做额外的工作。
	// 如果阈值过高，那么增长和收缩的 map 可以保留大量未使用的内存。
	// “太多”意味着（大约）与常规桶一样多的溢出桶。
	// 有关更多详细信息，请参阅 incrnoverflow。
	if B > 15 {
		B = 15
	}
	// 编译器在这里看不到 B < 16；掩蔽 B 以生成更短的移位代码。
	return noverflow >= uint16(1)<<(B&15)
}
```
由此可见map最多有`2的15次方`个桶的数量，当满足这2个其中的1个条件后就会触发扩容是同步的渐进式的，哈希表的重新哈希和迁移。具体的执行方法是`hashGrow`、`growWork`和`evacuate`


### Slice的扩容
相对map来说，slice的扩容只是申请一块更大的内存然后把数据copy过去，不会有rehash的过程。我们通过一个demo来观察一下扩容策略
```go
func main() {
    for i := 0; i < 2000; i += 100 {
        fmt.Println(i, cap(append(make([]bool, i), true)))
    }
}
```
得到的结果为：
```
0 8
100 208
200 416
300 640
400 896
500 1024
600 1280
700 1408
800 1792
900 2048
1000 2048
1100 1408
1200 1536
1300 1792
1400 1792
1500 2048
1600 2048
1700 2304
1800 2304
1900 2688
```
把这些点绘制成图会比较直观
![](https://img.sysummery.top/slice-resize3.jpg)
可以看出大致的正常趋势：

| 初始长度  |增长比例  |
| --- | --- |
| 256 |2.0  |
| 512 | 1.63 |
| 1024 | 1.44 |
| 4096 | 1.30 |


切片越大增加的越缓慢
