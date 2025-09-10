---
title: 如何实现带参数和不带参数通用的装饰器(decorator)
description: 这是一个 Python 装饰器的经典问题：既能 不带参数 使用，也能 带参数 使用。
date: 2023-01-11T21:47:14+08:00
lastmod: 2023-01-11T21:47:14+08:00
slug: how-to-implement-general-decorator

tags:
  - Python
  - Decorator
categories:
  - Python
---

> 这是一个 Python 装饰器的经典问题：既能 不带参数 使用，也能 带参数 使用。

## 基础思路

区别在于：

- **不带参数**时：`@decorator` → 直接传入函数。
- **带参数**时：`@decorator(x=1)` → 先传入参数，返回一个真正的装饰器。

所以需要写一个 **兼容两种调用方式的装饰器工厂**。

---

## 通用实现模板

```python
import functools

def my_decorator(_func=None, *, param1=None, param2=None):
    """
    通用装饰器：既支持 @my_decorator 也支持 @my_decorator(param1=xx)
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print(f"参数: param1={param1}, param2={param2}")
            return func(*args, **kwargs)
        return wrapper

    # 如果直接是 @my_decorator，_func 就是被装饰的函数
    if _func is not None:
        return decorator(_func)

    # 如果是 @my_decorator(...)，则返回真正的装饰器
    return decorator
```

---

## 使用示例

```python
@my_decorator
def foo():
    print("执行 foo")

@my_decorator(param1=123)
def bar():
    print("执行 bar")

foo()
# 输出:
# 参数: param1=None, param2=None
# 执行 foo

bar()
# 输出:
# 参数: param1=123, param2=None
# 执行 bar
```

---

## 总结

- **关键点**：用 `_func=None` 区分是否直接传入函数。
- **无参数调用**：`@my_decorator` → `_func` 是函数对象。
- **有参数调用**：`@my_decorator(param1=...)` → `_func=None`，返回一个 `decorator`。

---
