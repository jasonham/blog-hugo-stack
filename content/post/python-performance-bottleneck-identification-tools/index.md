---
title: Python性能瓶颈定位
description: 介绍 python 性能瓶颈定位中常用的三个核心工具：cProfile、line_profiler、memory_profiler。
date: 2025-09-11T23:21:41+08:00
lastmod: 2025-09-11T23:21:41+08:00
slug: python-performance-bottleneck-identification-tools

tags:
  - Python
  - 优化
categories:
  - Python
---

在开发中，尤其是高并发、大数据量、C 端产品的场景下，程序慢 ≠ 整体慢，往往是某几个函数、某几行代码、某次内存分配拖慢了整个系统。

性能瓶颈定位的目的：

- 找出程序中最耗时的函数或代码行（CPU Bound）
- 找出内存占用最高的地方（Memory Bound）
- 避免“拍脑袋优化”，做到“精准打击”
- 为后续优化提供数据支撑（优化前 vs 优化后对比）

性能瓶颈定位中常用的三个核心工具：cProfile、line_profiler、memory_profiler。它们分别用于函数级性能分析、行级性能分析、内存使用分析.

## cProfile

cProfile 是 Python 官方提供的一个性能分析工具，用于对 Python 代码进行性能分析。  
它会生成一个文件，里面包含函数调用的统计信息，包括函数调用的次数、执行时间、调用关系等信息。

> 注意：
>
> - cProfile 默认只能用于单线程，如果需要多线程，需要使用 multiprocessing 模块。
> - 它不统计“每一行代码”，而是“每个函数”。

cProfile 的使用方法如下：

cProfile 是 python 标准库的一部分，不需要安装。你可以直接引用，然后 run()方法来运行你的代码。

```python
import cProfile

def main():
    # 你的业务逻辑
    heavy_function()

def heavy_function():
    total = 0
    for i in range(1000000):
        total += i * i
    return total

if __name__ == '__main__':
    cProfile.run('main()', sort='cumulative')

```

示例输出片段

```console
   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.245    0.245 <string>:1(<module>)
        1    0.000    0.000    0.245    0.245 profile_test.py:5(main)
        1    0.245    0.245    0.245    0.245 profile_test.py:8(heavy_function)
```

ncalls 表示函数被调用的次数，tottime 表示函数内代码执行的总时间，percall 表示函数内代码平均执行时间，cumtime 表示函数被调用的累计时间，percall 表示函数被调用的累计时间平均值。

> 注意：
> 重点看 cumtime 高的函数，它们是性能瓶颈的“嫌疑人”。
