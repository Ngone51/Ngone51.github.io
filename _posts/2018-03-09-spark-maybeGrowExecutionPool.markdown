---
layout:     post
title:      "Spark UnifiedMemoryManager之maybeGrowExecutionPool()分析"
subtitle:   "Note：本文要求读者对UnifiedMemoryManager的原理有基本的了解"
date:       2018-03-09 17:35:00
author:     "wuyi"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spark
---

<p id = "build"></p>
---
本文试图分析UnifiedMemoryManager中的maybeGrowExecutionPool()方法。事实上，该方法是UnifiedMemoryManager的 acquireExecutionMemory()方法的内部方法。但个人觉得，理解清楚该方法有助于更好地理解UnifiedMemoryManager的工作原理。

先来看看该方法的全部源码：
```
def maybeGrowExecutionPool(extraMemoryNeeded: Long): Unit = {
      if (extraMemoryNeeded > 0) {
        // There is not enough free memory in the execution pool, so try to reclaim memory from
        // storage. We can reclaim any free memory from the storage pool. If the storage pool
        // has grown to become larger than `storageRegionSize`, we can evict blocks and reclaim
        // the memory that storage has borrowed from execution.
        val memoryReclaimableFromStorage = math.max(
          storagePool.memoryFree,
          storagePool.poolSize - storageRegionSize)
        if (memoryReclaimableFromStorage > 0) {
          // Only reclaim as much space as is necessary and available:
          val spaceToReclaim = storagePool.freeSpaceToShrinkPool(
            math.min(extraMemoryNeeded, memoryReclaimableFromStorage))
          storagePool.decrementPoolSize(spaceToReclaim)
          executionPool.incrementPoolSize(spaceToReclaim)
        }
      }
    }
```
首先，该方法何时被调用？
当execution pool的空闲内存不够用时，则该方法会被调用。从该方法的名字就可以得知，该方法试图增长execution pool的大小。

那么，如何增长execution pool呢？
通过阅读方法中的注释，我们可以看到有两种方式：
1. storage pool中有空闲内存，则借用storage pool中的空闲内存
2. storage pool的大小超过了```storageRegionSize ```，则驱逐存储在storage pool中的blocks，来回收storage pool从execution pool中借走的内存。

结合上述两种情况，本文针对该方法中的max()部分，展开更具体的讨论。**在不同的情况下，execution pool能使用的究竟是哪部分内存**？
```
val memoryReclaimableFromStorage = math.max(
    storagePool.memoryFree,
    storagePool.poolSize - storageRegionSize)
```

1.storage pool没有空闲内存可用。此时，有两种情况：
  **a)** storage poolSize <= storageRegionSize: execution pool不能驱逐             storage pool中的blocks来获取更多的内存。此时，execution pool只能等待内存主动释放，来获取更多内存。
        对应于max: memoryFree = 0, poolSize - storageRegionSize <= 0
        **b)** storage poolSize > storageRegionSize: execution pool可以驱逐storage pool中超过storageRegionSize的那部分内存中的blocks，来回收之前被storage pool借走的内存。
        对应于max: memoryFree = 0, poolSize - storageRegionSize > 0

2.storage pool有空闲内存可用。此时情况复杂些：
**a)** storage poolSize <= storageRegionSize: execution pool只能借用storage pool的空闲内存。
对应与max: memoryFree > 0, poolSize - storageRegionSize < 0
**b)** storage poolSize > storageRegionSize：此时又有两种情况：
**b.1)** storage pool usedMemory <= storageRegionSize:
 其中，usedMemory = poolSize - memoryFree。则，可得memoryFree >= poolSize - storageRegionSize。此时，execution pool的可用内存仅有storage pool的空闲内存。
**b.2)** storage pool usedMemory > storageRegionSize:
显然，此时memoryFree < poolSize - storageRegionSize。此时，execution pool最多可以利用的内存包括storage pool的空闲内存以及通过驱逐storage pool超过storageRegionSize的存储blocks而回收的内存。当然，spark会优先使用空闲内存。当空闲内存不够时，再通过驱逐storage pool的blocks回收内存。

**总结：可见，并不是当storage poolSize > storageRegionSize的时候，execution pool就会去驱逐storage pool中的blocks或者storage pool就没有空闲内存可用。而当storage poolSize < storageRegionSize的时候，
execution pool必然不能驱逐storage pool中的blocks。**
