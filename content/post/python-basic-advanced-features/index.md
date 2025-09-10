---
title: Python 语言基础与高级特性
description: Python装饰器、上下文管理器、元类、描述符等高级用法
date: 2023-01-10T21:03:30+08:00
lastmod: 2023-01-10T21:03:30+08:00
slug: python-basic-advanced-features

tags:
  - Python
categories:
  - Python
draft: true
---

这部分内容是 Python 高级编程的核心。它们不仅体现语言特性的掌握深度，更关系能否写出**可复用、可扩展、可维护**的高质量代码。

我们逐个详细解释：

## Python 装饰器（Decorator）

### 是什么？

装饰器是一种**语法糖**，用于在不修改原函数代码的前提下，**增强或修改函数的行为**。本质是一个“高阶函数”——接收函数作为参数，并返回一个新函数。

### 基本结构

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before function call")
        result = func(*args, **kwargs)
        print("After function call")
        return result
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# 输出：
# Before function call
# Hello!
# After function call
```

### 带参数的装饰器

```python
def repeat(n):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet():
    print("Hi!")

greet()
# 输出三次 "Hi!"
```

### 类装饰器

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Called {self.count} times")
        return self.func(*args, **kwargs)

@CountCalls
def say_hi():
    print("Hi")

say_hi()  # Called 1 times
say_hi()  # Called 2 times
```

### 常见应用场景

- 日志记录
- 权限校验
- 缓存（如 `@functools.lru_cache`）
- 性能计时
- 重试机制
- 接口限流

### 相关问题

- 装饰器如何保持原函数的元信息？→ 使用 `@functools.wraps(func)`
- 多个装饰器的执行顺序？→ **从下到上装饰，从上到下执行**
- 如何实现带参数和不带参数通用的装饰器？

---

## 上下文管理器（Context Manager）

### 是什么？

用于管理资源的“进入”和“退出”逻辑，确保资源（如文件、锁、数据库连接）**一定被释放**，即使发生异常。

最常见语法：`with` 语句。

### 实现方式一：类 + `__enter__` / `__exit__`

```python
class MyFileHandler:
    def __init__(self, filename):
        self.filename = filename
        self.file = None

    def __enter__(self):
        print("Opening file...")
        self.file = open(self.filename, 'w')
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Closing file...")
        if self.file:
            self.file.close()
        # 返回 True 表示“异常已处理”，不向上抛出
        if exc_type:
            print(f"Exception occurred: {exc_val}")
        return False  # 不压制异常

with MyFileHandler('test.txt') as f:
    f.write("Hello Context Manager!")
    # raise ValueError("Oops!")  # 可测试异常是否被处理
```

### 实现方式二：使用 `@contextlib.contextmanager` 装饰器（推荐用于简单场景）

```python
from contextlib import contextmanager

@contextmanager
def my_timer():
    import time
    start = time.time()
    yield  # 这里相当于 __enter__ 和 __exit__ 的分界
    end = time.time()
    print(f"Execution time: {end - start:.2f}s")

with my_timer():
    sum(i for i in range(1000000))
```

### 应用场景

- 文件操作（自动关闭）
- 数据库事务（自动 commit/rollback）
- 锁管理（自动 acquire/release）
- 计时、日志、临时环境变量修改

### 相关问题

- `__exit__` 的三个参数是什么？→ `exc_type`, `exc_value`, `traceback`
- 如何在 `__exit__` 中“吞掉”异常？→ `return True`
- `contextmanager` 装饰器内部原理？→ 基于生成器 + try/finally

---

## 元类（Metaclass）

### 是什么？

“类的类”。默认情况下，类是由 `type` 创建的。元类允许你**自定义类的创建过程**。

类 → 实例  
元类 → 类

### 基本用法

```python
class MyMeta(type):
    def __new__(cls, name, bases, attrs):
        print(f"Creating class {name}")
        # 可以在这里修改类属性、方法等
        attrs['added_by_meta'] = True
        return super().__new__(cls, name, bases, attrs)

class MyClass(metaclass=MyMeta):
    pass

# 输出：Creating class MyClass
print(MyClass.added_by_meta)  # True
```

