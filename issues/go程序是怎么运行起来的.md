## Go程序启动过程
本文会从源码来分析Go程序是如何启动，并会介绍两个如何调试Go代码的工具，能让你清晰的看到整个过程

## 测试代码
```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("hello world")
}
```

## gdb调试
### gdb安装
安装用homebrew就可以
```
brew install gdb
```

### 文件编译
```
go build -gcflags "-N -l" -ldflags=-compressdwarf=false -o filename main.go
```

### gdb调试
mac gdb调试还是遇到了两个问题
* r或者run之后卡住了，无法继续执行，只能kill -9杀掉进程
* 报错no debugging symbols found

如果遇到这两个问题，可以参考下边链接来解决
* https://github.com/Homebrew/homebrew-core/issues/49631#issue-558007139
* https://www.cnblogs.com/zhuxiaoxi/p/10095097.html

## 程序真正的入口
### 打断点
安装编译完成之后，执行
```
sudo gdb filename
```
就能进入到gdb的环境中，这样就表示成功了
![20240404161815](https://cdn.jsdelivr.net/gh/yaojie91/pic@main/blogpic/20240404161815.png)

然后可以执行
```
info file
```
可以看到
```
(gdb) info file
	...
	Entry point: 0x105bbc0
	0x0000000001001000 - 0x0000000001089830 is .text
	0x0000000001089840 - 0x0000000001089942 is __TEXT.__symbol_stub1
	0x0000000001089960 - 0x00000000010c166d is __TEXT.__rodata
	0x00000000010c1680 - 0x00000000010c1ba0 is __TEXT.__typelink
	0x00000000010c1ba0 - 0x00000000010c1c10 is __TEXT.__itablink
	0x00000000010c1c10 - 0x00000000010c1c10 is __TEXT.__gosymtab
	0x00000000010c1c20 - 0x000000000111b0e8 is __TEXT.__gopclntab
	0x000000000111c000 - 0x000000000111c1e0 is __DATA.__go_buildinfo
	0x000000000111c1e0 - 0x000000000111c338 is __DATA.__nl_symbol_ptr
	0x000000000111c340 - 0x000000000112c980 is __DATA.__noptrdata
	0x000000000112c980 - 0x0000000001133e10 is .data
	0x0000000001133e20 - 0x0000000001162e80 is .bss
	0x0000000001162e80 - 0x0000000001167e20 is __DATA.__noptrbss
```
上边这些可以看到entry point的地址是0x105bbc0，然后执行
```
(gdb) b *0x105bbc0  // b即break 打断点
```
可以看到在`rt0_darwin_amd64.s`第8行打了断点，可知这个位置就是程序执行的起点，这是mac的，linux也有自己的文件，rt0指runtime0，意思是最开始的运行时
```
(gdb) b *0x105bbc0
Breakpoint 1 at 0x105bbc0: file /usr/local/go/src/runtime/rt0_darwin_amd64.s, line 8.
```
然后输入`r`或者`run`开始执行，会在这个断点位置停下
```
Thread 2 hit Breakpoint 1, _rt0_amd64_darwin () at /usr/local/go/src/runtime/rt0_darwin_amd64.s:8
8		JMP	_rt0_amd64(SB)
```
这里通过JMP跳到了`_rt0_amd64`函数，对这个函数打断点
```
(gdb) b _rt0_amd64
Breakpoint 7 at 0x10581c0: file /usr/local/go/src/runtime/asm_amd64.s, line 16.
```
输入`c`即`continue`，会执行到这个断点
```
Thread 2 hit Breakpoint 7, _rt0_amd64 () at /usr/local/go/src/runtime/asm_amd64.s:16
16		MOVQ	0(SP), DI	// argc
```
通过`list`我们看到代码会跳转到`runtime/asm_amd64.s runtime·rt0_go(SB)`
```
// 该方法将程序输入的 argc 和 argv 从内存移动到寄存器中。
// 栈指针（SP）的前两个值分别是 argc 和 argv，其对应参数的数量和具体各参数的值。
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB) // 跳转到rt0_go
```

然后看`rt0_go`的代码
```
// 绑定m0和g0，源码里m0和g0只看到了定义，应该是在编译过程中赋值的
// save m->g0 = g0
MOVQ	CX, m_g0(AX)
// save m0 to g0->m
MOVQ	AX, g_m(CX)
```

这部分是初始化相关的操作

```
CALL	runtime·check(SB)

MOVL	16(SP), AX		// copy argc
MOVL	AX, 0(SP)
MOVQ	24(SP), AX		// copy argv
MOVQ	AX, 8(SP)
// 该函数在runtime/runtime1.go/args(c int32, v **byte) 进行命令行参数的初始化
CALL	runtime·args(SB)
// 获取cpu个数和内存页大小初始化
CALL	runtime·osinit(SB)
// 命令行参数、环境变量、gc、栈空间、内存管理、所有P实例、HASH算法等初始化
CALL	runtime·schedinit(SB)  
```

下边这快是真正执行代码的入口
```
// create a new goroutine to start program    
// mainPC其实通过赋值，实际执行的是proc.go的main, DATA指令就是赋值
// DATA	runtime·mainPC+0(SB)/8,$runtime·main(SB)
MOVQ	$runtime·mainPC(SB), AX		// entry
PUSHQ	AX                          // 压栈main方法 newproc的第二个函数
PUSHQ	$0			// arg size     // 压栈参数是0 newproc的第一个函数
CALL	runtime·newproc(SB)
POPQ	AX                          // 出栈
POPQ	AX                          // 出栈
```

### runtime.main

```
func main() {
	// 获取当前g
	g := getg()
	
	...
	
	// 切换到g0栈，在系统栈上运行 sysmon
	systemstack(func() {
		// 分配一个新的m，运行sysmon系统后台监控
		// （定期垃圾回收和调度抢占）
		// 后边会解释哪里用到了
		newm(sysmon, nil)
	})
	...
	
	// 确保是主线程
	if g.m != &m0 {
		throw("runtime.main not on m0")
	}
	
	// runtime 内部 init 函数的执行，编译器动态生成的。
	runtime_init() // must be before defer
	
	...
	
	// gc 启动一个goroutine进行gc清扫
	gcenable()
	
	...
	
	// 真正的执行main func in package main
	// main_main函数是一个linkname指向了main包的编译的main
	fn = main_main 
	fn()
	
	...
	
	// 退出程序
	exit(0)
	
	// 为何这里还需要for循环？
	// 下面的for循环一定会导致程序崩掉，这样就确保了程序一定会退出
	for {
		var x *int32
		*x = 0
	}
}
```

上边介绍的就是runtime的main方法，其实从上边`rt0_go`的代码可以看到是执行了newproc函数，runtime的main方法是作为newproc的参数

```
func newproc(siz int32, fn *funcval) {   // fn即runtime的main
	...
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
	// systemstack会切换到g0的栈
	systemstack(func() {
		newproc1(fn, argp, siz, gp, pc)
	})
}
```

### newproc1

	func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) {
		// 获取当前g，即g0
		_g_ := getg()
		...
		
		// 获取g0对应的p的指针
		_p_ := _g_.m.p.ptr()
		// 从p中获取一个g
		newg := gfget(_p_)
		if newg == nil {
			// 获取不到就new一个g
			newg = malg(_StackMin)
			casgstatus(newg, _Gidle, _Gdead)
			allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
		}
		...
		// 把g放到队列
		runqput(_p_, newg, true)
	}

