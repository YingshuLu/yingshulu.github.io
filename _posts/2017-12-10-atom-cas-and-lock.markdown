---
layout: post
title:  "原子 CAS 与 锁"
date:   2017-12-10 22:15:55
categories: Algorithm
---

## 原子操作
******
### 何为原子操作？

原子操作是指不会被其他例程调度机制打断的操作，这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个例程）。

### 如何实现原子操作  

从原子操作的概念来看， 只要确保操作期间不会被打断都可以视为原子性。
所以不仅基本类型，那些mutex, spin_lock 保护的操作也是原子操作。

* 基本类型的原子操作

gcc 提供了一系列基本类型的 [atomic memory access](https://gcc.gnu.org/onlinedocs/gcc-4.4.3/gcc/Atomic-Builtins.html).
 C++/Java 都提供了atomic的Library.

但是理解atomic需要深入到assembly层面， 以linux kernel 为例，int对应的原子类型是volatile的封装。

volatile 起到[内存屏障](https://en.wikipedia.org/wiki/Memory_barrier)的作用, 能够将CPU cache实时更新，确保所有CPU cache 都是最新值. 这一机制是CPU的缓存一致协议自动完成的。([CPU缓存一致协议MESI](https://blog.csdn.net/reliveIT/article/details/50450136))

在单核OS时代，实现原子操作很简单，只需要关中断阻止进程调度即可。
SMP-OS中，就需要使用CPU的原语"LOCK"，阻止多例程访问地址。

```
/*Linux Kernel atomic operations
* From: linux-2.6.0/linux-2.6.0/include/asm-x86_64/atomic.h
*/

typedef struct { volatile int counter; } atomic_t;

#define ATOMIC_INIT(i)	{ (i) }

/**
 * atomic_read - read atomic variable
 * @v: pointer of type atomic_t
 *
 * Atomically reads the value of @v.  Note that the guaranteed
 * useful range of an atomic_t is only 24 bits.
 */
#define atomic_read(v)		((v)->counter)

/**
 * atomic_set - set atomic variable
 * @v: pointer of type atomic_t
 * @i: required value
 *
 * Atomically sets the value of @v to @i.  Note that the guaranteed
 * useful range of an atomic_t is only 24 bits.
 */
#define atomic_set(v,i)		(((v)->counter) = (i))

/**
 * atomic_add - add integer to atomic variable
 * @i: integer value to add
 * @v: pointer of type atomic_t
 *
 * Atomically adds @i to @v.  Note that the guaranteed useful range
 * of an atomic_t is only 24 bits.
 */
static __inline__ void atomic_add(int i, atomic_t *v)
{
	__asm__ __volatile__(
		LOCK "addl %1,%0"
		:"=m" (v->counter)
		:"ir" (i), "m" (v->counter));
}

/**
 * atomic_sub - subtract the atomic variable
 * @i: integer value to subtract
 * @v: pointer of type atomic_t
 *
 * Atomically subtracts @i from @v.  Note that the guaranteed
 * useful range of an atomic_t is only 24 bits.
 */
static __inline__ void atomic_sub(int i, atomic_t *v)
{
	__asm__ __volatile__(
		LOCK "subl %1,%0"
		:"=m" (v->counter)
		:"ir" (i), "m" (v->counter));
}

/**
 * atomic_sub_and_test - subtract value from variable and test result
 * @i: integer value to subtract
 * @v: pointer of type atomic_t
 *
 * Atomically subtracts @i from @v and returns
 * true if the result is zero, or false for all
 * other cases.  Note that the guaranteed
 * useful range of an atomic_t is only 24 bits.
 */
static __inline__ int atomic_sub_and_test(int i, atomic_t *v)
{
	unsigned char c;

	__asm__ __volatile__(
		LOCK "subl %2,%0; sete %1"
		:"=m" (v->counter), "=qm" (c)
		:"ir" (i), "m" (v->counter) : "memory");
	return c;

```
* 锁机制保护的操作

  用户态的锁（mutex, spin_lock） 是有基本类型的原子操作实现的。在并发/并行的程序中，锁机制能够确保数据/计算的同步执行。

  mutex 会抢占锁， 如果失败会挂起线程， 从而让其他线程运行，在锁解除时，通知等待线程重新抢占。
  spin_lock 也是抢占， 但是失败会不停尝试， 不会主动挂起线程，对于临界区比较小的较为适合。

  这里插个有趣的话题， 如何选择mutex spin_lock?

  <<Java 多线程编程实战>> 里面有个比较形象的比喻, 让我想起这个[环岛和设红绿灯的十字路口，哪个更有效率?](http://daily.zhihu.com/story/7509196)

  其中结论:
  ```
  总车流量中低：当车流量较大时，如果是红绿灯信号控制的交叉口，可以通过增加车道数的方法提高通行能力，
  然而环岛的几何造型决定了它最适宜设置一或两条车道，否则车辆就要被迫在弯道内迅速变道，
  即不安全也影响通行效率。环岛的另一局限是车速：车流在环岛内只能慢行，无法像在信号交叉口一样快速通过。
  可见环岛的造型是把双刃剑，车流量适中时笑纳四方来车，车流量太大时就成了瓶颈。
  注意：当车流量非常低时，不设置信号灯（也不设置环岛）效率反而更高，主要是因为这样能允许司机酌情判断
  通行时机，避免等待信号所浪费的时间。
  ```

  mutex 就是交通信号灯，没抢在绿灯的都需要车等待，spin_lock 就是环岛。
  这样应该就知道什么时候用mutex和spin_lock了， 忘记了就到十字路口思考下。

  其实编程中的抽象都是来自人类对现实生活经验的总结与提取。

  ***后续blog会聊一聊这些锁和同步原语的实现。***