### 更实用的例子：自动注册子类

```python
class PluginMeta(type):
    plugins = {}

    def __new__(cls, name, bases, attrs):
        new_class = super().__new__(cls, name, bases, attrs)
        if name != 'BasePlugin':  # 不注册基类
            cls.plugins[name] = new_class
        return new_class

class BasePlugin(metaclass=PluginMeta):
    pass

class EmailPlugin(BasePlugin):
    def send(self): pass

class SMSPlugin(BasePlugin):
    def send(self): pass

print(PluginMeta.plugins)
# {'EmailPlugin': <class '__main__.EmailPlugin'>, 'SMSPlugin': <class '__main__.SMSPlugin'>}
```

### 应用场景

- ORM 框架（如 Django Model、SQLAlchemy）自动建表
- 插件系统自动注册
- 接口强制规范（如要求所有子类实现某个方法）
- 单例模式、抽象基类控制

### 相关问题

- `__new__` 和 `__init__` 在元类中的区别？
  - `__new__` 创建类对象，`__init__` 初始化类对象
- 为什么说“元类是深水区”？→ 滥用会导致代码难以理解和调试
- Django Model 是如何使用元类的？→ `ModelBase` 收集字段、建立映射、注册到 apps

---

## 描述符（Descriptor）

### 是什么？

描述符是实现了**特定协议方法**（`__get__`, `__set__`, `__delete__`）的对象，用于**自定义属性访问行为**。

它是 Python 面向对象中“属性管理”的底层机制，`@property`、`classmethod`、`staticmethod` 都是基于描述符实现的。

### 基本结构

```python
class NonNegative:
    def __init__(self, name):
        self.name = name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name, 0)

    def __set__(self, instance, value):
        if value < 0:
            raise ValueError("Value cannot be negative")
        instance.__dict__[self.name] = value

class Product:
    price = NonNegative('price')
    quantity = NonNegative('quantity')

    def __init__(self, name, price, quantity):
        self.name = name
        self.price = price
        self.quantity = quantity

p = Product("Apple", 5, 10)
print(p.price)  # 5
p.price = -1    # ValueError: Value cannot be negative
```

### 数据描述符 vs 非数据描述符

- **数据描述符**：实现了 `__set__` 或 `__delete__` → 优先级高于实例字典
- **非数据描述符**：只实现 `__get__` → 实例字典优先

### 应用场景

- 数据校验（如非负、类型检查、范围限制）
- 延迟计算属性（懒加载）
- ORM 字段定义（如 Django Field）
- 缓存属性值（避免重复计算）

### 相关问题

- 描述符和 `@property` 的关系？→ `@property` 是描述符的语法糖
- 为什么 `instance.__dict__[name] = value` 而不是 `setattr(instance, name, value)`？→ 避免无限递归调用 `__set__`
- 描述符是类属性，为什么能管理不同实例的值？→ 通过 `instance` 参数区分不同对象

---

## 总结对比表

| 特性         | 用途               | 触发时机             | 典型应用场景            |
| ------------ | ------------------ | -------------------- | ----------------------- |
| 装饰器       | 增强函数/类行为    | 函数定义/调用时      | 日志、缓存、权限        |
| 上下文管理器 | 管理资源生命周期   | with 语句进入/退出时 | 文件、锁、事务          |
| 元类         | 控制类的创建过程   | 类定义时             | ORM、插件注册、强制规范 |
| 描述符       | 自定义属性访问行为 | 属性读/写/删除时     | 数据校验、延迟计算、ORM |

---

掌握这四个高级特性，不仅能让你写出更优雅、健壮的代码，更能体现你对 Python 语言设计哲学的理解 —— “**简洁但不简单，灵活但有章法**”。

它们是区分“普通开发者”和“资深架构师”的重要分水岭。
