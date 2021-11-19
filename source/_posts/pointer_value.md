title: 指针和值
tags: [goland]
categories:
  - 编程语言
date: 2021-07-24 15:51:00
---
# 指针
## 值和指针
值的声明直接写类型即可，如int
指针的声明需要在类型前加*，如*int
```
`var v int = 1  // 开辟的一块空间，名字为v, 空间的值为1`
var p *int = &v  // 获取v的地址存到名为p的内存空间，*int 指向int类型的指针
```
![&和*的操作关系](http://assets.processon.com/chart_image/6016be58e401fd15813cfba8.png?_=1636815032732)
## 指针操作
* 操作符&(取址符) : 是返回该变量的内存地址
* 操作符* (取值符) : 是返回该指针指向的变量的值, 同时也可以进行修改指针指向内存地址的值
**取地址操作符&和取值操作符\*是一对互补操作符**
````
package main

import (
    "fmt"
)

func  TestGetAddAndVal(t *testing.T)  {

    // 准备一个字符串类型
    var house = "Malibu Point 10880, 90265"

    // 对字符串取地址, ptr类型为*string
    ptr := &house

    // 打印ptr的类型
    fmt.Printf("ptr type: %T\n", ptr)  // ptr type: *string

    // 打印ptr的指针地址
    fmt.Printf("address: %p\n", ptr) // address: 0xc0420401b0

    // 对指针进行取值操作
    value := *ptr

    // 取值后的类型
    fmt.Printf("value type: %T\n", value) // value type: string

    // 指针取值后就是指向变量的值
    fmt.Printf("value: %s\n", value) // value: Malibu Point 10880, 90265

}
````
## 指针细节
### 指针 赋值

go 的函数和接受者都只有值传递方式，不存在引用传递。指针传递也是传递指针的地址
以数值交换为例
1. 使用多重赋值进行值交换
```
package main

import "fmt"

func swap(a, b int) {
    b, a = a, b
     fmt.Println(b, a)
         fmt.Println(&b, &a) 
}

func main() {
    x, y := 1, 2
    swap(x, y)
    fmt.Println(x, y)
}
```
2. 交换指针
```

package main

import "fmt"

func swap(a, b *int) {
    b, a = a, b
}

func main() {
    x, y := 1, 2
    swap(&x, &y)
    fmt.Println(x, y)
}
```
## 接受者指针和值
指针不仅可以作为函数的入参和返回值，也可以作为接受者
接收者可以为指针和值，指针会拷贝指针地址给到接收者，值会重新拷贝值给到接收者
```
package pointer

type Project struct {
    Code int
}
func (p Project) printVal() {
    // 拷贝一个新的Project给p，不省内存
    fmt.Printf("p: type:%T, add:%p\n", p, &p) // 地址和origin 不一样
}
func (p *Project) printProinter() {
    // 拷贝一个新的地址给p
    fmt.Printf("p: type:%T, add:%p\n", p, p) // 地址和origin 一样
}
func TestReceiver(t *testing.T) {
    p := Project{}
    fmt.Printf("origin p: type:%T, add:%p\n", p, &p)
    p.printVal()
    p.printProinter()
}
```
*  指针接收者节省内存，，结构体很大时，性能比较好，应用运行通过指针修改变量的场景
* 值接收者可以实现每个接收者都是全新的拷贝，可以实现不可变型的变量，提高代码的安全性，但是占用内存比较高

## 实现不可变型结构体
由于go值传递的特性，通过设置接收者的可见性为包内，并且通过值作为接受者进行拷贝新的值来实现不可变。废话不多说，直接上代码
```

// 指针 -> 修改原来地址的值
// 值 -> 没法修改原来地址的值
// 不可变型: 值一旦被创建就无法被改变对象,修改通过生成一个新的对象来实现
type Person struct {
    name           string // 包内可见，避免被修改
}
func (p Person) WithName(newname string) Person { // 拷贝新的结构体，返回新的结构体
    p.name = newname
    fmt.Printf("WithName p's add:%p\n", &p) // 打印地址 0xc000064540
    return p
}
func (p Person) Name() string { // getter
    return p.name
}
// 值接收者实现不可变型
func TestImmutableForm(t *testing.T) {
    me := Person{} // 默认初始化
    fmt.Printf("me data:%+#v\n", me.Name())
    fmt.Printf("add:%p, data:%+#v\n", &me, me) // 0xc0000644e0
    m2 := me.WithName("Elliot")
    m2.Name()
    fmt.Printf("m2 data:%+#v\n", m2.Name())
}
```