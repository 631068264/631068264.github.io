---
layout:     post
rewards: false
title:    performance
categories:
- ml
tags:
- opencv
---

# 时间

```python
import cv2 as cv

# 时钟周期数
e1 = cv.getTickCount()
# your code execution
e2 = cv.getTickCount()

# 时间 = 时钟周期数/时钟周期的频率
time = (e2 - e1)/ cv.getTickFrequency()

print(time)
```
# 优化

Many of the OpenCV functions are optimized using SSE2, AVX etc. OpenCV runs the optimized code if it is enabled.

`cv.useOptimized()` to check if it is enabled/disabled and `cv.setUseOptimized()` to enable/disable

通常，OpenCV函数比Numpy函数更快。因此，对于相同的操作，OpenCV功能是首选。但是，可能有例外，尤其是当Numpy使用视图而不是副本时。

- 尽量避免在Python中使用循环，尤其是双循环/三循环等。它们本质上很慢。
- 将算法/代码矢量化到最大可能范围，因为Numpy和OpenCV针对向量运算进行了优化。
- 利用缓存一致性。
- 除非需要，否则永远不要复制数组。尝试使用视图。阵列复制是一项昂贵的操作。
- 即使在完成所有这些操作之后，如果您的代码仍然很慢，或者使用大型循环是不可避免的，请使用其他库（如Cython）来加快速度。


## numpy 视图 副本

视图

数据的一个别称或引用，对视图进行修改，它会影响到原始数据，物理内存在同一位置。
- numpy 的切片操作返回原数据的视图
- 调用 ndarray 的 view() 函数产生一个视图

副本

一个数据的完整的拷贝，如果我们对副本进行修改，它不会影响到原始数据，物理内存不在同一位置。
- Python deepCopy()函数
- 调用 ndarray 的 copy() 函数产生一个副本。