至此我们main包的main函数已经作为g被放到待调度任务里面了。那么下一步就是开启调度循环能让我们的任务跑起来的。

### runtime.mstart

这个mstart也是封装了mstart1

	func mstart() {
		_g_ := getg()
		// ... 省略cgo相关
		_g_.stackguard0 = _g_.stack.lo + _StackGuard
		_g_.stackguard1 = _g_.stackguard0
		mstart1()
		// ...
	}

然后看下mstart1

    func mstart1() {
        _g_ := getg()
        if _g_ != _g_.m.g0 {
            throw("bad runtime·mstart")
        }

        // 初始化m0
        minit()

        // 针对m0做一些特殊处理，主要是信号相关
        if _g_.m == &m0 {
            mstartm0()
        }
        // 执行挂在m上的函数
      // newm(fn func(), _p_ *p, id int64)的fn参数就是挂在m.mstartfn上的
        if fn := _g_.m.mstartfn; fn != nil {
            fn()
        }

      // 和前面的M0和G0不用P呼应上了。
        if _g_.m != &m0 {
            acquirep(_g_.m.nextp.ptr())
            _g_.m.nextp = 0
        }
        // 启动调度循环
        schedule()
    }

`_g_.m.mstartfn`还记得前面在runtime.main的时候通过newm创建了个M，对应的mstartfn就是sysmon，我们也能看出，其实sysmon也是不用绑定P就能执行。


