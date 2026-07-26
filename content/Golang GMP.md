%% [[Golang]] %%
# GMP设计思想

 - G：goroutine协程
- M：thread线程
- P：processor处理器

## GMP模型概述

在Go中，线程是运行goroutine的实体，调度器的功能是把可运行的goroutine分配到工作线程上。

![[GMP 1.png]]
1. 全局队列（Global Queue）：存放等待运行的G。
2. P的本地队列：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建G'时，G'优先加入到P的本地队列，如果队列满了，则会把本地队列中一半的G移动到全局队列。
3. P列表：所有的P都在程序启动时创建，并保存在数组中，最多有GOMAXPROCS个(可配置)。
4. M：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M也会尝试从全局队列拿一批G放到P的本地队列，或从其他P的本地队列偷一半放到自己P的本地队列。M运行G，G执行之后，M会从P获取下一个G，不断重复下去。

Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线程分配到CPU的核上执行。
### 个数问题

1. G的数量
	理论上没有数量上限限制的。查看当前G的数量可以使用`runtime. NumGoroutine()`
2. P的数量
	由启动时环境变量$GOMAXPROCS或者runtime的方法`GOMAXPROCS()`决定，这意味着在程序执行的任意时刻都只有 GOMAXPROCS个goroutine在同时运行。
3. M的数量
	- 默认10000，这是由go程序启动时，会设置M的最大数量，但是内核很难支持那么多的线程数，所以这个限制可以忽略。
	- 在runtime/debug中的`SetMaxThreads()`函数，设置M的最大数
	- 一个M阻塞了，会创建新的M。

M和P的数量没有绝对关系，一个M阻塞，P就会创建或者切换另一个M，所以，即使P的默认数量是1，也有可能会创建很多个M出来。
### 创建P和M的时机

1. P创建时机
	在确定了P的最大数量n后，运行时系统会根据这个数量创建n个P（那创建好的P在哪
2. M创建时机
	没有足够的M来关联P并运行其中的可运行的G。比如所有的M此时都阻塞住了，而P中还有很多就绪任务，就会去寻找空闲的M，而没有空闲的，就会去创建新的M。
## 调度器的设计策略

**复用线程**：避免频繁的创建、销毁线程，而是对线程的复用。

1. work stealing机制：当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。（这个到底是怎么实现的...?

2. hand off机制：当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行

	利用并行：GOMAXPROCS设置P的数量，最多有GOMAXPROCS个线程分布在多个CPU上同时运行。GOMAXPROCS也限制了并发的程度，比如GOMAXPROCS = 核数/2，则最多利用了一半的CPU核进行并行。

	抢占：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这就是goroutine不同于coroutine的一个地方。

	全局G队列：在新的调度器中依然有全局G队列，但功能已经被弱化了，当M执行work stealing从其他P偷不到G时，它可以从全局G队列获取G。

## `go func()`调度流程

![[GMP 2.png]]

1. `go func()`来创建goroutine
2. 有两个存储G的队列，一个是局部调度器P的本地队列、一个是全局G队列。新创建的G会先保存在P的本地队列中，如果P的本地队列已经满了就会保存在全局的队列中；
3. G只能运行在M中，一个M必须持有一个P，M与P是1：1的关系。M会从P的本地队列弹出一个可执行状态的G来执行，如果P的本地队列为空，就会想其他的MP组合偷取一个可执行的G来执行；
4. 一个M调度G执行的过程是一个循环机制；
5. 当M执行某一个G时候如果发生了syscall或则其余阻塞操作，M会阻塞，如果当前有一些G在执行，runtime会把这个线程M从P中摘除(detach)，然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)来服务于这个P；
6. 当M系统调用结束时候，这个G会尝试获取一个空闲的P执行，并放入到这个P的本地队列。如果获取不到P，那么这个线程M变成休眠状态， 加入到空闲线程中，然后这个G会被放入全局队列中。

## 调度器的生命周期

![[GMP 3.png]]
**特殊的M0和G0**

1. M0
	M0是启动程序后的编号为0的主线程，这个M对应的实例会在全局变量runtime.m0中，不需要在heap上分配，M0负责执行初始化操作和启动第一个G， 在之后M0就和其他的M一样了。
2. G0
	G0是每次启动一个M都会第一个创建的gourtine，**G0仅用于负责调度的G**，G0不指向任何可执行的函数，每个M都会有一个自己的G0。在调度或系统调用时会使用G0的栈空间，全局变量的G0是M0的G0。

```Go
package main

import "fmt"func main() {
    fmt.Println("Hello world")
}
```

1. runtime创建最初的线程m0和goroutine g0，并把2者关联。
2. 调度器初始化：初始化m0、栈、垃圾回收，以及创建和初始化由GOMAXPROCS个P构成的P列表。
3. 示例代码中的main函数是`main.main`，`runtime`中也有1个main函数——`runtime.main`，代码经过编译后，`runtime.main`会调用`main.main`，程序启动时会为`runtime.main`创建goroutine，称它为main goroutine吧，然后把main goroutine加入到P的本地队列。
4. 启动m0，m0已经绑定了P，会从P的本地队列获取G，获取到main goroutine。
5. G拥有栈，M根据G中的栈信息和调度信息设置运行环境
6. M运行G
7. G退出，再次回到M获取可运行的G，这样重复下去，直到`main.main`退出，`runtime.main`执行Defer和Panic处理，或调用`runtime.exit`退出程序。

