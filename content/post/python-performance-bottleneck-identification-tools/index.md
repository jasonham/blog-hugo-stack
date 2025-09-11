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

## line_profiler

`line_profiler` 是一个强大的 Python 代码逐行性能分析工具。与内置的 `cProfile` 不同，`line_profiler` 能够精确到每一行代码的执行时间，帮助你快速找出性能瓶颈。特别适合优化“函数内部哪一行最慢”。

> 注意：它需要你手动标记要分析的函数（用装饰器 @profile），不能全局分析。

### 1\. 安装

首先，你需要使用 `pip` 安装 `line_profiler`：

```bash
pip install line_profiler
```

---

### 2\. 标记要分析的函数

你需要使用 `@profile` 装饰器来标记你想要分析的函数。这个装饰器来自 `kernprof.py` 脚本，因此即使你没有直接导入它，只要你使用正确的运行方式，它也能正常工作。

**示例代码 (test_script.py):**

```python
import time

@profile
def slow_function_a():
    """一个模拟的慢函数 A"""
    total = 0
    for i in range(1000):
        total += i
        time.sleep(0.0001)  # 模拟一些耗时的操作
    return total

def fast_function_b():
    """一个模拟的快函数 B"""
    total = 0
    for i in range(100000):
        total += i
    return total

def main():
    print("开始执行...")
    slow_function_a()
    fast_function_b()
    print("执行完毕。")

if __name__ == "__main__":
    main()
```

---

### 3\. 运行分析器

要运行分析，你需要使用 `kernprof.py` 脚本。它会生成一个以 `.lprof` 为后缀的文件，其中包含了分析结果。

```bash
# -v: 详细模式，运行后直接打印结果到控制台
# -l: 启用逐行分析
# 脚本名称: 你要分析的 Python 文件
kernprof -v -l test_script.py
```

执行上述命令后，你会看到类似下面的输出：

```text
开始执行...
执行完毕。
Wrote profile results to test_script.py.lprof
Timer unit: 1e-06 s

Total time: 0.10651 s
File: test_script.py
Function: slow_function_a at line 5

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     5                                           @profile
     6     def slow_function_a():
     7     """一个模拟的慢函数 A"""
     8       total = 0
     9       for i in range(1000):
    10         total += i
    11         time.sleep(0.0001)
    12     return total
```

### 4\. 解读分析结果

分析结果表格提供了非常详细的信息：

- **Line \#**：行号。
- **Hits**：该行代码被执行的次数。
- **Time**：该行代码总共花费的累积时间（以微秒为单位）。
- **Per Hit**：每次执行该行代码的平均时间。
- **% Time**：该行代码占整个函数总运行时间的百分比。

从上面的示例中，你可以清楚地看到 `time.sleep(0.0001)` 这一行占用了绝大部分时间，这正是我们想要分析出来的瓶颈。

---

### 5\. 查看结果文件

如果你没有使用 `-v` 参数，`kernprof.py` 只会生成 `.lprof` 文件。你可以使用 `line_profiler` 提供的 `lpstat` 脚本来查看结果。

```bash
# 将结果输出到控制台
python -m line_profiler test_script.py.lprof
```

### 总结

`line_profiler` 的使用流程非常直观：

1.  **安装**：`pip install line_profiler`。
2.  **标记**：在你想分析的函数前加上 `@profile` 装饰器。
3.  **运行**：使用 `kernprof -v -l your_script.py` 命令。
4.  **分析**：查看输出结果，找出 `% Time` 最高的行。

这种逐行分析的方法非常适合诊断那些由特定几行代码导致的性能问题。

## `memory_profiler` 是一个用于分析 Python 代码内存使用情况的工具，它可以精确到每行代码。这对于找出内存泄漏或内存使用过高的代码非常有用。

以下是 `memory_profiler` 的详细使用方法：

### 1\. 安装

首先，你需要使用 pip 安装 `memory_profiler` 和它的依赖 `psutil`：

