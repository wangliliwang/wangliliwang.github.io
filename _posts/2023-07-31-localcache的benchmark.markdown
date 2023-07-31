---
layout: post
title:  "localcache的benchmark"
date:   2023-07-31 17:22:33 +0800
categories: "Computer Science"
tags: ["Computer Science"]
toc: false
language: chinese
sidebar: true
author: Wang Li
---

当需要衡量localcache的性能时，我们需要数据集和指标，以及benchmark工具。本文会介绍一些测试集、性能指标，以及一些工具。

## 数据集

| 数据集      | 适用场景     | 说明                                  | 数据示例            |
|----------|----------|-------------------------------------|-----------------|
| ARC-S3   | Search   | 由大型商业搜索引擎响应各种网络搜索请求而发起的磁盘读取访问数据     | 333024536 1 0 0 |
| ARC-DS1  | Database | 运行在商业站点上的数据库服务器，在商业数据库之上运行 ERP 应用程序 | 94151700 4 0 0  |
| LRIS-GLI | Looping  | 循环                                  | 100 (1011循环一次)  |
| ARC-OLTP |CODASYL|一小时内对 CODASYL 数据库的引用。| 10 1 0 0        |


## 测试指标

先来考察cache的使用场景：我们通过key尝试在cache中获取数据，如果在cache中miss，则回退到代价更大的db中获取数据。

所以要想提升系统性能，需要从两方面入手：
1. 命中率。命中率高，从cache中获取数据的次数比例就越高。
2. 以ops衡量的吞吐量。也就是单位时间内处理的请求次数。

以下作为示例的测试指标，均是在以下两个项目上对比的结果：（todo）
1. [ristretto](https://github.com/dgraph-io/ristretto)
2. [fastlocalcache-v1](https://github.com/wangliliwang/fastlocalcache/tree/v1)

运行benchmark的机器参数如下：
```text
Model Name: MacBook Air
Model Identifier: MacBookAir10,1
Chip: Apple M1
Total Number of Cores: 8 (4 performance and 4 efficiency)
Memory: 16 GB
Type: LPDDR4
```

### 命中率 hit ratio

计算方式用伪代码表示如下：

其中`keyValues`是某个数据集内的全部数据。

```golang
hitCount := 0
missCount := 0
for key, value := range keyValues {
	if _, ok := cache.Get(key); ok {
	    hitCount++	
    } else {
        missCount++	
    }
	cache.Set(key, value)
}
hitRatio := float64(hitCount) / float64(hitCount + missCount)
```

不同的缓存失效方式、容量限制策略，会在很大程度上影响命中率。相同的cache，也会在不同数据集上有迥异的表现。

如，LRIS-GLI是一个循环数据集，数据以1011为周期进行循环，如果使用cap<1011的FIFO淘汰策略的cache进行缓存，那么命中率会是0.

如果没有容量限制、淘汰策略，那么在一个数据集上的命中率是一个确定的值，这个值也是命中率的上限。

(todo 绘图)

分析：fastlocalcache这个库没有淘汰机制，所以总是最高命中率。所以我们在实际使用时，需要根据自己的需求来决定到底使用哪个库。

### 吞吐量 throughput

吞吐量和访问模式挂钩。常见的benckmark的访问模式有：

| 访问模式     | 说明                           | 伪代码                                                                                                                                               |
|----------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| get-same | set一次，get多次                  | cache.Set("\*", []byte("\*"))<br/>for i := 0; i < b.N; i++ {<br/>  cache.Get("*")<br/>}                                                           |
|get-zipf| 使用Zipfian分布，模拟更接近于实际工作流的访问方式 | for _, key := range keys {<br/>cache.Set(key, []byte("*"))<br/>}<br/>for i := uint64(0); i < b.N; i++ {<br/>cache.Get(keys[i&(capacity-1)])<br/>} |
|set-get| set一次，get一次                  | for i := 0; i < b.N; i++ {<br/>if i&1 == 0 {<br/>cache.Set("\*", vals)<br/>} else {<br/>cache.Get("\*")<br/>}<br/>}                               |
|set-same| 一直set相同的key                  | for i := 0; i < b.N; i++ {<br/>cache.Set("\*", "\*")<br/>}                                                                                        |
|set-zipf| 一直set周期性重复的key               | for i := uint64(0); pb.Next(); i++ {<br/>cache.Set(keys[i&(capacity-1)], vals)<br/>}                                                                                        |
|set-get-zipf| -                            |for i := uint64(0); i < b.N; i++ {<br/>if _, ok := cache.Get(keys[i&(capacity-1)]); !ok {<br/>cache.Set(keys[i&(capacity-1)], vals)<br/>}<br/>}|

如果使用golang标准库`testing`中的benchmark方法，或同时计算出以下数据：
1. mop/s: million operations per second
2. ns/op: nanoseconds per operation
3. ac: allocations per operation
4. byt: bytes allocated per operation

对于吞吐量这个指标，我们只需要关注`mop/s`这条数据。

对于`get-zipf`, `set-same`, `set-get-zipf`这三种访问模式，在示例库上运行的数据如下：
（todo）

分析：性能差异，差距在哪里。

## 参考

1. https://scinapse.io/papers/1860107648
2. 
