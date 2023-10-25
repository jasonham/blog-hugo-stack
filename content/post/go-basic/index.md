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

#### 普通枚举

``` go
const (
    cpp = 1
    java = 2
    python = 3
    golang = 4
)
```

#### 自增值枚举

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

#### 普通写法

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

#### 循环写法

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

## 循环

### for循环

#### 简单循环

``` go
 sum := 0
 for i := 1; i <= 100; i++ {
    sum += i
 }
```

#### 无起始条件

``` go
func convertToBin(n int) string {
    result := ""
    for ; n > 0; n /= 2 {
        lsb := n % 2
        result = strconv.Itoa(lsb) + result
        }
        return result
}
```

#### 无起始条件，无递增条件，只有结束条件

``` go
func printFile(filename string) {
    file, err := os.Open(filename)
    if err != nil {
        panic(err)
        }
        printFileContents(file)
}

func printFileContents(reader io.Reader) {
    scanner := bufio.NewScanner(reader)
    for scanner.Scan() {
        fmt.Println(scanner.Text())
        }
}
```

#### 无结束条件  

死循环

``` go
func forever() {
    for {
        fmt.Println("abc")
    }
}
```

## 函数

### func

#### 匿名函数  

func关键字后没有函数名

``` go
func(a int, b int) int {
    return int(math.Pow(
        float64(a), float64(b)))
        }, 3, 4)
```

#### 可变参数列表  

**...** 代表不定参数数量

``` go
func sum(numbers ...int) int {
    s := 0
    for i := range numbers {
        s += numbers[i]
        }
        return s
        }
```  

## 指针

* \* 代表指针
* &代表引用
  
## 数组

### 定义数组

数组是值类型

``` go
func main() {
    var arr1 [5]int
    arr2 := [3]int{1, 3, 5}
    arr3 := [...]int{2, 4, 6, 8, 10}  // 不标明个数，但不是切片
    var grid [4][5]int  
}
```

### 循环数组

``` go
for i, v := range arr {
    fmt.Println(i, v)
    }
```

## 切片（slice）

不是值类型，是数组的一个view  

* slices可以向后扩展，不可以向前扩展
* 注意区分len，cap的概念

### 定义切片

``` go
arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}  // 数组

fmt.Println("arr[2:6] =", arr[2:6])  // 切片
```

### slice的向后扩展

``` go
s1 = arr[2:6]
s2 = s1[3:5] // [s1[3], s1[4]] 对slice做了向后扩展
```

### slice的添加

* 添加元素的时候不用考虑原caps，超过的时候会自动分配新的数组
* 必须有接收对象

``` go
s3 := append(s2, 10)
s4 := append(s3, 11)
s5 := append(s4, 12)
```

### 创建slice

``` go
var s []int // Zero value for slice is nil
s1 := []int{2, 4, 6, 8}
s2 := make([]int, 16)
s3 := make([]int, 10, 32) // len 10 cap 32
```

### copy slice

``` go
copy(s2, s1)
```

### 删除 slice

#### 删除中间的元素

``` go
s2 = append(s2[:3], s2[4:]...)
```

#### 删除头尾

``` go
fmt.Println("Popping from front")
front := s2[0]
s2 = s2[1:]

fmt.Println("Popping from back")
tail := s2[len(s2)-1]
s2 = s2[:len(s2)-1]
```

## Map

### 定义map

``` go
m := map[string]string{
    "name":    "ccmouse",
    "course":  "golang",
    "site":    "imooc",
    "quality": "notbad",
    }

m2 := make(map[string]int) // m2 == empty map
var m3 map[string]int // m3 == nil

```

### 遍历map

``` go
 for k, v := range m {
    fmt.Println(k, v)
    }
```

### 获取value

``` go
courseName := m["course"]

if causeName, ok := m["cause"]; ok {
    fmt.Println(causeName)
    } else {
        fmt.Println("key 'cause' does not exist")
        }
```

### 删除元素

``` go
delete(m, "name")
```

## 字符串

中文转换

``` go
var ch = []rune(s)
for i, chr := range []rune(s) {}
```

## 面向对象

* 仅支持封装，不支持继承和多态
* go语言没有class，只有struct
* 不论地址还是结构本身，一律使用 ．来访问成员

结构定义

``` go
type TreeNode struct {
    Left, Right *TreeNode Value int
}
// 接收者方式
func (node TreeNode) print () {
    fmt. Print (node.Value)
}
// 使用指针作为方法接收者
// 只有使用指针才可以改变结构内容
func (node *TreeNode) setValue (value int ) {
    node.Value = value
}

// 使用自定义工厂函数
func createTreeNode (value int) *TreeNode {
    return &TreeNode{Value: value}
}
root.Left.Right = createTreeNode(3)

//  嵌套写法
func (node *treeNode) traverse() {
    if node == nil {
        return
        }
        node.left.traverse()
        node.print()
        node.right.traverse()
```

## 封装

* 名字一般使用CamelCase
* 首字母大写：public
* 首字母小写：private

## 扩充（继承）

### 组合

``` go
type myTreeNode struct {
    node *tree.Node
}

func (myNode *myTreeNode) postOrder () {
    if myNode == nil Il myNode. node == nil {
        return
        }
    left := myTreeNode{myNode. node.Left} 
    right := myTreeNode{myNode.node.Right}
    left.postOrder ()
    right.post0rder)
    myNode.node.Print ()
```

### 别名

``` go
type Queue []int

func (g *Queue) Push(v int) {
    *q = append (*q, v)
}
func (q *Queue) Pop() int {
    head := (*q)[0]
    *q = (*q)[1:]
    return head
}
func (q *Queue) IsEmpty() bool {
    return len (*q) == 0
}
```

## 接口

### 定义

``` go
type Retriever interface{
    Get(source string)string
}
func download (retriever Retriever) string {
    return retriever.Get ("www.baidu.com")
}
```

### 实现

``` go
type Retriever struct {
	Contents string
}

func (r *Retriever) String() string {
	return fmt.Sprintf(
		"Retriever: {Contents=%s}", r.Contents)
}

func (r *Retriever) Post(url string,
	form map[string]string) string {
	r.Contents = form["contents"]
	return "ok"
}

func (r *Retriever) Get(url string) string {
	return r.Contents
}

```

### 调用

``` go
fmt.Println(download(mock.Retriever{"this is fake baidu.com"}))
```
