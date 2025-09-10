---
title: 异步编程：async/await、asyncio事件循环、协程调度、常见陷阱（如阻塞调用、线程安全）
description: 关于异步编程的一些知识点
date: 2023-04-02T22:25:01+08:00
lastmod: 2024-05-24T22:25:01+08:00
slug: asynchronous-programming

tags:
  - asynchronous
  - async
  - asyncio
categories:
  - Python
---

**async/await、asyncio 事件循环、协程调度、常见陷阱（如阻塞调用、线程安全）**。这些是 Python 异步编程（尤其是 asyncio）的核心组成部分，理解它们对写出高效、健壮的异步程序至关重要。

---

## async/await —— 异步语法糖

### 基本概念

`async/await` 是 **Python 3.5+** 引入的语法(Django 4.x 才完全支持异步数据库操作)，用于定义和调用**协程（coroutine）**。它们让异步代码看起来像同步代码，极大提升了可读性和可维护性。

- `async def` 定义一个协程函数。
- `await` 用于“等待”一个可等待对象（awaitable）完成，如另一个协程、Future、Task 等。
- `await` 只能在 `async` 函数中使用。

### 示例

```python
import asyncio

async def fetch_data():
    print("开始获取数据...")
    await asyncio.sleep(2)  # 模拟 I/O 操作，非阻塞
    print("数据获取完成")
    return "data"

async def main():
    result = await fetch_data()
    print(result)

asyncio.run(main())
```

输出：

```
开始获取数据...
（等待2秒）
数据获取完成
data
```

### 关键点

- `await asyncio.sleep(2)` 不会阻塞整个程序，而是让出控制权，事件循环可以去执行其他任务。
- `asyncio.run()` 是 Python 3.7+ 推荐的启动异步程序入口。

---

## asyncio 事件循环（Event Loop）

### 什么是事件循环？

事件循环是 asyncio 的核心，它负责**调度和执行协程**。你可以把它想象成一个“任务调度器”，它不断轮询哪些协程可以继续执行（比如 I/O 完成、定时器到期等），然后恢复它们的执行。

### 工作机制简述

1. 你通过 `asyncio.run(main())` 启动事件循环。
2. `main()` 被包装成 Task 加入事件循环。
3. 当遇到 `await`，当前协程暂停，事件循环去执行其他就绪的协程或等待 I/O。
4. 当被 `await` 的对象完成（如 sleep 结束、网络请求返回），事件循环恢复该协程。
5. 所有任务完成后，事件循环退出。

### 示例：多个协程并发执行

```python
import asyncio

async def task(name, delay):
    print(f"任务 {name} 开始")
    await asyncio.sleep(delay)
    print(f"任务 {name} 完成")

async def main():
    await asyncio.gather(
        task("A", 2),
        task("B", 1),
        task("C", 3)
    )

asyncio.run(main())
```

输出顺序可能是：

```
任务 A 开始
任务 B 开始
任务 C 开始
任务 B 完成   ← 1秒后
任务 A 完成   ← 2秒后
任务 C 完成   ← 3秒后
```

> 注意：虽然任务是“并发”执行的，但不是“并行”（单线程），靠的是 I/O 等待时切换任务。

---

## 协程调度（Coroutine Scheduling）

### 什么是协程调度？

协程调度是指事件循环如何决定**哪个协程在什么时候恢复执行**。在 asyncio 中，调度是协作式的（cooperative）：

- 协程必须主动让出控制权（通过 `await`）才能被切换。
- 没有抢占式调度 —— 如果一个协程长时间不 `await`，会阻塞整个事件循环！

### 调度单位：Task

`asyncio.create_task(coro)` 或 `asyncio.gather()` 会把协程包装成 **Task**，Task 是事件循环调度的基本单位。

```python
async def main():
    task1 = asyncio.create_task(fetch_data())
    task2 = asyncio.create_task(fetch_data())
    await task1
    await task2
```

### 调度策略

- FIFO（先进先出）为主，但受 I/O 事件、定时器等影响。
- `await` 时挂起当前 Task，事件循环选择下一个可运行的 Task。
- 所有 Task 都在同一个线程中交替运行（单线程并发）。

---

## 常见陷阱（Traps）

异步编程虽然强大，但容易踩坑。以下是几个最常见的陷阱：

---

### 陷阱 1：阻塞调用（Blocking Calls）

#### 错误示例：

```python
import time
import asyncio

async def bad_func():
    time.sleep(3)  # ← 阻塞！整个事件循环卡住3秒
    return "done"

async def main():
    task1 = asyncio.create_task(bad_func())
    task2 = asyncio.create_task(bad_func())
    await task1
    await task2

asyncio.run(main())
```

