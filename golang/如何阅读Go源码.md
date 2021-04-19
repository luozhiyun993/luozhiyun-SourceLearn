# 如何编译调试Go runtime源码

有朋友问我阅读源码，该怎么调试？这次我们简单看看如何编译调试 Go 的 runtime 源码，感兴趣的朋友可以自己手动操作一下。

## 编译修改 Go 源码进行调试

### 初次下载编译

我使用的是 centos 环境，所以需要先安装一下 `yum -y install gcc`；

然后下载 go 源码：

```sh
[root@localhost src]# git clone https://github.com/golang/go.git

#进入到src 目录执行
[root@localhost src]# ./all.bash

[root@localhost src]# ./all.bash
Building Go cmd/dist using /usr/local/go. (go1.15.8 linux/amd64)
Building Go toolchain1 using /usr/local/go.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for linux/amd64.
...

##### API check
Go version is "go1.15.10", ignoring -next /data/learn/go/api/next.txt

ALL TESTS PASSED
---
Installed Go for linux/amd64 in /data/learn/go
Installed commands in /data/learn/go/bin
*** You need to add /data/learn/go/bin to your PATH.
```

编译好的 go 和 gofmt 在 bin 目录下：

```sh
[root@localhost src]# cd ../bin/
[root@localhost bin]# ls
go  gofmt
```

为了防止我们修改的 go 和过去安装的 go 冲突，创建 mygo 软连接，指向修改的 go ：

```sh
[root@localhost bin]# mkdir -p ~/mygo/bin

[root@localhost bin]# cd ~/testgo/bin

[root@localhost bin]# ln -sf /data/learn/go/bin/go mygo
```

最后，把`~/testgo/bin`加入到`PATH`：

```sh
[root@localhost bin]# vim /etc/profile

export PATH=$PATH:/data/learn/mygo/bin

source /etc/profile
```

运行下mygo，查看一下版本：

```sh
[root@localhost bin]# mygo version
go version go1.15.10 linux/amd64
```

### GODEBUG

我们在修改源码的时候，可以借助 GODEBUG 变量来打印调试信息。

#### schedtrace

> schedtrace: setting schedtrace=X causes the scheduler to emit a single line to standard
> error every X milliseconds, summarizing the scheduler state.

schedtrace=X  表示运行时每 X 毫秒打印一行调度器的摘要信息到标准 err 输出中。

如设置 schedtrace=1000 程序开始后每个一秒就会打印一行调度器的概要信息 ：

```sh
[root@localhost gotest]# GOMAXPROCS=1 GODEBUG=schedtrace=1000 mygo run main.go SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=4 spinningthreads=0 idlethreads=0 runqueue=0 [2]
# command-line-arguments
SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=3 spinningthreads=0 idlethreads=1 runqueue=0 [2]
SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=3 spinningthreads=0 idlethreads=0 runqueue=0 [2]
```

0ms ：自从程序开始的毫秒数；

gomaxprocs=1：配置的处理器数；

idleprocs=0：空闲的 P（处理器）数；

threads=3： 运行期管理的线程数，目前6个线程；

spinningthreads=0：执行抢占的线程数；

idlethreads=1：空闲的线程数；

runqueue=0： 在全局的run队列中的 goroutine 数；

[2]： 本地run队列中的goroutine数，表示有两个在等待；

#### scheddetail 

> scheddetail: setting schedtrace=X and scheddetail=1 causes the scheduler to emit
> detailed multiline info every X milliseconds, describing state of the scheduler,
> processors, threads and goroutines.

schedtrace 和 scheddetail 一起设置可以提供处理器P,线程M和goroutine G的细节。

例如：

```sh
[root@localhost gotest]# GOMAXPROCS=1 GODEBUG=schedtrace=1000,scheddetail=1 mygo run main.go 
SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=4 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
  P0: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=2 gfreecnt=0 timerslen=0
  M3: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=-1
  M2: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=2 dying=0 spinning=false blocked=false lockedg=-1
  M1: p=-1 curg=17 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 spinning=false blocked=false lockedg=17
  M0: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 spinning=false blocked=false lockedg=1
  G1: status=1(chan receive) m=-1 lockedm=0
  G17: status=6() m=1 lockedm=1
  G2: status=1() m=-1 lockedm=-1
  G3: status=1() m=-1 lockedm=-1
  G4: status=4(GC scavenge wait) m=-1 lockedm=-1
...
```

