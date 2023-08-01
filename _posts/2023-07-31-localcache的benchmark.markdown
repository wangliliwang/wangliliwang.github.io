---
layout: post
title:  "cache的benchmark方法"
date:   2023-07-31 17:22:33 +0800
categories: "Computer-Science"
tags: ["Computer-Science"]
toc: true
language: chinese
sidebar: true
author: Wang Li
---

当需要衡量cache的性能时，我们需要数据集和指标，以及benchmark工具。本文会介绍这些内容。

由于笔者自己最近在开发一款localcache，所以文中会使用此cache和另一款知名localcache跑一些示例数据，两个localcache为：
1. [ristretto](https://github.com/dgraph-io/ristretto)
2. [fastlocalcache-v1](https://github.com/wangliliwang/fastlocalcache/tree/v1)

## 数据集

`ARC`前缀的数据集，是在真实场景（真实服务器的磁盘访问）搜集（trace）到的，所以这些数据集也被称为trace。

| 数据集      | 适用场景     | 说明                     | 数据示例           |
|----------|----------|------------------------|----------------|
| ARC-S3   | Search   | 运行搜索引擎服务器的磁盘访问数据       | 333024536 1 0 0 |
| ARC-DS1  | Database | 运行商业数据库服务器的磁盘访问数据      | 94151700 4 0 0 |
| ARC-OLTP |CODASYL| 运行CODASYL数据库服务器的磁盘访问数据 | 10 1 0 0       |
| LRIS-GLI | Looping  | 循环数据（周期为1011）          | 100  |

这些数据可以在 `https://github.com/dgraph-io/benchmarks/tree/master/cachebench/ristretto/trace` 下载到。

## 测试指标

考察cache的使用场景：通过key尝试在cache中获取数据，如果miss，则回退到代价更大的db中获取数据。

所以要想提升系统性能，需要从两方面入手：
1. 命中率。命中率越高，从cache中获取数据的次数比例就越高。
2. 以ops衡量的吞吐量。也就是单位时间内处理的请求次数。

说明：运行benchmark的机器参数如下：
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

```golang
hitCount := 0
missCount := 0
for key := range keys {
	if _, ok := cache.Get(key); ok {
	    hitCount++
    } else {
        missCount++	
    }
	cache.Set(key, "*")
}
hitRatio := float64(hitCount) / float64(hitCount + missCount)
```

不同的缓存失效方式、容量限制策略，会影响命中率。相同的cache，会在不同数据集上有迥异的表现。同时，一个数据集上的命中率会有一个确定的上限。

如，LRIS-GLI是一个循环数据集，数据以1011为周期进行循环，如果使用capacity小于1011的FIFO淘汰策略的cache进行缓存，命中率会是0。

以下是两个库在LRIS-GLI数据集上的命中率：

<img class="middle-image" src="/assets/image/20230731-localcache/hit-ratios.png" />

分析：fastlocalcache这个库没有淘汰机制，所以保持最高命中率。但命中率只体现一部分性能要素，我们在实际使用时，需要根据自己的需求来决定使用哪个库。

### 吞吐量 throughput

吞吐量和访问模式挂钩。常见的benckmark访问模式有：

| 访问模式     | 说明                     | 伪代码                                                                                                                                               |
|----------|------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| get-same | set一次，get多次            | cache.Set("\*", []byte("\*"))<br/>for i := 0; i < b.N; i++ {<br/>  cache.Get("*")<br/>}                                                           |
|get-zipf| 使用Zipfian分布规律进行get     | for _, key := range keys {<br/>cache.Set(key, []byte("*"))<br/>}<br/>for i := uint64(0); i < b.N; i++ {<br/>cache.Get(keys[i&(capacity-1)])<br/>} |
|set-get| set一次，get一次            | for i := 0; i < b.N; i++ {<br/>if i&1 == 0 {<br/>cache.Set("\*", vals)<br/>} else {<br/>cache.Get("\*")<br/>}<br/>}                               |
|set-same| 一直set相同的key            | for i := 0; i < b.N; i++ {<br/>cache.Set("\*", "\*")<br/>}                                                                                        |
|set-zipf| 使用Zipfian分布规律进行set     | for i := uint64(0); i < b.N; i++ {<br/>cache.Set(keys[i&(capacity-1)], vals)<br/>}                                                                                        |
|set-get-zipf| 使用Zipfian分布规律进行set-get |for i := uint64(0); i < b.N; i++ {<br/>if _, ok := cache.Get(keys[i&(capacity-1)]); !ok {<br/>cache.Set(keys[i&(capacity-1)], vals)<br/>}<br/>}|

注：Zipfian分布常用来模拟接近于实际场景的访问方式，以这种分布产生的随机数，会有更多的重复值。下图是Zipfian分布的概率质量函数：
<img class="middle-image" src="/assets/image/20230731-localcache/Zipf_distribution_PMF.png" />

如果使用golang标准库`testing`中的benchmark方法，会同时计算出以下数据：
1. mop/s: million operations per second
2. ns/op: nanoseconds per operation
3. ac: allocations per operation
4. byt: bytes allocated per operation

对于吞吐量这个指标，我们只需要关注`mop/s`这条数据。

在示例库上运行的benchmark数据如下：

<img class="middle-image" src="/assets/image/20230731-localcache/throughput.png" />

## 工具

本文用到的工具，可以在 `https://github.com/dgraph-io/benchmarks/tree/master/cachebench/ristretto` 找到。

## 参考

1. [ARC: a self-tuning, low overhead replacement cache](https://scinapse.io/papers/1860107648)
2. [ristretto bench](https://github.com/dgraph-io/benchmarks/tree/master/cachebench/ristretto)
3. [Introducing Ristretto: A High-Performance Go Cache](https://dgraph.io/blog/post/introducing-ristretto-high-perf-go-cache/)
