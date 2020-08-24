# Golang GPM调度模型


https://github.com/lifei6671/interview-go/blob/master/base/go-gpm.md

https://wudaijun.com/2018/01/go-scheduler/

https://www.cnblogs.com/williamjie/p/9267741.html

https://www.cnblogs.com/sunsky303/p/9705727.html

## CSP

通讯顺序进程（Communicating Sequential Processes），七大并发模型中的一种

核心理念是通过通道  `channel`  将两个 **并发执行的实体** 连接起来

Go 中  `GPM` 调度模型就是对 CSP 模型的实现



## GPM调度模型角色



|      | 角色      |
| ---- | --------- |
| G    | Goroutine |
| P    | Processor |
| M    | Machine   |



### Goroutine

#### 结构体信息

Goroutine 对应结构体 `g` 

- G并不是执行体，需要绑定到 P 上面才可以被调度去执行
- G 对象包括栈、指令指针等信息



```shell
$GOROOT/src/runtime/runtime2.go
```

```go
// line 395
type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic       *_panic // innermost panic - offset known to liblink
	_defer       *_defer // innermost defer
	m            *m      // current m; offset known to arm liblink
	sched        gobuf
	syscallsp    uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc    uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp     uintptr        // expected sp at top of stack, to check in traceback
	param        unsafe.Pointer // passed parameter on wakeup
	atomicstatus uint32
	stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid         int64
	schedlink    guintptr
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting

	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point

	// asyncSafePoint is set if g is stopped at an asynchronous
	// safe point. This means there are frames on the stack
	// without precise pointer information.
	asyncSafePoint bool

	paniconfault bool // panic (instead of crash) on unexpected fault address
	gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
	throwsplit   bool // must not split stack
	// activeStackChans indicates that there are unlocked channels
	// pointing into this goroutine's stack. If true, stack
	// copying needs to acquire channel locks to protect these
	// areas of the stack.
	activeStackChans bool

	raceignore     int8     // ignore race detection events
	sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
	sysexitticks   int64    // cputicks when syscall has returned (for tracing)
	traceseq       uint64   // trace event sequencer
	tracelastp     puintptr // last P emitted an event for this goroutine
	lockedm        muintptr
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
	gopc           uintptr         // pc of go statement that created this goroutine
	ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
	startpc        uintptr         // pc of goroutine function
	racectx        uintptr
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt        []uintptr      // cgo traceback context
	labels         unsafe.Pointer // profiler labels
	timer          *timer         // cached timer for time.Sleep
	selectDone     uint32         // are we participating in a select and did someone win the race?

	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}
```

其中 `sched` 字段最为重要，goroutine 调度相关的所有数据都存储在该字段中，保存了 goroutine 上下文；

```shell
$GOROOT/src/runtime/runtime2.go
```

```go
// line 310
type gobuf struct {
	// The offsets of sp, pc, and g are known to (hard-coded in) libmach.
	//
	// ctxt is unusual with respect to GC: it may be a
	// heap-allocated funcval, so GC needs to track it, but it
	// needs to be set and cleared from assembly, where it's
	// difficult to have write barriers. However, ctxt is really a
	// saved, live register, and we only ever exchange it between
	// the real register and the gobuf. Hence, we treat it as a
	// root during stack scanning, which means assembly that saves
	// and restores it doesn't need write barriers. It's still
	// typed as a pointer so that any other writes from Go get
	// write barriers.
	sp   uintptr
	pc   uintptr
	g    guintptr
	ctxt unsafe.Pointer
	ret  sys.Uintreg
	lr   uintptr
	bp   uintptr // for GOEXPERIMENT=framepointer
}
```





#### 动态扩容

Goroutine 并不是像操作系统线程那样分配固定大小的内存块用于栈，而是采用**动态扩容方式**

- 以本机 `1.14.4` 为例, 可以看到默认栈大小为  `2048`，也就是 2KB
- 根据任务执行的多少栈会不断扩大，但是最大 `1GB`(64bit OS), `256M`(32bit OS)

```shell
$GOROOT/src/runtime/stack.go

// line 72
_StackMin = 2048
_StackLimit = _StackGuard - _StackSystem - _StackSmall
```













### Processor

Logical Processor 逻辑处理器， 相对于 Goroutine 来说，Processor 相当于 CPU核

Goroutine 只有绑定到了 Processor 上才能被调度

对M来说，P提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等，P的数量决定了系统内最大可并行的G的数量（前提：物理CPU核数 >= P的数量），P的数量由用户设置的GOMAXPROCS决定，但是不论GOMAXPROCS设置为多大，P的数量最大为256。



### Machine

操作系统线程抽象，代表真正执行计算的资源

Machine 并不保留 Goroutine 的状态，这也是 Goroutine 可以跨越 Machine调度 的基础

Machine 的数量并不是固定的，通过 `Go Runtime` 进行调整，为了防止过多的操作系统线程，默认最大限制为 10000 个



### 基本调度模型

![](https://gitee.com/AbyssViper/pic/raw/master/images/golang-gpm-model/gpm-schedule.png)


