调度器的生命周期几乎占满了一个Go程序的一生，`runtime.main`的goroutine执行之前都是为调度器做准备工作，`runtime.main`的goroutine运行，才是调度器的真正开始，直到`runtime.main`结束而结束。

#  GMP调度场景全过程分析 
## 1.场景1 G1创建G2

P拥有G1，M1获取P后开始运行G1，G1使用`go func()`创建了G2，为了局部性G2优先加入到P1的本地队列。


![[GMP 场景 1.png]]

## 2.场景2 G1执行完毕

1. G1运行完成后(函数：goexit)
2. M上运行的goroutine切换为G0，G0负责调度时协程的切换（函数：schedule）。
3. 从P的本地队列取G2，从G0切换到G2，并开始运行G2(函数：execute)。

实现了线程M1的复用。

![[GMP 场景 2.png]]

## 3.场景3 G2开辟过多G

假设每个P的本地队列只能存3个G。G2要创建了6个G，前3个G（G3, G4, G5）已经加入p1的本地队列，p1本地队列满了。

![[GMP 场景 3.png]]

## 4.场景4 G2本地满再创建G7

G2在创建G7的时候，发现P1的本地队列已满，需要执行**负载均衡**（把P1中本地队列中前一半的G，还有新创建G**转移**到全局队列）

（实现中并不一定是新的G，如果G是G2之后就执行的，会被保存在本地队列，利用某个老的G替换新G加入全局队列）

![[GMP 场景 4.png]]

这些G被转移到全局队列时，会被打乱顺序。所以G3,G4,G7被转移到全局队列。

## 5.场景5 G2本地未满创建G8

G2创建G8时，P1的本地队列未满，所以G8会被加入到P1的本地队列。

G8加入到P1点本地队列的原因还是因为P1此时在与M1绑定，而G2此时是M1在执行。所以G2创建的新的G会优先放置到自己的M绑定的P上。

![[GMP 场景 5.png]]

## 6.场景6 唤醒正在休眠的M

规定：**在创建G时，运行的G会尝试唤醒其他空闲的P和M组合去执行**。

假定G2唤醒了M2，M2绑定了P2，并运行G0，但P2本地队列没有G，M2此时为自旋线程**（没有G但为运行状态的线程，不断寻找G）。**

![[GMP 场景 6.png]]

> **自旋线程** 在GMP模型中自旋线程通常指的是一个暂时没有找到可运行 Goroutine 的 M，不立刻休眠，而是在 CPU 上继续运行一小段时间，主动寻找可执行任务。

## 7.场景7 被唤醒的M2从全局队列取批量G

M2尝试从全局队列（简称“GQ”）取一批G放到P2的本地队列（函数：findrunnable）。M2从全局队列取的G数量符合下面的公式：

$$
n = min(len(GQ)/GOMAXPROCS + 1, len(GQ)/2)
$$

至少从全局队列取1个g，但每次不要从全局队列移动太多的g到p本地队列，给其他p留点。这是**从全局队列到P本地队列的负载均衡**。​

假定我们场景中一共有4个P（GOMAXPROCS设置为4，那么我们允许最多就能用4个P来供M使用）。

所以M2只从能从全局队列取1个G（即G3）移动P2本地队列，然后完成从G0到G3的切换，运行G3。

![[GMP 场景 7.png]]

## 8.场景8 M2从M1中偷取G

假设G2一直在M1上运行，经过2轮后，M2已经把G7、G4从全局队列获取到了P2的本地队列并完成运行，全局队列和P2的本地队列都空了，如场景8图的左半部分。

​**全局队列已经没有G，那m就要执行work stealing(偷取)：从其他有G的P哪里偷取一半G过来，放到自己的P本地队列**。

P2从P1的本地队列尾部取一半的G，本例中一半则只有1个G8，放到P2的本地队列并执行。

![[GMP 场景 8.png]]

## 9.场景9 自旋线程的最大限制

G1本地队列G5、G6已经被其他M偷走并运行完成，当前M1和M2分别在运行G2和G8，M3和M4没有goroutine可以运行，M3和M4处于**自旋状态**，它们不断寻找goroutine。

为什么要让m3和m4自旋，自旋本质是在运行，线程在运行却没有执行G，就变成了浪费CPU。为什么不销毁现场，来节约CPU资源。

因为创建和销毁CPU也会浪费时间，我们**希望当有新goroutine创建时，立刻能有M运行它**，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费CPU，所以系统中最多有GOMAXPROCS个自旋的线程(当前例子中的GOMAXPROCS=4，所以一共4个P)，多余的没事做线程会让他们休眠。

![[GMP 场景 9.png]]

## 10.场景10 G发生系统调用/阻塞