`schedule`函数会进入到调度循环，首先先尽可能找到可以执行的goroutine，调用execute

        // One round of scheduler: find a runnable goroutine and execute it.
        // Never returns.
        func schedule() {
        	_g_ := getg()
        	
        	...
        	// STW 方便gc
        	if sched.gcwaiting != 0 {
        		gcstopm()
        		goto top
        	}
        	...
        	// 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 schedtick 保证有一定几率会从全局的运行队列中查找对应的 Goroutine；
        	if gp == nil {
        		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
        			lock(&sched.lock)
        			gp = globrunqget(_g_.m.p.ptr(), 1)
        			unlock(&sched.lock)
        		}
        	}
        	// 从处理器本地的运行队列中查找待执行的 Goroutine；
        	if gp == nil {
        		gp, inheritTime = runqget(_g_.m.p.ptr())
        		// We can see gp != nil here even if the M is spinning,
        		// if checkTimers added a local goroutine via goready.
        	}
        	// 如果前两种方法都没有找到 Goroutine，会通过 runtime.findrunnable 进行阻塞地查找 Goroutine；
        	if gp == nil {
        		gp, inheritTime = findrunnable() // blocks until work is available
        	}
        }
        ...
        execute()

}

`findrunnable`实现非常复杂，这个函数通过以下的过程获取可运行的 Goroutine：

*   从本地运行队列、全局运行队列中查找；
*   从网络轮询器中查找是否有 Goroutine 等待运行；
*   通过 runtime.runqsteal 尝试从其他随机的处理器中窃取待运行的 Goroutine，该函数还可能窃取处理器的计时器；

因为函数的实现过于复杂，上述的执行过程是经过简化的，总而言之，当前函数一定会返回一个可执行的 Goroutine，如果当前不存在就会阻塞等待。

`execute`中先对g绑定m，然后它会通过`runtime.gogo`将 Goroutine 调度到当前线程上。

    func execute(gp *g, inheritTime bool) {
    	_g_ := getg()

    	_g_.m.curg = gp
    	gp.m = _g_.m  //绑定m
    	
        ...
        // gogo是汇编实现的，在不同的架构有不同的实现
    	gogo(&gp.sched)
    }

gogo的实现最终还会调用到schedule函数，也就是上边写的调度循环开始的地方，所以go的调度是一个不会退出的循环

至此，我们就分析完go启动的过程了，源码里还有很多细节可以去分析，欢迎大家一起讨论。

##### 参考资料

*   [go程序如何跑起来](https://zhuanlan.zhihu.com/p/404696085)
*   [sysmon函数](https://www.linkinstars.com/post/e94a7c60.html)
*   [go调度器](https://blog.csdn.net/qq_41822345/article/details/125880611)
*   [go语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)