下面我们先看看 G 代表的意思：

* status：G 的运行状态；

* m：隶属哪一个 M；

* lockedm：是否有锁定 M；

G 的运行状态大概有这些含义：

```go
const (
	//  刚刚被分配并且还没有被初始化
	_Gidle = iota // 0 
	// 没有执行代码，没有栈的所有权，存储在运行队列中
	_Grunnable // 1 
	// 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P
	_Grunning // 2 
	// 正在执行系统调用，拥有栈的所有权，没有执行用户代码，
	// 被赋予了内核线程 M 但是不在运行队列上
	_Gsyscall // 3 
	// 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，
	// 但是可能存在于 Channel 的等待队列上
	_Gwaiting // 4  
	// 表示当前goroutine没有被使用，没有执行代码，可能有分配的栈
	_Gdead // 6  
	// 栈正在被拷贝，没有执行代码，不在运行队列上
	_Gcopystack // 8 
	// 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒
	_Gpreempted // 9 
	// GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在
	_Gscan          = 0x1000 
	...
)
```

如果你看不懂的话，那么需要去看看我的调度循环的解析了：《详解Go语言调度循环源码实现 https://www.luozhiyun.com/archives/448》

M 代表的意思：

```
M0: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 spinning=false blocked=false lockedg=1
```

- p：隶属哪一个 P；
- curg：当前正在使用哪个 G；
- mallocing：是否正在分配内存；
- throwing：是否抛出异常；
- preemptoff：不等于空字符串（""）的话，保持 curg 在这个 m 上运行；
- runqsize：运行队列中的 G 数量；
- spinning：是否在抢占 G；

P 代表的意思：

```
P0: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=2 gfreecnt=0 timerslen=0
```

- status：P 的运行状态。
- schedtick：P 的调度次数。
- syscalltick：P 的系统调用次数。
- m：隶属哪一个 M。
- runqsize：运行队列中的 G 数量。
- gfreecnt：可用的G（状态为 Gdead）。

P 的 status 状态代表的含义：

```go
const ( 
	// 表示P没有运行用户代码或者调度器 
	_Pidle = iota 
	// 被线程 M 持有，并且正在执行用户代码或者调度器
	_Prunning 
	// 没有执行用户代码，当前线程陷入系统调用
	_Psyscall
	// 被线程 M 持有，当前处理器由于垃圾回收 STW 被停止
	_Pgcstop 
	// 当前处理器已经不被使用
	_Pdead
)
```

### 修改编译

比方说我们在 channel 这里做一下修改加上一个 print 打印：

```go
func makechan(t *chantype, size int) *hchan {
	...  
	if debug.schedtrace > 0 {
		print("bearluo makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	...
	return c
}
```

然后进入到 go 的 src 目录下重新编译：

```sh
[root@localhost src]# ./make.bash
Building Go cmd/dist using /usr/local/go. (go1.15.8 linux/amd64)
Building Go toolchain1 using /usr/local/go.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for linux/amd64.
---
Installed Go for linux/amd64 in /data/learn/go
Installed commands in /data/learn/go/bin
```

编写一个简单的demo（不能更简单）：

```go
package main

import (
	"fmt"
)

func main() {

	c := make(chan int, 10)
	fmt.Println(c)
}
```

执行：

```sh
[root@localhost gotest]# GODEBUG=schedtrace=1000 mygo run main.go 
bearluo makechan: chan=0xc000036070; elemsize=8; dataqsiz=2
SCHED 0ms: gomaxprocs=16 idleprocs=13 threads=6 spinningthreads=1 idlethreads=0 runqueue=0 [1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]
bearluo makechan: chan=0xc00010e000; elemsize=1; dataqsiz=0
bearluo makechan: chan=0xc00010e060; elemsize=0; dataqsiz=0
bearluo makechan: chan=0xc00010e180; elemsize=0; dataqsiz=0
bearluo makechan: chan=0xc0006a8000; elemsize=1; dataqsiz=35
bearluo makechan: chan=0xc0003f6660; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc000226540; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc0001381e0; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc0005043c0; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc00049c420; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc000594300; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc000090360; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc000220000; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc00075e000; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc000138840; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc000226780; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc0003ea420; elemsize=16; dataqsiz=2
bearluo makechan: chan=0xc00049d320; elemsize=16; dataqsiz=1
...
```