假定当前除了M3和M4为自旋线程，还有M5和M6为空闲的线程(没有得到P的绑定，注意我们这里最多就只能够存在4个P，所以P的数量应该永远是M>=P, 大部分都是M在抢占需要运行的P)

G8创建了G9，G8进行了**阻塞的系统调用**，M2和P2立即解绑，P2会执行以下判断：如果P2本地队列有G、全局队列有G或有空闲的M，P2都会立马唤醒1个M和它绑定，否则P2则会加入到空闲P列表，等待M来获取可用的p。

本场景中，P2本地队列有G9，可以和其他空闲的线程M5绑定。

![[GMP 场景 10.png]]

## 11.场景11 G发生系统调用/非阻塞

G8创建了G9，假如G8进行了**非阻塞系统调用**。

M2和P2会解绑，但M2会记住P2，然后G8和M2进入**系统调用**状态。当G8和M2退出系统调用时，会尝试获取P2，如果无法获取，则获取空闲的P，如果依然没有，G8会被记为可运行状态，并加入到全局队列，M2因为没有P的绑定而变成休眠状态（长时间休眠等待GC回收销毁）。

先尝试获取P2，再尝试获取空闲P，都没有则G进入全局队列，M进入休眠


![[GMP 场景 11.png]]

# GMP内部结构

## G

G的内部结构重要字段如下，完全结构请参见[源码](https://github.com/golang/go/blob/5622128a77b4af5e5dc02edf53ecac545e3af730/src/runtime/runtime2.go#L387)

```Go
type g struct {	
	// 执行环境 
	stack stack // 栈内存范围 [lo, hi)
	stackguard0 uintptr // 栈增长 + 抢占触发的关键
	sched gobuf // 寄存器快照 (sp/pc/lr/bp)

	// 身份与状态 
	goid uint64
	atomicstatus atomic.Uint32
	
	// 与 M 的绑定 
	m *m // 当前在哪个 M 上执行
	lockedm muintptr // LockOSThread 锁定
	
	// 队列链接
	schedlink guintptr // 运行队列 / 空闲池链表

	// 抢占
	preempt bool // 抢占信号（配合 stackguard0=StackPreempt）

	// 异常处理
	_panic *_panic 
	_defer *_defer 

	// 阻塞信息
	waiting *sudog // 阻塞等待链表
	waitreason waitReason // 阻塞原因
}
```

gobuf结构体，主要在调度器保存或者恢复上下文的时候用到

G 不是 OS 线程，OS 不会帮它保存/恢复寄存器。当调度器决定"换一个 G 执行"时，必须有一个地方存下当前 G 的所有寄存器状态，下次运行时再恢复，gobuf就会承担这个责任：

```Go
type gobuf struct {
    sp   uintptr         // 栈指针 — 恢复后 SP 指向这里
    pc   uintptr         // 程序计数器 — gogo 最后 JMP 到这个地址
    g    guintptr        // 指向自己的 G 指针（用 uintptr 绕过 GC 写屏障）
    ctxt unsafe.Pointer  // 闭包上下文（funcval），汇编和 Go 之间传递
    lr   uintptr         // 链接寄存器（ARM 等架构用，amd64 无视）
    bp   uintptr         // 帧指针 — 用于栈回溯
}
```

核心字段：sp、pc、bp

这里我们关注两个使用场景：

1. G被换下，当用户G正在执行时，时间片到了 / gopark / 系统调用将当前的sp、pc、bp写到sched中
2. G被换上，从 gobuf 恢复 sp/pc/bp

在执行过程中，G可能存在以下状态，以下列举出核心状态：

| 状态          | 值   | 含义                        |
| ----------- | --- | ------------------------- |
| \_Grunnable | 1   | 在队列里排队，等着被调度              |
| \_Grunning  | 2   | 正在某个 M 上执行用户代码            |
| \_Gwaiting  | 4   | 阻塞了（channel、锁、sleep），等待唤醒 |
| \_Gdead     | 6   | 刚退出/刚分配，在 free list 里待复用  |
重点只需要关注这几个：

- 等待中：\_Gwaiting、\_Gsyscall 和 \_Gpreempted，这几个状态表示G没有在执行；
- 可运行：\_Grunnable，表示G已经准备就绪，可以在线程运行;
- 运行中：\_Grunning，表示G正在运行；

状态的本质是，栈的”所有权“

>_"Beyond indicating the general state of a G, the G status acts like a **lock on the goroutine's stack** (and hence its ability to execute user code)."_

# 参考文章

- [G-M-P调度机制](https://go.cyub.vip/gmp/)
- [Golang的协程调度器原理及GMP设计思想](https://github.com/aceld/golang/blob/main/2%E3%80%81Golang%E7%9A%84%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6%E5%99%A8%E5%8E%9F%E7%90%86%E5%8F%8AGMP%E8%AE%BE%E8%AE%A1%E6%80%9D%E6%83%B3%EF%BC%9F.md)
- [Go源码](https://github.com/golang/go)
- [详解Go语言调度循环源码实现](https://www.luozhiyun.com/archives/448)



