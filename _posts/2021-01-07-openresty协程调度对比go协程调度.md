---
layout: post
title: "Openresty协程调度对比Go协程调度"
date: 2021-01-07
tags:
 - Go
 - Openresty
---

在web编程领域，Openresty与Go均有十分优秀的处理能力，在面对高并发的web编程，两者一般都是首选的技术方案。这两者我也一直使用，而且两者均有协程，现总结下，留个备忘。


## Openresty及其工作流程

**基于Openresty 1.18版本**

将Lua集成到Nginx中，而Nginx，更是高性能HTTP服务器的代表。

Nginx是多进程单线程：一个master进程和多个worker进程，处理请求的是worker进程。

### 启动流程

Openresty是在master进程创建时通过[ngx_http_lua_init_vm](https://sourcegraph.com/github.com/openresty/lua-nginx-module@v0.10.18/-/blob/src/ngx_http_lua_module.c#L826:14)函数初始化lua vm，在fork出work进程时，lua vm便集成到work进程，每个work进程均有一个lua vm。

worker启动起来后，worker进程便开始[循环处理请求](https://sourcegraph.com/github.com/nginx/nginx@release-1.18.0/-/blob/src/os/unix/ngx_process_cycle.c#L728:1)，当有新的请求到来时，只有[申请到ngx_accept_mutex](https://sourcegraph.com/github.com/nginx/nginx@release-1.18.0/-/blob/src/event/ngx_event_accept.c#L319:1)的worker才会处理它（注册listen fd到自己的epoll中）,避免惊群。

Nginx高性能的原因是其异步非阻塞的事件处理机制，即select/poll/epoll/kqueue这样的系统调用。

### 协程调度

假如有这个配置：

	配置项为：
	location ~ ^/api {
		content_by_lua_file test.lua;
	}

而对于每个请求，如请求为：request=/api?age=20

Openresty都会[创建一个协程来处理](https://sourcegraph.com/github.com/openresty/lua-nginx-module@v0.10.18/-/blob/src/ngx_http_lua_contentby.c#L54:10)。

而这个创建的协程是系统协程，是主协程，用户无法控制它。
而用户通过ngx.thread.spawn创建的协程是通过[ngx_http_lua_coroutine_create_helper](https://sourcegraph.com/github.com/openresty/lua-nginx-module@v0.10.18/-/blob/src/ngx_http_lua_uthread.c#L69:5)创建出来的，用户创建的协程是主协程的子协程。并通过[ngx_http_lua_co_ctx_s](https://sourcegraph.com/github.com/openresty/lua-nginx-module@v0.10.18/-/blob/src/ngx_http_lua_common.h#L442:8)保存协程的相关信息。

协程通过[ngx_http_lua_run_thread](https://sourcegraph.com/github.com/openresty/lua-nginx-module@v0.10.18/-/blob/src/ngx_http_lua_util.c#L1089:1)函数来运行与调度。当前待执行的协程为[ngx_http_lua_ctx_t->cur_co_ctx](https://sourcegraph.com/github.com/openresty/lua-nginx-module@v0.10.18/-/blob/src/ngx_http_lua_common.h#L521:30)。

对于每个worker进程来说，每个用户请求都会创建一个协程，每个协程都是互相隔离的，而且用户还会创建用户协程，这些协程最终交与当前worker进程中的lua vm进行执行。在同一个时间点，只能有一个协程来执行，这些协程该怎么调度呢？

其实这些协程是基于事件的（利用nginx的事件机制）协作式的调度：

1.对于系统创建的协程来说，当系统事件未触发，对应的就是IO事件未准备好（ET模式，epoll_wait返回的活跃fd一直读或写直到返回EAGIAN）时，当前执行的协程就会让出cpu，让别的协程进行执行；

2.对于用户创建的协程来说，除了上面提到的1外，如果用户代码执行了让出，也会进行让出操作。

## GO及其工作流程

**基于go 1.15版本**

Go是单进程，多线程，多协程。

### 启动流程

对于下面这个简单的go程序：

```go

package main
import "fmt"

func main() {
	fmt.Println("Hello world!")
}

```

我们可以通过gdb跟踪到启动流程。

Go程序启动后，在执行用户的main函数前，启动顺序为：
[runtime.args](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/runtime1.go#L60:6)->[runtime.osinit](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/os_linux.go#L298:6)->[runtime.schedinit](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L534:6)->[runtime.newproc](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L3550:6)->[runtime.mstart](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L1116:6)

其中：

	runtime.args:初始化argc,argv；遍历auxv,即辅助向量(auxiliary vector)来初始化一些系统运行变量：如内存页大小（physPageSize），startupRandomData（初始化随机数种子时会用到）,cpuid信息等
	
	runtime.osinit:设置cpu核数（ncpu）和huge page大页内存大小（physHugePageSize）

	runtime.schedinit:
		初始化栈、内存分配器、随机数种子;
		初始化m0并放入allm中；gc初始化;
		对所有p初始化，对allp[0]和m0进行绑定
	
	runtime.newproc:
		这个函数就是我们在go语言中使用go func()来创建协程时，go会调用它来实际创建goroutine.
		会优先在本地p中获取空闲g(Gdead状态)，
		如果本地p中没有，会获取全局空闲的g(schedt.gFree),
		仍没有就会在堆上创建一个初始栈为2k大小的g,
		初始化时是后者，直接在堆上创建一个g用来执行runtime.main.
	
	runtime.mstart:
		启动m,并进行g的调度(循环调度)

上面介绍了每步函数的解释，细节要复杂的多，上面函数中介绍了m0和所有p的初始化，至于g0，其实会在[入口用汇编](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/asm_amd64.s#L99:2)在栈上初始化的。启动时其栈大小约为64k(65432字节)。m0和g0的相互引用也用是[在这时确立](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/asm_amd64.s#L201:2)的，至此，m0,g0,allp[0]的关系确立。


### 调度模型

总览：

![](https://www.imflybird.cn/static/img/2020/GO/P_M_Q_overview.png)

其中：

G：g结构体对象，代表Goroutine。每个G代表一个待执行任务。

M：m结构体对象，代表工作线程（每个工作线程都有一个m与之对应）

P：p结构体对象，代表处理器（Processor）。
> 程序启动后会创建跟cpu核数相等的P（也可以自己更改，一般不修改）。每个P都持有一个待运行G的环形队列（即本地运行队列）。

M-P-G调度是在用户态完成。其中M和G的关系是多对多（M:N）。即M个线程负责对N个G进行调度，内核对M个线程进行调度。



### goroutine调度

循环调度，抢占调度（go 1.14开始实现了基于信号的抢占式调度）。

[schedule()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2609:6)->[execute()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2154:6)->[gogo()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/asm_amd64.s#L272:14)->g.sched.pc()->[goexit()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/asm_amd64.s#L1373:14)->[goexit1](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2942:6)->[goexit0()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2953:6)->[schedule()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L3010:2)

其中：

1.schedule()函数主要为了找寻一个可执行的g：

> 1.每经过61轮调度则从[全局运行队列中](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2673:9)获取g进行执行

> 2.在[本地运行队列](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2678:21)中获取g进行执行

> 3.如果上面两步都没有找到则会[一直找(阻塞)](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L2683:21)，直到找到一个可执行的g.

> 这个阶段会在尝试本地运行队列、全局运行队列、netpoll、窃取其他p的运行队列找到一个可执行的g

2.execute()函数主要设置当前线程的curg，关联当前待执行的g的m为当前线程，更改g的状态从_Grunnable为_Grunning


3.gogo()函数是汇编语言编写：

> 切换到当前的g（切换栈g0->g，恢复g.sched结构体中保存的寄存器的值到cpu寄存器中）

> 让cpu真正执行当前g（执行入口函数为g.sched.pc,即pc寄存器，下一条指令待执行的入口地址）。


4.g.sched.pc()，对于我们这个程序，就是main goroutine,入口函数为[runtime.main](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L114:6)：

> 1.启动一个线程执行[sysmon函数](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L4642:6)，负责整个程序的netpoll监控，gc，抢占调度（对陷入阻塞系统调用的g释放p，对长时间运行(>10ms)的g进行抢占）等。该线程独立运行（无需p，循环运行）

> 2.[runtime包初始化](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L151:2)
> 
> 3.[启动gc](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L167:2)
> 
> 4.[main包初始化](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L191:2)
> import的包也会在这个阶段初始化

5.执行[main.main函数](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L204:2)(我们定义的main函数)

6.从main.main函数返回后，执行系统调用[退出进程](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/proc.go#L226:2)

主goroutine结束后，我们的程序就结束了，这也就是为啥我们在main函数中启动了一个goroutine，如果没有做chan对协程进行数据接收，没看到协程执行结果的原因。

对于非main goroutine，执行完fn（即g.sched.pc）后:

goexit函数：执行runtime.goexit1函数

goexit1函数：mcall切换到g0栈执行runtime.goexit0函数

goexit0函数：g放入gFree队列重用，进行下一轮循环调度。


### 网络IO

同样实现了epoll/kqueue等系统调用，底层使用了汇编实现。如epoll相关函数：

* [epollcreate](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/sys_linux_amd64.s#L689:14)
* [epollctl](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/sys_linux_amd64.s#L704:9)
* [epollwait](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/sys_linux_amd64.s#L716:14)

我们用netpoller来称呼它,它把goroutine和io多路复用结合起来。
通过[netpoll()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/netpoll_epoll.go#L106:6)就可以获取fd活跃的goroutine列表。


### 一些go的冷知识：

1.m0是做什么的？m0和别的m有什么区别？

> 1.如字面所见，m0是第一个被创建的线程。
> 
> 2.m0的作用跟别的m一样，都是系统线程，cpu分配时间片执行任务的线程。
> 
> 3.m上限是10000个，g只有和m绑定后才能真正执行。
> 

2.g0到底是做什么的，g0和别的g有什么区别，g0是否也会被调度？

> 1.如字面所见，g0是第一个被创建的g，但它不是普通的g，并不会被调度.
> 
> 2.g0的作用就是提供一个栈供runtime代码执行，典型的就是[mcall()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/asm_amd64.s#L293:14)和[systemstack()](https://sourcegraph.com/github.com/golang/go@dev.boringcrypto.go1.15/-/blob/src/runtime/asm_amd64.s#L333:14)这两个函数，都是切换到g0栈执行函数，不同的是：前者只能由非g0发起切换到g0栈执行函数，并且不会跳转回来；而后者可以在g或g0发起切换。如果当前已经在g0栈则直接执行，否则会切换到g0栈执行函数，在函数执行完后切回到现在正在执行的代码继续执行后续代码。
> 
> 3.每个m都有一个g0。
> 
> 4.g0跟别的g不一样。首先初始化的栈大小不一样，普通g初始化栈2k大小，g0初始化栈大小有两种情况：将近64k大小和8k大小。在新的m建立的时候，非cgo情况下对应的g0会分配8k大小的栈。
> 
> 5.栈的位置不同。普通的g会在堆上分配栈空间，而g0会在系统栈上分配。

4.对于go程序，启动后会创建多少个线程？

> 各个平台不一样；对于windows平台，会在osinit阶段就提前创建好一些线程，对于linux平台，在执行到runtime.main前，只有一个线程，后面创建线程场景：
> 
> 1.在创建goroutine时会根据需要创建线程。
> 
> 2.runtime阶段创建线程，如启动sysmon系统监控线程，cgo调用启动线程startTemplateThread等。
> 
> 3.cgo执行时，多个cgo同时执行，每个都会需要一个线程。
> 
> 4.在调度go协程时，p找不到需要空闲的m进行执行时。典型场景如web开发中，goroutine执行了阻塞的syscall调用，还有新到的go协程需要处理时。

> 最多创建10000个

5.p,m,g在程序运行过程中会改变吗？

> p确定后就不会改变了，为cpu核心数量(除非人为调整)，存入allp中。
> 
> g和m会增加，但不会减少。g无上限，跟内存有关。空闲的g会放入gFree里（p的gFree为本地空闲队列；schedt的gFree为全局空闲队列）；
> m最多10000个，存入allm中。

6.在做高性能web开发时，需要协程池吗？

> 在面对高并发时，如果不限制g的数量，每个请求一个g的话，则本地p满了后（每个p中存256个），就会放入全局队列中，大量的g会增加gc扫描压力，同时会占用大量内存，大量全局p会有锁访问。
> 
> 所以有必要限制g的数量。而我们其实无法对goroutine进行控制的，而go调度器会自己复用gFree里的goroutine。
> 
> 所以协程池更准确的叫法应该是消费池（请求如同生产，我们处理请求如同消费）。
> 所以我们要做的就是
> 
> 1.尽可能的减少堆内存分配及对内存复用（pool），
> 
> 2.避免阻塞系统调用，
> 
> 3.优化下游及算法响应时间，
> 
> 4.做好限流,限制g的数量，
> 
> 如果做了上面这些后，仍然都是有效访问，且压力很大，那就增加机器吧。


## 对比

1.Openresty启动后，每个cpu核心绑定一个进程，而对于Go来说，每个工作线程对应一个cpu核心，有异曲同工之妙。

2.可以看出,go的调度模型要复杂的多。

Openresty是基于nginx事件的协作式调度

Go实现了一套高效的P-M-G调度（基于信号的抢占式调度（1.14开始））

3.对于网络成面，底层都使用多路IO复用提升web性能。

4.在作为高性能web服务器时，都应避免阻塞的系统调用：
如果涉及到耗时很长的阻塞系统调用，
对于Openresty来说，当前协程一直占用cpu，导致进程直接被阻塞，导致处理性能大幅下降；

对于Go来说，当前goroutine陷入阻塞系统调用后，虽然p会被释，但是工作线程同样会陷入，对于别的要处理的goroutine，发现没有空闲工作线程，就会持续创建工作线程，大量的线程会大幅增加上下文切换，导致性能下降。



