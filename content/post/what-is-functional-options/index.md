---
title: Golang 中的 Functional Options(函数式选项)是什么？
description: Golang 基本语法的记录
date: 2025-03-03T22:56:12+08:00
lastmod: 2025-03-03T22:56:12+08:00
slug: go-basic

tags:
  - Golang
categories:
  - Golang
# draft: true
---

# Functional Options(函数式选项)

## 什么是Functional Options(函数式选项)

Functional Options(函数式选项)是一种设计模式，用于在函数调用时传递多个参数。
它主要解决一个问题,当一个对象有很多可选配置时，如何让构造函数既灵活、可扩展，又保持调用代码清晰。  
对于传统的实现，它还有一个优势是可以方便的实现默认配置。

## 函数式选项的实现

其核心概念是定义一个闭包函数类型，并在闭包中修改结构体指针。把每一个配置项变成一个函数。

```go
type Server struct {
    host string
    port int
}

// 定义 Option 函数签名
type Option func(*Server)

// 具体的配置函数
func WithHost(host string) Option {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

// 构造函数
func NewServer(opts ...Option) *Server {
    s := &Server{
        host: "localhost", // 默认值
        port: 8080,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

于是调用：

```go
server := NewServer(
	WithPort(9000),
	WithHost("192.168.1.1"),
)
```

优点

- 极其简洁灵活：新增配置项只需要写一个新的 WithXxx 函数，无需修改结构体或接口定义。

- 支持动态逻辑与校验：Option 函数内部不仅可以赋值，还能包含复杂逻辑、校验或者默认值重置。

- 平滑演进（向后兼容）：添加新的选项不会破坏现有的客户端代码。

- 组合性极佳：可以通过链式或组合多个 Option（例如 Option 数组合为一个大 Option）。

## 何时用

可选参数 ≥ 3 个、且未来可能增长、且需要默认值时。如果只有 1–2 个必填参数，直接传参更清楚。

## 进阶用法

在基础的 Functional Options 模式中，Option 函数通常是无返回值的（func(\*Server)）。当涉及复杂参数校验（如端口范围检查、地址格式验证、必填项校验）或外部资源加载（如读取配置文件、建立连接）时，直接在闭包里 panic 显然不够优雅。

### 使用签名返回 error

直接修改 Option 的函数签名，使其返回 error。

```go
type Server struct {
    host string
    port int
}

// Option 函数返回 error
type Option func(*Server) error

func WithHost(host string) Option {
    return func(s *Server) error {
        if host == "" {
            return errors.New("host cannot be empty")
        }
        s.host = host
        return nil
    }
}

func WithPort(port int) Option {
    return func(s *Server) error {
        if port < 1 || port > 65535 {
            return fmt.Errorf("invalid port %d: must be between 1 and 65535", port)
        }
        s.port = port
        return nil
    }
}

// 构造函数：遍历应用 Option，遇到错误立即中断返回
func NewServer(opts ...Option) (*Server, error) {
    s := &Server{
        host: "localhost", // 默认值
        port: 8080,
    }

    for _, opt := range opts {
        if err := opt(s); err != nil {
            return nil, fmt.Errorf("apply option failed: %w", err)
        }
    }

    return s, nil
}
```