> 输出：两个任务**串行执行**，总耗时 6 秒 —— 完全失去了并发优势！

#### 正确做法：

- 使用异步版本（如 `asyncio.sleep` 代替 `time.sleep`）
- 或者把阻塞操作放到线程池中：

```python
import asyncio

async def good_func():
    await asyncio.sleep(3)  # 非阻塞
    return "done"

# 或者使用线程池运行阻塞函数
def blocking_io():
    time.sleep(3)

async def run_in_thread():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_io)
    return result
```

> `loop.run_in_executor` 使用线程池运行阻塞函数，不阻塞事件循环。

---

### 陷阱 2：忘记 await

#### 错误示例：

```python
async def fetch():
    return "data"

async def main():
    fetch()  # ← 忘记 await！协程根本没执行
    print("done")
```

> 输出：只打印 "done"，`fetch()` 没有被调用（只是创建了协程对象）。

#### 正确做法：

```python
result = await fetch()
```

> 或者用 `create_task()` 并后续 `await`：

```python
task = asyncio.create_task(fetch())
# 做点别的事...
result = await task
```

---

### 陷阱 3：在非异步上下文中调用协程

#### 错误示例：

```python
async def coro():
    return 1

def sync_func():
    coro()  # ← 在同步函数中直接调用协程 → 无效果 + 警告！

sync_func()
```

> 输出：`RuntimeWarning: coroutine 'coro' was never awaited`

#### 正确做法：

- 在异步函数中 `await coro()`
- 或者在同步函数中使用 `asyncio.run(coro())`（仅限顶层）

---

### 陷阱 4：线程安全问题（Thread Safety）

asyncio 默认是**单线程**的，所有协程都在同一线程中运行，因此：

- 通常不需要担心竞态条件（race condition）。
- 但如果你混合使用多线程（如 `run_in_executor`、`concurrent.futures`），就可能出现线程安全问题！

#### 示例：

```python
counter = 0

async def increment():
    global counter
    tmp = counter
    await asyncio.sleep(0.001)  # 模拟切换
    counter = tmp + 1

async def main():
    tasks = [increment() for _ in range(100)]
    await asyncio.gather(*tasks)
    print(counter)  # 期望100？实际可能小于100！

asyncio.run(main())
```

> 为什么？因为 `await asyncio.sleep(...)` 让出控制权，多个协程同时读写 `counter`，导致竞态！

#### 解决方案：

- 使用 `asyncio.Lock()`：

```python
lock = asyncio.Lock()

async def increment():
    global counter
    async with lock:
        tmp = counter
        await asyncio.sleep(0.001)
        counter = tmp + 1
```

> 或者避免共享状态 —— 使用消息传递、队列等方式。

---

### 陷阱 5：在协程中启动新事件循环

#### 错误示例：

```python
async def nested():
    asyncio.run(another_coro())  # ← 在已有事件循环中再启动一个！

async def main():
    await nested()
```

> 报错：`RuntimeError: asyncio.run() cannot be called from a running event loop`

#### 正确做法：

- 使用 `await another_coro()` 直接等待
- 或者用 `asyncio.create_task()` + `await`

---

## 总结对比表

| 概念          | 说明                                   | 常见错误/注意点                  |
| ------------- | -------------------------------------- | -------------------------------- |
| `async/await` | 定义和等待协程的关键字                 | 忘记 await、在同步函数中调用协程 |
| 事件循环      | 调度协程的核心引擎                     | 阻塞调用会卡住整个循环           |
| 协程调度      | 协作式调度，靠 await 让出控制权        | 长时间计算不 await 会导致“假死”  |
| 阻塞调用      | 如 `time.sleep`, `requests.get`        | 必须用异步替代或放到线程池       |
| 线程安全      | 单线程内协程安全；混合线程时需加锁     | 共享变量在多协程中需用 Lock 保护 |
| 启动嵌套循环  | `asyncio.run()` 只能在无事件循环时调用 | 在协程内不能再调 `asyncio.run()` |

---

## 最佳实践建议

1. **优先使用异步库**：如 `aiohttp` 代替 `requests`，`aiomysql` 代替 `pymysql`。
2. **CPU 密集型任务交给线程/进程池**：用 `loop.run_in_executor`。
3. **共享状态用 asyncio.Lock 保护**。
4. **避免在协程中写阻塞代码**。
5. **使用类型注解和 lint 工具**（如 mypy, pylint）帮助发现未 await 的协程。
6. **测试异步代码时，使用 `pytest-asyncio`**。

---

## 参考来源

- [Python asyncio 官方文档](https://docs.python.org/3/library/asyncio.html)
- 《Python 异步编程实战》
- David Beazley 的 “Understanding AsyncIO” 演讲（YouTube）

---
