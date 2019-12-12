---
layout: "post"
title: "【协程系列】 libco 深入分析"
category: 技术
date: "2018-01-26 12:38"
---

## 简介
libco是微信后台大规模使用的c/c++协程库，2013年至今稳定运行在微信后台的数万台机器上。  
libco通过仅有的几个函数接口 co_create/co_resume/co_yield 再配合 co_poll，可以支持同步或者异  
步的写法，如线程库一样轻松。同时库里面提供了socket族函数的hook，使得后台逻辑服务几乎不用修改逻辑代码就可以完成异步化改造。[1]

## 基本架构

libco 使用共享栈的协程调度库。


## 数据结构

**协程属性上下文**
```cpp
struct stCoRoutineAttr_t
{
	int stack_size;
	stShareStack_t*  share_stack;
	stCoRoutineAttr_t()
	{
		stack_size = 128 * 1024;
		share_stack = NULL;
	}
}__attribute__ ((packed));

// 共享栈
struct stShareStack_t
{
	unsigned int alloc_idx;
	int stack_size;
	int count;
	stStackMem_t** stack_array;
};

//协程运行栈
struct stStackMem_t
{
	stCoRoutine_t* occupy_co;
	int stack_size;
	char* stack_bp; //stack_buffer + stack_size
	char* stack_buffer;
};

```
协程属性用于初始化协程运行栈, stack_size 表示运行栈内存大小， share_stack 表示共享运行栈。  
如果share_stack为 _NULL_, 协程使用独立运行栈， 否则使用共享栈。

共享栈的数据结构是 stShareStack_t, alloc_idx 是单调递增的计数器， 表示该共享栈分配次数，  
stack_size 同样表示每个运行栈的内存大小， count 表示运行栈的个数。

运行栈的数据结构是 stStackMem_t, occupy_co 表示当前使用该栈的协程上下文, stack_size 表示  
栈空间大小， stack_buffer 表示实际的内存起始地址， stack_bp 是栈基址，由于运行栈空间是由高  
地址向低地址增长的， 所以栈基址 stack_bp = stack_buffer + stack_size。

**调度上下文**
```cpp
struct stCoRoutineEnv_t
{
	stCoRoutine_t *pCallStack[ 128 ];
	int iCallStackSize;
	stCoEpoll_t *pEpoll;

	//for copy stack log lastco and nextco
	stCoRoutine_t* pending_co;
	stCoRoutine_t* occupy_co;
};
```
调度上下文的数据结构是 stCoRoutineEnv_t, 每个线程仅有一个调度器，pCallStack是128深度的协程调用栈，这就意味着如果  
在协程中递归创建运行新协程， 那么最大到128深度就会溢出。
iCallStackSize 表示目前调用栈上等待调度的协程数， pEpoll 是epoll的包装， 用于poll wait阻塞的
socket, 帮助协程调度和定时器的实现。  
pending_co 表示下一个运行的协程上下文， occupy_co 表示当前协程上下文。

**协程上下文**
```cpp
struct stCoRoutine_t
{
	stCoRoutineEnv_t *env;
	pfn_co_routine_t pfn;
	void *arg;
	coctx_t ctx;

	char cStart;
	char cEnd;
	char cIsMain;
	char cEnableSysHook;
	char cIsShareStack;

	void *pvEnv;

	stStackMem_t* stack_mem;

	//save satck buffer while confilct on same stack_buffer;
	char* stack_sp;
	unsigned int save_size;
	char* save_buffer;

	stCoSpec_t aSpec[1024];

};
```
协程上下文的数据结构是stCoRoutine_t, env 表示所在线程调度上下文， pfn 为协程运行函数， arg是函数的传入参数，  
ctx 是协程函数运行时资源（见_上下文切换_）， cEnableSysHook 表示是否hook 系统函数， cIsShareStack 表示该协程是否  
使用共享栈， save_buffer 仅当使用共享栈有效， 用于临时拷贝共享栈内容， stack_sp 表示计算协程当前栈顶地址， 用于计算运行栈使用大小， aSpec 为协程本地变量表。

## 上下文切换
协程对于linux 线程来说，本质是一个个函数的调用。 函数的运行所需的资源，主要是程序计数寄存器（pc）、运行栈空间， 以及必要的运算寄存器， 状态寄存器， 索引寄存器等。  
所以协程的上下文切换就是保存 前一个协程的运行资源， 并将载入待运行协程的运行资源即可。事实上linux提供了context函数族来处理上下文切换的问题。  
libco 并不是使用context系列函数，而是使用汇编重新实现上下文切换函数， 从而去掉context switch 函数中
不必要的寄存器和变量的切换，加快了上下文切换的速度。

## 共享栈原理
每次创建新的协程时， 共享栈的 alloc_id 自增， 并对stack_size取模作为栈数组的下标索引，从而等到
实际的运行栈内存。
```cpp
stStackMem_t* co_get_stackmem(stShareStack_t* share_stack)
{
	if (!share_stack)
	{
		return NULL;
	}
	int idx = share_stack->alloc_idx % share_stack->count;
	share_stack->alloc_idx++;

	return share_stack->stack_array[idx];
}
```

