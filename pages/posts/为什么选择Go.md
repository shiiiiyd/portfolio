---
title: Go中代码测试和性能测试
date: 2021/10/17
description: Go中的代码测试和性能测试简单方法
tag: Go
author: shiyd
---

直接编译成二进制，没有虚拟化损失；自带运行环境，无需处理GC问题；一次编译可以适用多种平台；超强的并发支持和易用性。

## 何为Runtime？

runtime就是程序的运行环境，作为程序的一部分打包进二进制产物。随用户程序一起运行，与用户程序没有明显界限，直接通过函数调用。

go的runtime负责内存管理，垃圾回收，协程调度。**runtime的能力：**

- 内存管理的能力
- 垃圾回收的能力
- 超强的并发能力
- runtime有屏蔽系统调用的能力，不同系统调用的方式不一样，runtime处理不同系统调用
- 一些go关键字是runtime下的函数

## Go的编译过程

首先是执行 compile.exe 生成.a 文件，然后执行 link.exe。

词法分析→句法分析→语义分析 →中间码生成 →代码优化 →机器码生成 →链接(将各个包进行链接)

### **词法分析：**

- 将源代码翻译成Token
- Token是代码中的最小语义结构，如下代码中的 package、main、import就是token：

```go
package main

import "fmt"

fun main(){
	fmt.Println("hello, 世界")
}
```

### **句法分析：**

- token序列经过处理，变成语法树（SST）

### **语义分析：**

- 类型推断和类型检查
- 查看类型是否匹配
- 函数调用内联
- 逃逸分析

### **中见码生成：**

- 为了处理不同平台的差异，生成中间代码（SSA）
- 查看从代码到SSA中间码的整个过程

```bash
# win
env:GOSSAFUNC="main"
# linux
export GOSSAFUNC=main
# build 后生成一个ssa.html 文件
go build
```

![Untitled](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/4399edba-6728-404d-944d-1c18d101a655/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20230105%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20230105T095230Z&X-Amz-Expires=86400&X-Amz-Signature=ed068ba192b141f8abf6bfeba08010d938364df1808b02704488ae941c20c812&X-Amz-SignedHeaders=host&response-content-disposition=filename%3D%22Untitled.png%22&x-id=GetObject)

### 机器码生成

- 先生成Plan9汇编代码
- 最后编译为机器码
- 输出机器码为.a文件

查看Plan9汇编代码：

```bash
go build -gcflags -S main.go
```

### 链接

将各个包进行链接，包括runtime

## Go语言是如何运行的？

Go程序的入口不是main，而是runtime/rt0_XXX.s。不管是win还是linux都要走 _*rt0_amd64(SB) 这个方法。*

1、读取命令行参数：复制参数数量argc和参数值argv到栈上

2、初始化g0执行栈：g0是为了调度协程而产生的协程，g0是每个Go程序的第一个协程。

3、运行时检测：检查各种类型长度，检查指针操作，检查结构体字段偏移量，检查atomic原子操作，检查CAS操作，检查栈大小是否是2的幂次。

4、初始化runtime.args:对命令行的参数进行处理，参数数量赋给argc int32，参数值复制给argv **byte

5、调度器初始化 runtime.schedinit：全局栈空间内存分配；加载命令行参数到 os.Args；堆内存空间的初始化；加载操作系统环境变量；初始化当前系统线程；垃圾回收器的参数初始化；算法初始化（map，hash）；设置process数量

6、创建主协程：创建一个新的协程，执行runtime.main

7、初始化M：初始化一个M，用来调度主协程

8、主协程执行主函数：执行runtime包中的init方法；启动垃圾收集器，执行用户包依赖的init方法，执行用户主函数 main.main()

## Go是面向对象的语言吗

Go 没有对象，没有继承、没有实现、没有类。而是通过组合匿名字段达到类似的效果。

Go允许OO编程风格，Go的struct可以看作其他语言的class，Go缺乏其他语言的继承结构，Go的接口和其他语言有很大差异。

在其他语言中，往往用class表示一类数据，class的每个实例称作“对象”，Go是使用Struct表示一类数据，Struct每个实例并不是对象，而是此类型的值，Struct也可以定义方法。

Go并没有继承，而是组合。组合中的匿名字段，通过语法糖达成类似继承的效果。

```go
package main

type Main struct{
	People
}

type People struct{	
}

func (p People) walk() {
	fmt.Println("walk")
}

func main(){
	m:=Main()
	m.walk() // m.People.walk()
}
```

Go 的接口，接口可以定义Go中一组行为相似的struct，struct并不是显示的实现接口，而是隐士的实现接口。

## Go 项目包管理方法

之前的使用的是godep、govendor、golide解决。现在使用的Go Modules

Go vender缓存到本地：

```go
go mod vender
go build -mod vendor
```

如何创建Go Moody