```bash
pip install memory-profiler
```

`psutil` 是 `memory_profiler` 依赖的一个库，用于获取系统进程信息。

---

### 2\. 标记要分析的函数

要分析代码，你需要使用 `@profile` 装饰器来标记你想监控的函数。这个装饰器由 `memory_profiler` 模块提供，所以你需要在脚本中导入它。

**示例代码 (my_script.py):**

```python
import time
from memory_profiler import profile

# 这是一个不会被分析的函数
def create_list_of_ints():
    return [i for i in range(1000000)]

@profile
def my_function():
    a = [1] * (10 ** 6)
    b = [2] * (2 * 10 ** 7)
    del b
    c = [3] * (10 ** 6)

    # 调用一个没有被 @profile 装饰的函数
    list_of_ints = create_list_of_ints()

if __name__ == '__main__':
    my_function()
```

在这个例子中，只有 `my_function` 会被分析。`Notes_of_ints` 函数虽然被调用，但由于没有 `@profile` 装饰，它的内存使用不会被逐行记录。

---

### 3\. 运行分析器

有两种主要的方式来运行 `memory_profiler`：

#### 方法 A：直接运行脚本

在终端中，使用 `-m` 参数运行 `python`。这是最推荐的方法，因为它非常简单。

```bash
python -m memory_profiler my_script.py
```

运行后，你会看到类似下面的输出：

```text
Filename: my_script.py

Line #    Mem usage    Increment   Line Contents
================================================
     7   26.250 MiB   26.250 MiB   @profile
     8                             def my_function():
     9   33.816 MiB    7.566 MiB       a = [1] * (10 ** 6)
    10  186.273 MiB  152.457 MiB       b = [2] * (2 * 10 ** 7)
    11   33.816 MiB -152.457 MiB       del b
    12   41.383 MiB    7.566 MiB       c = [3] * (10 ** 6)
    13
    14   49.035 MiB    7.652 MiB       list_of_ints = create_list_of_ints()
    15
```

#### 方法 B：作为可执行脚本运行

如果你已经将 `memory_profiler` 所在的目录添加到系统路径中，你也可以直接运行它：

```bash
mprof run my_script.py
```

这种方法会生成一个名为 `mprof_*.dat` 的数据文件。你可以使用 `mprof` 命令来绘制内存使用图表，这对于可视化内存波动非常有用。

```bash
# 绘制内存使用图表
mprof plot
```

这会生成一个 `.png` 格式的图表，清晰地展示程序运行时的内存变化。

---

### 4\. 解读分析结果

分析结果表格提供了关键信息：

- **Line \#**：行号。
- **Mem usage**：执行到该行代码时，**整个进程**的内存使用总量（以 MiB 为单位）。
- **Increment**：执行该行代码后，**内存使用量的变化**（增加或减少）。这个值通常是你最关心的，因为它直接告诉你该行代码消耗了多少内存。
- **Line Contents**：具体的代码内容。

从上面的示例中，我们可以清楚地看到：

- 第 9 行创建列表 `a` 增加了 `7.566 MiB` 的内存。
- 第 10 行创建列表 `b` 增加了 `152.457 MiB` 的内存，这显示了它是一个非常大的内存消耗者。
- 第 11 行 `del b` 成功释放了这部分内存，`Increment` 变为负值。
- 第 12 行创建列表 `c` 再次增加了 `7.566 MiB`。
- 第 14 行调用 `Notes_of_ints()` 导致内存再次增加。

### 总结

`memory_profiler` 是一个非常实用的内存分析工具，它的使用流程很简单：

1.  **安装**：`pip install memory-profiler`。
2.  **标记**：在你想分析的函数前加上 `@profile` 装饰器。
3.  **运行**：使用 `python -m memory_profiler your_script.py` 或 `mprof run your_script.py`。
4.  **分析**：查看输出或生成的图表，重点关注 **Increment** 列来找出内存消耗大户。