在协程上下文切换时，首先检查待运行协程的运行栈， 如果该运行栈的当前占用协程不是待运行协程， 那么保持当前占用协程的栈空间到save_buffer （动态分配内存）。当协程再次切换回来时， 最拷贝save_buffer 到共享栈， 从而恢复运行栈空间。

```cpp
env->pending_co = pending_co;
stCoRoutine_t* occupy_co = pending_co->stack_mem->occupy_co;
pending_co->stack_mem->occupy_co = pending_co;
env->occupy_co = occupy_co;
if (occupy_co && occupy_co != pending_co)
{
  save_stack_buffer(occupy_co);
}

coctx_swap(&(curr->ctx),&(pending_co->ctx) );

stCoRoutineEnv_t* curr_env = co_get_curr_thread_env();
stCoRoutine_t* update_occupy_co =  curr_env->occupy_co;
stCoRoutine_t* update_pending_co = curr_env->pending_co;
if (update_occupy_co && update_pending_co && update_occupy_co != update_pending_co)
{

if (update_pending_co->save_buffer && update_pending_co->save_size > 0)
{
  memcpy(update_pending_co->stack_sp, update_pending_co->save_buffer, update_pending_co->save_size);
}
```  

## epoll的调度

## 定时器


## hook原理  

### 强弱符号

编译器默认函数和初始化了的全局变量为强符号（Strong Symbol），未初始化的全局变量为弱符号（Weak Symbol）。  
>针对强弱符号的概念，链接器就会按照如下规则处理与选择被多次定义的全局符号：  
> 规则1：不允许强符号被多次定义（即不同的目标文件中不能有同名的强符号）；如果有多个强符号定义，则链接器报符号重复定义错误。  
> 规则2：如果一个符号在某个目标文件中是强符号，在其他文件中都是弱符号，那么选择强符号。  
> 规则3：如果一个符号在所有目标文件中都是弱符号，那么选择其中占用空间最大的一个。比如目标文件A定义全局变量global为int型，  
> 占4个字节；目标文件B定义global为doulbe型，占8个字节，那么目标文件A和B链接后，符号global占8个字节（尽量不要使用多个不  
同类型的弱符号，否则容易导致很难发现的程序错误）。  

socket系列函数的弱符号由运行时装载libc.so最终确定函数地址， libco源文件重写了socket系列函数， 这样在编译期socket系列函数被gcc 解释为强符号（初始化了的全局变量）。

当然最重要的是显示调用hook函数：
```cpp
void co_enable_hook_sys()
{
    stCoRoutine_t *co = GetCurrThreadCo();
    if( co )
    {
        co->cEnableSysHook = 1;
    }
}
```
co_enable_hook_sys 声明被协程显示调用，这样hook的源文件在链接期不会被忽略。因为如果该文件生成的目标文件中的符号如果最终全部都没有被强引用到的话(比如在一个.h文件中对某个函数进行声明，那么是对这个函数符号的弱引用，而对这个函数进行调用，则是对这个函数符号的强引用)，那么该目标文件在静态链接的过程中会被忽略掉。

如何保证第三方库调用的是libco自身实现的相关socket函数呢？[2]
简单而言，就是通过调整最终生成的可执行文件的链接顺序，使其全局符号表中的跟socket相关函数的符号为libco协程库中的符号。
这里需要简述一下目标文件生成最终的可执行文件的链接过程。
我们先简单看一下链接过程中会发生什么。在链接过程中，链接器会按顺序扫描输入的目标文件，将其中的符号加入全局符号表,再计算出合并后的各个段的长度和位置。之后将进行符号解析和重定位。

另外对于动态链接来说，其还将遵循如下一条规则

### 全局符号介入
>    linux下的动态链接器存在以下原则：当共享对象被load进来的时候，它的符号表会被合并到进程的全局符号表中(这里说的全局符号表并不是指里面的符号全部是全局符号，而是指这是一个汇总的符号表)，当一个符号需要加入全局符号表时，
如果相同的符号名已经存在，则后面加入的符号被忽略。  

由于glibc是c/cpp程序的运行库，因此它是最后以动态链接的形式链接进来的，我们可以保证其肯定是最后加入全局符号表的，由于全局符号介入机制，glibc中的相关socket函数符号被忽略了(但是libco中巧妙的运用RTLD_NEXT参数获取到了其地址)，也因此只要最终的可执行文件链接了libco协程库，就可以基本保证相关的socket函数被hook掉了。
但是有socket相关函数定义的动态库,不止glibc，pthread库中也有！不过也没有关系，只需要保证libco库位于pthread库之前链接即可：
```
gcc main.c -o test -LSOME_PATH -llibco -lpthread
```

## 局限  
1. 共享栈的拷贝切换，会导致协程间传递的地址失效。  
2. 通过epoll_wait实现的定时器，会导致CPU空转。

## 文献
[1] https://github.com/Tencent/libco  
[2] https://www.cnblogs.com/unnamedfish/p/8460441.html
