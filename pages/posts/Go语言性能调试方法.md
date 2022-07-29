---
title: Go语言性能调试方法
date: 2021/9/19
description: Go语言内置性能调试工具和方法测试
tag: Go
author: sh;;;;yd
---

Go 语言内置了性能调优的工具

### 一、准备工作

- 安装 graphviz
  - `brew install graphviz`
- 将 $ GOPATH/bin 加入 $PATH
  - Mac OS：在 .bash_profile 中修改路径
- 安装 go-torch（火炬图）
  - go get github.com/uber/go-torch
  - 下载并复制 flamegraph.pl (https://github.com/brendangregg/FlameGraph) 至 $GOPATH/bin 路径下
  - 将 $GOPATH/bin 加入 $PATH

### 二、性能调优步骤和方法

示例代码：https://github.com/shiiiiyd/csgo/blob/main/tools/file/prof.go

#### 1、build 程序

首先使用 `go build prof.go` 命令构建prof.go 文件，然后运行构建好的prof，使用 `ls` 命令可以查看到当前目录下会多出几个文件，包含：cpu.prof、goroutine.prof、mem.prof文件

#### 2、使用pprof命令查看性能

**查看CPU使用情况：**

```visual basic
csgo/tools/file on  main [!+?] via 🐹 v1.17.12 took 2s 
❯ ls
cpu.prof       goroutine.prof mem.prof       prof           prof.go

csgo/tools/file on  main [!+?] via 🐹 v1.17.12 
❯ go tool pprof prof cpu.prof		## 进入pprof命令
File: prof
Type: cpu
Time: Jul 28, 2022 at 12:27pm (CST)
Duration: 907.52ms, Total samples = 760ms (83.74%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
(pprof) top			## top 命令查看 cpu 性能
Showing nodes accounting for 760ms, 100% of 760ms total
Showing top 10 nodes out of 21
      flat  flat%   sum%        cum   cum%
     720ms 94.74% 94.74%      730ms 96.05%  main.fillMatrix
      20ms  2.63% 97.37%       20ms  2.63%  main.calculate (inline)
      10ms  1.32% 98.68%       10ms  1.32%  math/rand.(*rngSource).Int63
      10ms  1.32%   100%       10ms  1.32%  runtime.(*spanSet).pop
         0     0%   100%      760ms   100%  main.main
         0     0%   100%       10ms  1.32%  math/rand.(*Rand).Int31 (inline)
         0     0%   100%       10ms  1.32%  math/rand.(*Rand).Int31n
         0     0%   100%       10ms  1.32%  math/rand.(*Rand).Int63 (inline)
         0     0%   100%       10ms  1.32%  math/rand.(*Rand).Intn
         0     0%   100%       10ms  1.32%  os.Create (inline)
(pprof)
(pprof)
(pprof) list fillMatrix     ## 查看fillMatrix具体耗费时间的代码片段
Total: 760ms
ROUTINE ======================== main.fillMatrix in /Users/shiyd/go/src/github.com/csgo/tools/file/prof.go
     720ms      730ms (flat, cum) 96.05% of Total
         .          .     19:func fillMatrix(m *[row][col]int) {
         .          .     20:   s := rand.New(rand.NewSource(time.Now().UnixNano()))
         .          .     21:
         .          .     22:   for i := 0; i < row; i++ {
         .          .     23:           for j := 0; j < col; j++ {
     720ms      730ms     24:                   m[i][j] = s.Intn(100000)
         .          .     25:           }
         .          .     26:   }
         .          .     27:}
         .          .     28:
         .          .     29:func calculate(m *[row][col]int) {
(pprof) 
(pprof)
```

**生成SVG图**

```visual basic
## svg 命令是显示 graphviz 输出的svg文件，该文件是可以查看代码耗费的时间
(pprof) svg		
Generating report in profile001.svg
(pprof)exit
```



**查看内存占用（mem）情况：**

```visual basic
csgo/tools/file on  main [!+?] via 🐹 v1.17.12 
❯ go tool pprof prof mem.prof 
File: prof
Type: inuse_space
Time: Jul 28, 2022 at 12:27pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
(pprof) top
Showing nodes accounting for 3233.64kB, 100% of 3233.64kB total
Showing top 10 nodes out of 19
      flat  flat%   sum%        cum   cum%
 1184.27kB 36.62% 36.62%  1184.27kB 36.62%  runtime/pprof.StartCPUProfile
 1025.12kB 31.70% 68.33%  1025.12kB 31.70%  runtime.allocm
  512.20kB 15.84% 84.17%   512.20kB 15.84%  runtime.malg
  512.04kB 15.83%   100%   512.04kB 15.83%  runtime.bgscavenge
         0     0%   100%  1184.27kB 36.62%  main.main
         0     0%   100%  1184.27kB 36.62%  runtime.main
         0     0%   100%   512.56kB 15.85%  runtime.mcall
         0     0%   100%   512.56kB 15.85%  runtime.mstart
         0     0%   100%   512.56kB 15.85%  runtime.mstart0
         0     0%   100%   512.56kB 15.85%  runtime.mstart1
(pprof) 
(pprof) list main.main
Total: 3.16MB
ROUTINE ======================== main.main in /Users/shiyd/go/src/github.com/csgo/tools/file/prof.go
         0     1.16MB (flat, cum) 36.62% of Total
         .          .     41:   if err != nil {
         .          .     42:           log.Fatal("could not create CPU profile: ", err)
         .          .     43:   }
         .          .     44:
         .          .     45:   // 获取系统信息
         .     1.16MB     46:   if err := pprof.StartCPUProfile(f); err != nil { //监控cpu
         .          .     47:           log.Fatal("could not start CPU profile: ", err)
         .          .     48:   }
         .          .     49:   defer pprof.StopCPUProfile()
         .          .     50:
         .          .     51:   // 主逻辑区，进行一些简单的代码运算
(pprof) 
(pprof) 
```



#### 3、使用 flame graphs 查看

**go-torch 命令**

使用`go-torch cpu.prof` 命令生成火焰图

```bash
❯ go-torch cpu.prof
INFO[10:03:05] Run pprof command: go tool pprof -raw -seconds 30 cpu.prof
INFO[10:03:05] Writing svg to torch.svg
```



### 三、通过HTTP方式输出Profile

上面的方式适用于非运行时的服务，如果是运行时的服务则可以通过HTTP访问的方式输出Profile 文件查看。

- 简单，适用于持续性运行的应用

- 在应用程序中导入 import _ "net/http/pprof"，并启动 http server 即可

  - http调用：`import _ "net/http/pprof"`
  - Gin 调用：`import "github.com/DeanThompson/ginpprof" ` `r := gin.Default() ginpprof.Wrap(r)`
  - 示例：go-torch -u http://localhost:8080/ -t 30

- 测试命令：

  - `http://<host>:<port>/debug/pprof/`

    - 示例：http://127.0.0.1:8081/debug/pprof

      又可能不会出现包含block、mutex、heap的界面，需要重启一下服务

  - `go tool pprof http://<host>:<port>/debug/pprof/profile?seconds=10` (默认为30秒)

    - 示例：`go tool pprof http://127.0.0.1:8080/debug/pprof/profile -t 10`

  - `go-torch -seconds 10 http://<host>:<port>/debug/pprof/profile`



### 参考

[1]：https://pkg.go.dev/github.com/uber/go-torch@v0.0.0-20181107071353-86f327cc820e#section-readme

[2]：https://time.geekbang.org/course/detail/100024001-91253

[3]：https://ruanyifeng.com/blog/2017/09/flame-graph.html

[4]：https://blog.csdn.net/qq_17612199/article/details/80353355
