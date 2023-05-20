---
layout:     post
tags:
    - golang
    - memory
    - pprof
---
# golang memory profile (repost)

源于<https://blog.detectify.com/2019/09/05/how-we-tracked-down-a-memory-leak-in-one-of-our-go-microservices/,> 本文对此文做些概述、归纳。  
当我们写的golang程序碰到疑似内存泄漏的时候，最常规的做法是使用pprof工具进行排查。  
用pprof进行debug之前，先对go程序内存泄漏的场景进行总结：  

- Creating substrings and subslices
- Wrong use of the defer statement
- Unclosed HTTP response bodies (or unclosed resources in general)
- Orphaned hanging go routines
- Global variables  

更多的案例可以阅读 [go101](https://go101.org/article/memory-leaking.html), [vividcortex](https://www.vividcortex.com/blog/2014/01/15/two-go-memory-leaks/) and [hackernoon](https://hackernoon.com/avoiding-memory-leak-in-golang-api-1843ef45fca8).

中间介绍了多个pprof操作，排查内存泄漏。  

关于issues on changes in the Go memory management system:  

- [runtime: mechanism for monitoring heap size](https://github.com/golang/go/issues/16843)
- [runtime: scavenging doesn’t reduce reported RSS on darwin, may lead to OOMs on iOS](https://github.com/golang/go/issues/29844)
- [runtime: provide way to disable MADV_FREE](https://github.com/golang/go/issues/28466)
- [runtime: Go routine and writer memory not being released](https://github.com/golang/go/issues/32124)  

golang的两个重要的runtime signal: MADV_DONTNEED and MADV_FREE:

- MADV_DONTNEED: runtime sends a MADV_DONTNEED signal on unused memory and the operating system immediately reclaims the unused memory pages  
- MADV_FREE:  tells the operating system that it can reclaim some unused memory pages if it needs to, meaning it doesn’t always do that unless the system is under memory pressure from different processes  

golang12版本开始引入MADV_FREE, 而Linux 4.5及之后的版本中，默认使用MADV_FREE方式，这意味程序内存不会立刻回收，即RSS值不会立刻下降，只有当OS内存紧缺时才会回收Go程序的内存返回给OS。因此如果需要使程序内存占用下降很快的话，可设置环境变量GODEBUG=madvdontneed=1 (<https://bbs.huaweicloud.com/blogs/150520>)，主动FreeOSMemory将很快释放内存给OS, 否则默认5min未使用则OS回收。