## Delve 调试

目前Go语言支持GDB、LLDB和Delve几种调试器。只有Delve是专门为Go语言设计开发的调试工具。而且Delve本身也是采用Go语言开发，对Windows平台也提供了一样的支持。本节我们基于Delve简单解释如何调试Go runtime代码以及汇编程序。

项目地址：https://github.com/go-delve/delve

安装：

```
go get github.com/go-delve/delve/cmd/dlv
```

首先编写一个test.go的一个例子：

```go
package main

import "fmt"

type A struct {
	test string
}
func main() {
	a := new(A)
	fmt.Println(a)
}
```

然后命令行进入包所在目录，然后输入`dlv debug`命令进入调试：

```powershell
PS C:\document\code\test_go\src> dlv debug
Type 'help' for list of commands.
```

然后可以使用break命令在main包的main方法上设置一个断点：

```powershell
(dlv) break main.main
Breakpoint 1 set at 0x4bd30a for main.main() c:/document/code/test_go/src/test.go:8
```

通过breakpoints查看已经设置的所有断点：

```powershell
(dlv) breakpoints
Breakpoint runtime-fatal-throw at 0x4377e0 for runtime.fatalthrow() c:/software/go/src/runtime/panic.go:1162 (0)
Breakpoint unrecovered-panic at 0x437860 for runtime.fatalpanic() c:/software/go/src/runtime/panic.go:1189 (0)
        print runtime.curg._panic.arg
Breakpoint 1 at 0x4bd30a for main.main() c:/document/code/test_go/src/test.go:8 (0)
```

通过continue命令让程序运行到下一个断点处：

```powershell
(dlv) continue
> main.main() c:/document/code/test_go/src/test.go:8 (hits goroutine(1):1 total:1) (PC: 0x4bd30a)
     3: import "fmt"
     4:
     5: type A struct {
     6:         test string
     7: }
=>   8: func main() {
     9:         a := new(A)
    10:         fmt.Println(a)
    11: }
    12:
    13:
```

通过disassemble反汇编命令查看main函数对应的汇编代码：

