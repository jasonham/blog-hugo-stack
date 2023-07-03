---
title: Golang 基本语法
description: Golang 基本语法的记录
date: 2023-07-03T22:56:12+08:00
lastmod: 2023-07-03T22:56:12+08:00
slug: go-basic

tags:
 - Golang
categories:
 - Golang
# draft: true
---

## 打印变量

``` golang
fmt.Println("hello world")
```

## 定义变量

### 空变量

```golang
var i int
var s string
fmt.Printf("%d %q\n", i, s)
```

### 变量赋值

``` golang
var a, b int = 3, 4
var s string = "abc"
fmt.Pringln(a, b, s)
```

### 变量类型推断

``` golang
var a, b, c, s = 3, 4, ture, "def"
fmt.Pringln(a, b, c, s)
```

### 变量类型推断简写  

<mark>**推荐写法**</mark>，但是不适合用于函数外使用

``` golang
a, b, c, s := 3, 4, ture, "def"
b = 5
fmt.Pringln(a, b, c, s)
```

包内推荐

``` golang
var(
    aa = 11
    bb = "kkk"
    ss = true
)
```

## 变量类型

内建变量类型

* bool, string
* (u)int, (u)int8, (u)int16, (u)int32, (u)int64, uintptr # 有u为无符号，uintptr为指针
* byte, rune # rune相当于char类型
* float32, float64, complex64, complex128  # complex代表复数
* 类型转为需要显示强制转换

## 定义常量

### 常量赋值

定义常量,可以指定类型，可以自动替换**不是推断**

``` go
const filename string="abc.txt"
const a, b = 3, 4
```

### 枚举类型

普通枚举

``` go
const (
    cpp = 1
    java = 2
    python = 3
    golang = 4
)
```

自增值枚举

``` go
const (
    cpp = iota
    -
    python
    golang
)
```

iota可以参与运算

``` go
// b, kb, mb, gb, tb, pb
const (
    b = 1 << (10 * iota)
    kb
    mb
    gb
    tb
    pb
)
```

## 条件语句

### if

普通写法

``` go
func bounded(v int) int {
    if v > 100 {
        return 100
    } else if v < 0 {
        return 0
    } else {
        return v
    }
}

```

循环写法

``` go
const filename = "abc.txt"
if contents, err := ioutil.ReadFile(filename); err != nil {
    fmt.Println(err)
} else {
    fmt.Printf("%s\n", contents)
}
```

### switch

* switch 有表达式的写法

``` go
func eval(a, b int, op string) int {
    var result int
    switch op {
    case "+":
        result = a + b
    case "-":
        result = a - b
    case "*":
        result = a * b
    case "/":
        result = a / b
    default:
        panic("unsupported operator:" + op)
 }
 return result
}
```

* switch 没有表达式的写法

``` go
func grade(score int) string {
    g := ""
    switch {
    case score < 0 || score > 100:
        panic(fmt.Sprintf(
            "Wrong score: %d", score))
    case score < 60:
        g = "F"
    case score < 80:
        g = "C"
    case score < 90:
        g = "B"
    case score <= 100:
        g = "A"
}
    return g
}
```