```powershell
(dlv) disassemble
TEXT main.main(SB) C:/document/code/test_go/src/test.go
        test.go:8       0x4bd2f0        65488b0c2528000000      mov rcx, qword ptr gs:[0x28]
        test.go:8       0x4bd2f9        488b8900000000          mov rcx, qword ptr [rcx]
        test.go:8       0x4bd300        483b6110                cmp rsp, qword ptr [rcx+0x10]
        test.go:8       0x4bd304        0f8697000000            jbe 0x4bd3a1
=>      test.go:8       0x4bd30a*       4883ec78                sub rsp, 0x78
        test.go:8       0x4bd30e        48896c2470              mov qword ptr [rsp+0x70], rbp
        test.go:8       0x4bd313        488d6c2470              lea rbp, ptr [rsp+0x70]
        test.go:9       0x4bd318        488d0581860100          lea rax, ptr [__image_base__+874912]
        test.go:9       0x4bd31f        48890424                mov qword ptr [rsp], rax
        test.go:9       0x4bd323        e8e800f5ff              call $runtime.newobject
        test.go:9       0x4bd328        488b442408              mov rax, qword ptr [rsp+0x8]
        test.go:9       0x4bd32d        4889442430              mov qword ptr [rsp+0x30], rax
        test.go:10      0x4bd332        4889442440              mov qword ptr [rsp+0x40], rax
        test.go:10      0x4bd337        0f57c0                  xorps xmm0, xmm0
        test.go:10      0x4bd33a        0f11442448              movups xmmword ptr [rsp+0x48], xmm0
        test.go:10      0x4bd33f        488d442448              lea rax, ptr [rsp+0x48]
        test.go:10      0x4bd344        4889442438              mov qword ptr [rsp+0x38], rax
        test.go:10      0x4bd349        8400                    test byte ptr [rax], al
        test.go:10      0x4bd34b        488b4c2440              mov rcx, qword ptr [rsp+0x40]
        test.go:10      0x4bd350        488d15099f0000          lea rdx, ptr [__image_base__+815712]
        test.go:10      0x4bd357        4889542448              mov qword ptr [rsp+0x48], rdx
        test.go:10      0x4bd35c        48894c2450              mov qword ptr [rsp+0x50], rcx
        test.go:10      0x4bd361        8400                    test byte ptr [rax], al
        test.go:10      0x4bd363        eb00                    jmp 0x4bd365
        test.go:10      0x4bd365        4889442458              mov qword ptr [rsp+0x58], rax
        test.go:10      0x4bd36a        48c744246001000000      mov qword ptr [rsp+0x60], 0x1
        test.go:10      0x4bd373        48c744246801000000      mov qword ptr [rsp+0x68], 0x1
        test.go:10      0x4bd37c        48890424                mov qword ptr [rsp], rax
        test.go:10      0x4bd380        48c744240801000000      mov qword ptr [rsp+0x8], 0x1
        test.go:10      0x4bd389        48c744241001000000      mov qword ptr [rsp+0x10], 0x1
        test.go:10      0x4bd392        e869a0ffff              call $fmt.Println
        test.go:11      0x4bd397        488b6c2470              mov rbp, qword ptr [rsp+0x70]
        test.go:11      0x4bd39c        4883c478                add rsp, 0x78
        test.go:11      0x4bd3a0        c3                      ret
        test.go:8       0x4bd3a1        e82a50faff              call $runtime.morestack_noctxt
        .:0             0x4bd3a6        e945ffffff              jmp $main.main
```

现在我们可以使用break断点到runtime.newobject函数的调用上：

```powershell
(dlv) break runtime.newobject
Breakpoint 2 set at 0x40d426 for runtime.newobject() c:/software/go/src/runtime/malloc.go:1164
```

输入continue跳到断点的位置：

```powershell
(dlv) continue
> runtime.newobject() c:/software/go/src/runtime/malloc.go:1164 (hits goroutine(1):1 total:1) (PC: 0x40d426)
Warning: debugging optimized function
  1159: }
  1160:
  1161: // implementation of new builtin
  1162: // compiler (both frontend and SSA backend) knows the signature
  1163: // of this function
=>1164: func newobject(typ *_type) unsafe.Pointer {
  1165:         return mallocgc(typ.size, typ, true)
  1166: }
  1167:
  1168: //go:linkname reflect_unsafe_New reflect.unsafe_New
  1169: func reflect_unsafe_New(typ *_type) unsafe.Pointer {
```

print命令来查看typ的数据：

```powershell
(dlv) print typ
*runtime._type {size: 16, ptrdata: 8, hash: 875453117, tflag: tflagUncommon|tflagExtraStar|tflagNamed (7), align: 8, fieldAlign: 8, kind: 25, equal: runtime.strequal, gcdata: *1, str: 5418, ptrToThis: 37472}
```

可以看到这里打印的size是16bytes，因为我们A结构体里面就一个string类型的field。

进入到mallocgc方法后，通过args和locals命令查看函数的参数和局部变量：

```sh
(dlv) args
size = (unreadable could not find loclist entry at 0x8b40 for address 0x40ca73)
typ = (*runtime._type)(0x4d59a0)
needzero = true
~r3 = (unreadable empty OP stack)
(dlv) locals
(no locals)
```

## Reference

Installing Go from source https://golang.org/doc/install/source

Scheduler Tracing In Go https://www.ardanlabs.com/blog/2015/02/scheduler-tracing-in-go.html

GODEBUG https://golang.org/pkg/runtime/

用 GODEBUG 看调度跟踪 https://eddycjy.com/posts/go/tools/2019-08-19-godebug-sched/

