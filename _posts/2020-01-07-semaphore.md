---
layout: post
title: "并发：semaphore"
subtitle: 'semaphore'
author: "cslqm"
header-style: text
tags:
  - Linux
---

上一篇对 spinlock 有了初步的了解。

本篇可以讲 semaphore 了。

## spinlock vs semaphore

Spinlock and semaphore differ mainly in four things:

### 1.What they are

A spinlock is one possible implementation of a lock, namely one that is implemented by busy waiting ("spinning"). A semaphore is a generalization of a lock (or, the other way around, a lock is a special case of a semaphore). Usually, but not necessarily, spinlocks are only valid within one process whereas semaphores can be used to synchronize between different processes, too.

A lock works for mutual exclusion, that is one thread at a time can acquire the lock and proceed with a "critical section" of code. Usually, this means code that modifies some data shared by several threads.

A semaphore has a counter and will allow itself being acquired by one or several threads, depending on what value you post to it, and (in some implementations) depending on what its maximum allowable value is.

Insofar, one can consider a lock a special case of a semaphore with a maximum value of 1.

### 2.What they do

As stated above, a spinlock is a lock, and therefore a mutual exclusion (strictly 1 to 1) mechanism. It works by repeatedly querying and/or modifying a memory location, usually in an atomic manner. This means that acquiring a spinlock is a "busy" operation that possibly burns CPU cycles for a long time (maybe forever!) while it effectively achieves "nothing".

The main incentive for such an approach is the fact that a context switch has an overhead equivalent to spinning a few hundred (or maybe thousand) times, so if a lock can be acquired by burning a few cycles spinning, this may overall very well be more efficient. Also, for realtime applications it may not be acceptable to block and wait for the scheduler to come back to them at some far away time in the future.

A semaphore, by contrast, either does not spin at all, or only spins for a very short time (as an optimization to avoid the syscall overhead). If a semaphore cannot be acquired, it blocks, giving up CPU time to a different thread that is ready to run. This may of course mean that a few milliseconds pass before your thread is scheduled again, but if this is no problem (usually it isn't) then it can be a very efficient, CPU-conservative approach.

### 3.How they behave in presence of congestion

It is a common misconception that spinlocks or lock-free algorithms are "generally faster", or that they are only useful for "very short tasks" (ideally, no synchronization object should be held for longer than absolutely necessary, ever).

The one important difference is how the different approaches behave in presence of congestion.

A well-designed system normally has low or no congestion (this means not all threads try to acquire the lock at the exact same time). For example, one would normally not write code that acquires a lock, then loads half a megabyte of zip-compressed data from the network, decodes and parses the data, and finally modifies a shared reference (append data to a container, etc.) before releasing the lock. Instead, one would acquire the lock only for the purpose of accessing the shared resource.

Since this means that there is considerably more work outside the critical section than inside it, naturally the likelihood for a thread being inside the critical section is relatively low, and thus few threads are contending for the lock at the same time. Of course every now and then two threads will try to acquire the lock at the same time (if this couldn't happen you wouldn't need a lock!), but this is rather the exception than the rule in a "healthy" system.

In such a case, a spinlock greatly outperforms a semaphore because if there is no lock congestion, the overhead of acquiring the spinlock is a mere dozen cycles as compared to hundreds/thousands of cycles for a context switch or 10-20 million cycles for losing the remainder of a time slice.

On the other hand, given high congestion, or if the lock is being held for lengthy periods (sometimes you just can't help it!), a spinlock will burn insane amounts of CPU cycles for achieving nothing.

A semaphore (or mutex) is a much better choice in this case, as it allows a different thread to run useful tasks during that time. Or, if no other thread has something useful to do, it allows the operating system to throttle down the CPU and reduce heat / conserve energy.

Also, on a single-core system, a spinlock will be quite inefficient in presence of lock congestion, as a spinning thread will waste its complete time waiting for a state change that cannot possibly happen (not until the releasing thread is scheduled, which isn't happening while the waiting thread is running!). Therefore, given any amount of contention, acquiring the lock takes around 1 1/2 time slices in the best case (assuming the releasing thread is the next one being scheduled), which is not very good behaviour.

### 4. How they're implemented
A semaphore will nowadays typically wrap sys_futex under Linux (optionally with a spinlock that exits after a few attempts).

A spinlock is typically implemented using atomic operations, and without using anything provided by the operating system. In the past, this meant using either compiler intrinsics or non-portable assembler instructions. Meanwhile both C++11 and C11 have atomic operations as part of the language, so apart from the general difficulty of writing provably correct lock-free code, it is now possible to implement lock-free code in an entirely portable and (almost) painless way.


### 关于信号量

#### 什么是信号量

信号量的使用主要是用来保护共享资源，使得资源在一个时刻只有一个进程（线程）所拥有。

	信号量的值为正的时候，说明它空闲。所测试的线程可以锁定而使用它。若为0，说明它被占用，测试的线程要进入睡眠队列中，等待被唤醒。

为了防止出现因多个程序同时访问一个共享资源而引发的一系列问题，我们需要一种方法，它可以通过生成并使用令牌来授权，在任一时刻只能有一个执行线程访问代码的临界区域。

临界区域是指执行数据更新的代码需要独占式地执行。而信号量就可以提供这样的一种访问机制，让一个临界区同一时间只有一个线程在访问它，也就是说信号量是用来调协进程对共享资源的访问的。

信号量是一个特殊的变量，程序对其访问都是原子操作，且只允许对它进行等待（即P(信号变量))和发送（即V(信号变量))信息操作。

最简单的信号量是只能取0和1的变量，这也是信号量最常见的一种形式，叫做二进制信号量。而可以取多个正整数的信号量被称为通用信号量。这里主要讨论二进制信号量。

#### 工作原理

主要就是有三个字母了，P、V，S（信标/标志位/旗标）。

计数信号量具备两种操作动作，称为V（signal()）与P（wait()）（即部分参考书常称的“PV操作”）。V操作会增加信号标S的数值，P操作会减少它。

运作方式：

初始化，给与它一个非负数的整数值。

运行P（wait()），信号标S的值将被减少。企图进入临界区段的进程，需要先运行P（wait()）。当信号标S减为负值时，进程会被挡住，不能继续；当信号标S不为负值时，进程可以获准进入临界区段。

运行V（signal()），信号标S的值会被增加。结束离开临界区段的进程，将会运行V（signal()）。当信号标S不为负值时，先前被挡住的其他进程，将可获准进入临界区段。

	举个例子，就是两个进程共享信号量sv，一旦其中一个进程执行了P(sv)操作，它将得到信号量，并可以进入临界区，使sv减1。而第二个进程将被阻止进入临界区，因为当它试图执行P(sv)时，sv为0，它会被挂起以等待第一个进程离开临界区域并执行V(sv)释放信号量，这时第二个进程就可以恢复执行。

#### 信号量区分

POSIX 信号量和 XSI 信号量。

Linux提供两种信号量：

- 内核信号量，由内核控制路径使用
- 用户态进程使用的信号量，这种信号量又分为 POSIX 信号量和 SYSTEM V 信号量。

POSIX 信号量又分为有名信号量和无名信号量

有名信号量，其值保存在文件中, 所以它可以用于线程也可以用于进程间的同步。无名信号量，其值保存在内存中。


对 POSIX 来说，信号量是个非负整数。常用于线程间同步。

而 SYSTEM V 信号量则是一个或多个信号量的集合，它对应的是一个信号量结构体，这个结构体是为 SYSTEM V IPC 服务的，信号量只不过是它的一部分。常用于进程间同步。

POSIX 信号量的引用头文件是<semaphore.h>，而 SYSTEM V 信号量的引用头文件是<sys/sem.h>

从使用的角度，SYSTEM V 信号量是复杂的，而 POSIX 信号量是简单。比如，POSIX 信号量的创建和初始化或 PV 操作就很非常方便。

### Linux 内核信号量

Linux 内核的信号量在概念和原理上与用户态的 SYSTEM V 的 IPC 机制信号量是一样的，但是它绝不可能在内核之外使用，它是一种睡眠锁。

如果有一个任务想要获得已经被占用的信号量时，信号量会将其放入一个等待队列（它不是站在外面痴痴地等待而是将自己的名字写在任务队列中）然后让其睡眠。

当持有信号量的进程将信号释放后，处于等待队列中的一个任务将被唤醒（因为队列中可能不止一个任务），并让其获得信号量。

这一点与自旋锁不同，处理器可以去执行其它代码。

它与自旋锁的差异：由于争用信号量的进程在等待锁重新变为可用时会睡眠，所以信号量适用于锁会被长时间持有的情况；

相反，锁被短时间持有时，使用信号量就不太适宜了，因为睡眠、维护等待队列以及唤醒所花费的开销可能比锁占用的全部时间表还要长；

由于执行线程在锁被争用时会睡眠，所以只能在进程上下文中才能获得信号量锁，因为在中断上下文中是不能进行调试的；持有信号量的进行也可以去睡眠，当然也可以不睡眠，因为当其他进程争用信号量时不会因此而死锁；不能同时占用信号量和自旋锁，因为自旋锁不可以睡眠而信号量锁可以睡眠。相对而来说信号量比较简单，它不会禁止内核抢占，持有信号量的代码可以被抢占。

信号量还有一个特征，就是它允许多个持有者，而自旋锁在任何时候只能允许一个持有者。

当然我们经常遇到也是只有一个持有者，这种信号量叫二值信号量或者叫互斥信号量。允许有多个持有者的信号量叫计数信号量，在初始化时要说明最多允许有多少个持有者（Count 值）

信号量在创建时需要设置一个初始值，表示同时可以有几个任务可以访问该信号量保护的共享资源，初始值为1就变成互斥锁（Mutex），即同时只能有一个任务可以访问信号量保护的共享资源。

当任务访问完被信号量保护的共享资源后，必须释放信号量，释放信号量通过把信号量的值加 1 实现，如果信号量的值为非正数，表明有任务等待当前信号量，因此它也唤醒所有等待该信号量的任务。

####  内核信号量的构成

内核信号量类似于自旋锁，因为当锁关闭着时，它不允许内核控制路径继续进行。然而，当内核控制路径试图获取内核信号量锁保护的忙资源时，相应的进程就被挂起。只有在资源被释放时，进程才再次变为可运行。

只有可以睡眠的函数才能获取内核信号量；中断处理程序和可延迟函数都不能使用内核信号量。

内核信号量是struct semaphore类型的对象，在内核源码中位于include\linux\semaphore.h文件

``` c
## 伪代码
struct semaphore
{
	atomic_t count;
	int sleepers;
	wait_queue_head_t wait;
}
```

| 成员 | 描述 |
| :-: | :-: |
| count | 相当于信号量的值，大于0，资源空闲；等于0，资源忙，但没有进程等待这个保护的资源；小于0，资源不可用，并至少有一个进程等待资源 |
| wait | 存放等待队列链表的地址，当前等待资源的所有睡眠进程都会放在这个链表中 |
| sleepers | 存放一个标志，表示是否有一些进程在信号量上睡眠 |


#### 等待队列

上面已经提到了内核信号量使用了等待队列 wait_queue 来实现阻塞操作。

当某任务由于没有某种条件没有得到满足时，它就被挂到等待队列中睡眠。当条件得到满足时，该任务就被移出等待队列，此时并不意味着该任务就被马上执行，因为它又被移进工作队列中等待 CPU 资源，在适当的时机被调度。

内核信号量是在内部使用等待队列的，也就是说该等待队列对用户是隐藏的，无须用户干涉。由用户真正使用的等待队列我们将在另外的篇章进行详解。

#### 相关的函数(头文件)

	注意：文章中见到的代码都是 2.6.32-27 版本内核。

``` c
static inline void sema_init(struct semaphore *sem, int val)
{
	static struct lock_class_key __key;
	*sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);
	lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}

#define init_MUTEX(sem)		sema_init(sem, 1)
#define init_MUTEX_LOCKED(sem)	sema_init(sem, 0)

extern void down(struct semaphore *sem);
extern int __must_check down_interruptible(struct semaphore *sem);
extern int __must_check down_killable(struct semaphore *sem);
extern int __must_check down_trylock(struct semaphore *sem);
extern int __must_check down_timeout(struct semaphore *sem, long jiffies);
extern void up(struct semaphore *sem);
```

#### 与初始化有关的函数

``` c
# linux/semaphore.h
#define __SEMAPHORE_INITIALIZER(name, n)				\
{									\
	.lock		= __SPIN_LOCK_UNLOCKED((name).lock),		\
	.count		= n,						\
	.wait_list	= LIST_HEAD_INIT((name).wait_list),		\
}
```

该宏声明一个信号量name是直接将结构体中count值设置成n，此时信号量可用于实现进程间的互斥量。

``` c
#define DECLARE_MUTEX(name)	\
	struct semaphore name = __SEMAPHORE_INITIALIZER(name, 1)
```

该宏声明一个互斥锁 name，但把它的初始值设置为1。

``` c
static inline void sema_init(struct semaphore *sem, int val)
{
	static struct lock_class_key __key;
	*sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);
	lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}
```
初始化一个 sem。该函用于数初始化设置信号量的初值，它设置信号量sem的值为val。

``` c
#define init_MUTEX(sem)		sema_init(sem, 1)
```
初始化一个默认打开状态的 sem。初始化一个互斥锁，即它把信号量sem的值设置为1。

``` c
#define init_MUTEX_LOCKED(sem)	sema_init(sem, 0)
```
初始化一个互斥锁，但它把信号量sem的值设置为0，即一开始就处在已锁状态

	注意：对于信号量的初始化函数 Linux 最新版本存在变化，如 init_MUTEX 和 init_MUTEX_LOCKED 等初始化函数目前新的内核中已经没有或者更换了了名字等
	因此建议以后在编程中遇到需要使用信号量的时候尽量采用sema_init(struct semaphore *sem, int val)函数，因为这个函数就目前为止从未发生变化。

#### 获取信号量–申请内核信号量所保护的资源

``` c
/**
 * down - acquire the semaphore
 * @sem: the semaphore to be acquired
 *
 * Acquires the semaphore.  If no more tasks are allowed to acquire the
 * semaphore, calling this function will put the task to sleep until the
 * semaphore is released.
 *
 * Use of this function is deprecated, please use down_interruptible() or
 * down_killable() instead.
 */
void down(struct semaphore *sem)
{
	unsigned long flags;

	spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		__down(sem);
	spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(down);
```

1. sem->count > 0 是表示未锁。所以直接减 1。
2. 如果等于或者小于 0 表示已锁。调用__down(sem)。

``` c
static noinline void __sched __down(struct semaphore *sem)
{
	__down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}

```

直接调用__down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT)。

``` c
/*
 * Because this function is inlined, the 'state' parameter will be
 * constant, and thus optimised away by the compiler.  Likewise the
 * 'timeout' parameter for the cases without timeouts.
 */
static inline int __sched __down_common(struct semaphore *sem, long state,
								long timeout)
{
	struct task_struct *task = current;
	struct semaphore_waiter waiter;

	list_add_tail(&waiter.list, &sem->wait_list);
	waiter.task = task;
	waiter.up = 0;

	for (;;) {
		if (signal_pending_state(state, task))
			goto interrupted;
		if (timeout <= 0)
			goto timed_out;
		__set_task_state(task, state);
		spin_unlock_irq(&sem->lock);
		timeout = schedule_timeout(timeout);
		spin_lock_irq(&sem->lock);
		if (waiter.up)
			return 0;
	}

 timed_out:
	list_del(&waiter.list);
	return -ETIME;

 interrupted:
	list_del(&waiter.list);
	return -EINTR;
}
```

1. list_add_tail(&waiter.list, &sem->wait_list) 向等待列表中添加一个空的新任务。
2. 更新任务状态__set_task_state(task, state)。
3. 释放锁spin_unlock_irq(&sem->lock)，睡眠操作第一步。
4. schedule_timeout(timeout) 开始睡眠。
5. signal_pending_state(state, task) 可中断在这里进行。

___


``` c
/**
 * down_interruptible - acquire the semaphore unless interrupted
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a signal, this function will return -EINTR.
 * If the semaphore is successfully acquired, this function returns 0.
 */
int down_interruptible(struct semaphore *sem)
{
	unsigned long flags;
	int result = 0;

	spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		result = __down_interruptible(sem);
	spin_unlock_irqrestore(&sem->lock, flags);

	return result;
}
EXPORT_SYMBOL(down_interruptible);
```
在 sem-> count 小于等于 0 时调用__down_interruptible(sem)。

``` c
static noinline int __sched __down_interruptible(struct semaphore *sem)
{
	return __down_common(sem, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
```
发起可中断的睡眠。

___


``` c
/**
 * down_killable - acquire the semaphore unless killed
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a fatal signal, this function will return
 * -EINTR.  If the semaphore is successfully acquired, this function returns
 * 0.
 */
int down_killable(struct semaphore *sem)
{
	unsigned long flags;
	int result = 0;

	spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		result = __down_killable(sem);
	spin_unlock_irqrestore(&sem->lock, flags);

	return result;
}
EXPORT_SYMBOL(down_killable);
```
在加锁状态下调用__down_killable(sem)。

``` c
static noinline int __sched __down_killable(struct semaphore *sem)
{
	return __down_common(sem, TASK_KILLABLE, MAX_SCHEDULE_TIMEOUT);
}
```
进行可以 kill 的睡眠。

___


``` c
/**
 * down_trylock - try to acquire the semaphore, without waiting
 * @sem: the semaphore to be acquired
 *
 * Try to acquire the semaphore atomically.  Returns 0 if the mutex has
 * been acquired successfully or 1 if it it cannot be acquired.
 *
 * NOTE: This return value is inverted from both spin_trylock and
 * mutex_trylock!  Be careful about this when converting code.
 *
 * Unlike mutex_trylock, this function can be used from interrupt context,
 * and the semaphore can be released by any task or interrupt.
 */
int down_trylock(struct semaphore *sem)
{
	unsigned long flags;
	int count;

	spin_lock_irqsave(&sem->lock, flags);
	count = sem->count - 1;
	if (likely(count >= 0))
		sem->count = count;
	spin_unlock_irqrestore(&sem->lock, flags);

	return (count < 0);
}
EXPORT_SYMBOL(down_trylock);
```
将 sem->count - 1 的结果赋值给 count，count 为 >= 0 表示可以未锁。
return (count < 0) 

| 返回值 | count 值 | 描述 |
| :-: | :-: | :-: |
| 1 | 负数 | 已锁，trylock 是失败的 |
| 0 | 大于等于 0 | 未锁，trylock 是成功的 |

可以看到 down_trylock 不会进行休眠。不成功直接返回非负数。

___

``` c
/**
 * down_timeout - acquire the semaphore within a specified time
 * @sem: the semaphore to be acquired
 * @jiffies: how long to wait before failing
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the semaphore is not released within the specified number of jiffies,
 * this function returns -ETIME.  It returns 0 if the semaphore was acquired.
 */
int down_timeout(struct semaphore *sem, long jiffies)
{
	unsigned long flags;
	int result = 0;

	spin_lock_irqsave(&sem->lock, flags);
	if (likely(sem->count > 0))
		sem->count--;
	else
		result = __down_timeout(sem, jiffies);
	spin_unlock_irqrestore(&sem->lock, flags);

	return result;
}
EXPORT_SYMBOL(down_timeout);
```

在已锁时，发起一个有超时时间的睡眠。

``` c
static noinline int __sched __down_timeout(struct semaphore *sem, long jiffies)
{
	return __down_common(sem, TASK_UNINTERRUPTIBLE, jiffies);
}
```

#### 释放内核信号量所保护的资源

``` c
/**
 * up - release the semaphore
 * @sem: the semaphore to release
 *
 * Release the semaphore.  Unlike mutexes, up() may be called from any
 * context and even by tasks which have never called down().
 */
void up(struct semaphore *sem)
{
	unsigned long flags;

	spin_lock_irqsave(&sem->lock, flags);
	if (likely(list_empty(&sem->wait_list)))
		sem->count++;
	else
		__up(sem);
	spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(up);
```
1. list_empty(&sem->wait_list) 如果任务列表为空，直接sem->count 加 1 开锁。
2. &sem->wait_list 不为空，存在等待任务，调用__up(sem) 取出一个任务，尝试唤醒此任务。

``` c
static noinline void __sched __up(struct semaphore *sem)
{
	struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
						struct semaphore_waiter, list);
	list_del(&waiter->list);
	waiter->up = 1;
	wake_up_process(waiter->task);
}
```

1. 从任务列表中删除任务 list_del(&waiter->list)。
2. wake_up_process(waiter->task) 唤醒任务。

#### semaphore 例子

``` c
/*                                                     
 * $Id: hellop.c,v 1.4 2004/09/26 07:02:43 gregkh Exp $ 
 */                                                    
#include <linux/init.h>
#include <linux/module.h>
#include <linux/semaphore.h>
#include <linux/fs.h>

#include <linux/wait.h>
#include <linux/sched.h>
#include <asm/uaccess.h>

MODULE_LICENSE("Dual BSD/GPL");

/*                                                        
 * These lines, although not shown in the book,           
 * are needed to make hello.c run properly even when      
 * your kernel has version support enabled                
 */                                                       


#define MAJOR_NUM 0

static ssize_t globalvar_read(struct file *, char *, size_t, loff_t*);
static ssize_t globalvar_write(struct file *, const char *, size_t, loff_t*);

struct file_operations globalvar_fops =
{
        read : globalvar_read,
        write: globalvar_write,
};

static int global_var = 0;
static struct semaphore sem;
static wait_queue_head_t outq;
static int flag = 0;

static int __init globalvar_init(void)
{
        int ret;
        ret = register_chrdev(MAJOR_NUM, "globalvar", &globalvar_fops);
        if (ret)
        {
                printk("globalvar register success");
                init_MUTEX(&sem);   //初始化一个互斥锁，即它把信号量sem的值设置为1
                init_waitqueue_head(&outq); //初始化等待队列outq
                return 0;
        }
        return ret;
}

static void __exit globalvar_exit(void)
{
        unregister_chrdev(MAJOR_NUM, "globalvar");
}

//读设备
static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
        //等待数据可获得
        if (wait_event_interruptible(outq, flag != 0)) // wait_event_interruptible把进程状态设为TASK_INTERRUPTIBLE，nonexclusive
        {
                return - ERESTARTSYS;
        }

        if (down_interruptible(&sem)) //获得信号量，相当于获得锁
        {
                return - ERESTARTSYS;
        }

        flag = 0;
        if (copy_to_user(buf, &global_var, sizeof(int)))
        {
                up(&sem);  //如果读取失败，则释放信号量，相当于释放锁
                return -EFAULT;
        }
        up(&sem); //若读取成功也释放信号量
        return sizeof(int);
}

//写设备
static ssize_t globalvar_write(struct file *filp, const char *buf, size_t len,loff_t *off)
{
        if (down_interruptible(&sem)) //获得信号量
        {
                return -ERESTARTSYS;
        }
        if (copy_from_user(&global_var, buf, sizeof(int)))
        {
                up(&sem); //写失败后释放锁
                return -EFAULT;
        }
        up(&sem); //写成功后，也释放信号量
        flag = 1;
        wake_up_interruptible(&outq); //唤醒等待进程，通知已经有数据可以读取
        return sizeof(int);
}

module_init(globalvar_init);
module_exit(globalvar_exit);
```

转自： https://blog.csdn.net/wangchaoxjtuse/article/details/6025385


### Linux 读写信号量

ldd 中紧跟着就讲到了 rwsem，

跟自旋锁一样，信号量也有区分读-写信号量之分

如果一个读写信号量当前没有被写者拥有并且也没有写者等待读者释放信号量，那么任何读者都可以成功获得该读写信号量；

否则，读者必须被挂起直到写者释放该信号量。如果一个读写信号量当前没有被读者或写者拥有并且也没有写者等待该信号量，那么一个写者可以成功获得该读写信号量，否则写者将被挂起，直到没有任何访问者。因此，写者是排他性的，独占性的。

读写信号量有两种实现，一种是通用的，不依赖于硬件架构，因此，增加新的架构不需要重新实现它，但缺点是性能低，获得和释放读写信号量的开销大；另一种是架构相关的，因此性能高，获取和释放读写信号量的开销小，但增加新的架构需要重新实现。在内核配置时，可以通过选项去控制使用哪一种实现。

2.6.32.27 内核中的 rwsem 定义。

``` c
# asm/rwsem.h
struct rw_semaphore {
	rwsem_count_t		count;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
};
```



读写信号量的相关API有：

``` c
# asm/rwsem.h
DECLARE_RWSEM(name)

#define DECLARE_RWSEM(name)					\
	struct rw_semaphore name = __RWSEM_INITIALIZER(name)

#define __RWSEM_INITIALIZER(name)				\
{								\
	RWSEM_UNLOCKED_VALUE, __SPIN_LOCK_UNLOCKED((name).wait_lock), \
	LIST_HEAD_INIT((name).wait_list) __RWSEM_DEP_MAP_INIT(name) \
}
```
该宏声明一个读写信号量 name 并对其进行初始化。

___

``` c
#define init_rwsem(sem)						\
do {								\
	static struct lock_class_key __key;			\
								\
	__init_rwsem((sem), #sem, &__key);			\
} while (0)

extern void __init_rwsem(struct rw_semaphore *sem, const char *name,
			 struct lock_class_key *key);

# lib/rwsem.h 可能不对，没有调试过
/*
 * Initialize an rwsem:
 */
void __init_rwsem(struct rw_semaphore *sem, const char *name,
		  struct lock_class_key *key)
{
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/*
	 * Make sure we are not reinitializing a held semaphore:
	 */
	debug_check_no_locks_freed((void *)sem, sizeof(*sem));
	lockdep_init_map(&sem->dep_map, name, key, 0);
#endif
	sem->count = RWSEM_UNLOCKED_VALUE;
	spin_lock_init(&sem->wait_lock);
	INIT_LIST_HEAD(&sem->wait_list);
}

EXPORT_SYMBOL(__init_rwsem);
```

该函数对读写信号量 sem 进行初始化。


#### 获取信号量–申请内核信号量所保护的资源

``` c
# kernel/rwsem.c
/*
 * lock for reading
 */
void __sched down_read(struct rw_semaphore *sem)
{
	might_sleep();
	rwsem_acquire_read(&sem->dep_map, 0, 0, _RET_IP_);

	LOCK_CONTENDED(sem, __down_read_trylock, __down_read);
}

EXPORT_SYMBOL(down_read);

# asm/rwsem.h
/*
 * lock for reading
 */
static inline void __down_read(struct rw_semaphore *sem)
{
	asm volatile("# beginning down_read\n\t"
		     LOCK_PREFIX _ASM_INC "(%1)\n\t"
		     /* adds 0x00000001, returns the old value */
		     "  jns        1f\n"
		     "  call call_rwsem_down_read_failed\n"
		     "1:\n\t"
		     "# ending down_read\n\t"
		     : "+m" (sem->count)
		     : "a" (sem)
		     : "memory", "cc");
}

# /arch/x86/lib/rwsem_64.S
/* Fix up special calling conventions */
ENTRY(call_rwsem_down_read_failed)
        save_common_regs
        pushq %rdx
        movq %rax,%rdi
        call rwsem_down_read_failed
        popq %rdx
        restore_common_regs
        ret
        ENDPROC(call_rwsem_down_read_failed)

```

1. 在未配置 CONFIG_DEBUG_SPINLOCK_SLEEP 情况下，might_sleep 和 rwsem_acquire_read 为空。
2. LOCK_CONTENDED 宏最后为__down_read(sem)。
3. 读者调用该函数来得到读写信号量sem。该函数会导致调用者睡眠，因此只能在进程上下文使用

``` c
# kernel/rwsem.c
/*
 * trylock for reading -- returns 1 if successful, 0 if contention
 */
int down_read_trylock(struct rw_semaphore *sem)
{
	int ret = __down_read_trylock(sem);

	if (ret == 1)
		rwsem_acquire_read(&sem->dep_map, 0, 1, _RET_IP_);
	return ret;
}

EXPORT_SYMBOL(down_read_trylock);

# arch/x86/asm/rwsem.h
/*
 * trylock for reading -- returns 1 if successful, 0 if contention
 */
static inline int __down_read_trylock(struct rw_semaphore *sem)
{
	rwsem_count_t result, tmp;
	asm volatile("# beginning __down_read_trylock\n\t"
		     "  mov          %0,%1\n\t"
		     "1:\n\t"
		     "  mov          %1,%2\n\t"
		     "  add          %3,%2\n\t"
		     "  jle	     2f\n\t"
		     LOCK_PREFIX "  cmpxchg  %2,%0\n\t"
		     "  jnz	     1b\n\t"
		     "2:\n\t"
		     "# ending __down_read_trylock\n\t"
		     : "+m" (sem->count), "=&a" (result), "=&r" (tmp)
		     : "i" (RWSEM_ACTIVE_READ_BIAS)
		     : "memory", "cc");
	return result >= 0 ? 1 : 0;
}
```

1. 在未配置 CONFIG_DEBUG_SPINLOCK_SLEEP 情况下，rwsem_acquire_read 为空。
2. 该函数类似于down_read，只是它不会导致调用者睡眠。它尽力得到读写信号量sem，如果能够立即得到，它就得到该读写信号量，并且返回1，否则表示不能立刻得到该信号量，返回0。因此，它也可以在中断上下文使用。

``` c
# kernel/rwsem.c
/*
 * lock for writing
 */
void __sched down_write(struct rw_semaphore *sem)
{
	might_sleep();
	rwsem_acquire(&sem->dep_map, 0, 0, _RET_IP_);

	LOCK_CONTENDED(sem, __down_write_trylock, __down_write);
}

EXPORT_SYMBOL(down_write);

# arch/x86/include/asm/rwsem.h
static inline void __down_write(struct rw_semaphore *sem)
{
	__down_write_nested(sem, 0);
}

/*
 * lock for writing
 */
static inline void __down_write_nested(struct rw_semaphore *sem, int subclass)
{
	rwsem_count_t tmp;

	tmp = RWSEM_ACTIVE_WRITE_BIAS;
	asm volatile("# beginning down_write\n\t"
		     LOCK_PREFIX "  xadd      %1,(%2)\n\t"
		     /* subtract 0x0000ffff, returns the old value */
		     "  test      %1,%1\n\t"
		     /* was the count 0 before? */
		     "  jz        1f\n"
		     "  call call_rwsem_down_write_failed\n"
		     "1:\n"
		     "# ending down_write"
		     : "+m" (sem->count), "=d" (tmp)
		     : "a" (sem), "1" (tmp)
		     : "memory", "cc");
}

# /arch/x86/lib/rwsem_64.S
ENTRY(call_rwsem_down_write_failed)
        save_common_regs
        movq %rax,%rdi
        call rwsem_down_write_failed
        restore_common_regs
        ret
        ENDPROC(call_rwsem_down_write_failed)
```

1. 在未配置 CONFIG_DEBUG_SPINLOCK_SLEEP 情况下，might_sleep 和 rwsem_acquire 为空。
2. LOCK_CONTENDED 宏实际为__down_write(sem)。
3. 写者使用该函数来得到读写信号量sem，它也会导致调用者睡眠，因此只能在进程上下文使用。

``` c
# kernel/rwsem.h
/*
 * trylock for writing -- returns 1 if successful, 0 if contention
 */
int down_write_trylock(struct rw_semaphore *sem)
{
	int ret = __down_write_trylock(sem);

	if (ret == 1)
		rwsem_acquire(&sem->dep_map, 0, 1, _RET_IP_);
	return ret;
}

# arch/x86/include/asm/rwsem.h
/*
 * trylock for writing -- returns 1 if successful, 0 if contention
 */
static inline int __down_write_trylock(struct rw_semaphore *sem)
{
	rwsem_count_t ret = cmpxchg(&sem->count,
				    RWSEM_UNLOCKED_VALUE,
				    RWSEM_ACTIVE_WRITE_BIAS);
	if (ret == RWSEM_UNLOCKED_VALUE)
		return 1;
	return 0;
}

```

1. 在未配置 CONFIG_DEBUG_SPINLOCK_SLEEP 情况下， rwsem_acquire 为空。
2. 该函数类似于down_write，只是它不会导致调用者睡眠。该函数尽力得到读写信号量，如果能够立刻获得，就获得该读写信号量并且返回1，否则表示无法立刻获得，返回0。它可以在中断上下文使用。

#### 释放内核信号量所保护的资源

``` c
# linux/rwsem.h
/*
 * release a read lock
 */
void up_read(struct rw_semaphore *sem)
{
	rwsem_release(&sem->dep_map, 1, _RET_IP_);

	__up_read(sem);
}

# arch/x86/include/asm/rwsem.h
/*
 * unlock after reading
 */
static inline void __up_read(struct rw_semaphore *sem)
{
	rwsem_count_t tmp = -RWSEM_ACTIVE_READ_BIAS;
	asm volatile("# beginning __up_read\n\t"
		     LOCK_PREFIX "  xadd      %1,(%2)\n\t"
		     /* subtracts 1, returns the old value */
		     "  jns        1f\n\t"
		     "  call call_rwsem_wake\n"
		     "1:\n"
		     "# ending __up_read\n"
		     : "+m" (sem->count), "=d" (tmp)
		     : "a" (sem), "1" (tmp)
		     : "memory", "cc");
}

# /arch/x86/lib/rwsem_64.S
ENTRY(call_rwsem_wake)
        decw %dx    /* do nothing if still outstanding active readers */
        jnz 1f
        save_common_regs
        movq %rax,%rdi
        call rwsem_wake
        restore_common_regs
1:      ret
        ENDPROC(call_rwsem_wake)
```
1. rwsem_release 啥也没做，空的。
2. 读者使用该函数释放读写信号量sem。它与down_read或down_read_trylock配对使用。如果down_read_trylock返回0，不需要调用up_read来释放读写信号量，因为根本就没有获得信号量。

``` c
# linux/rwsem.h
/*
 * release a write lock
 */
void up_write(struct rw_semaphore *sem)
{
	rwsem_release(&sem->dep_map, 1, _RET_IP_);

	__up_write(sem);
}

EXPORT_SYMBOL(up_write);

# arch/x86/include/asm/rwsem.h
/*
 * unlock after writing
 */
static inline void __up_write(struct rw_semaphore *sem)
{
	rwsem_count_t tmp;
	asm volatile("# beginning __up_write\n\t"
		     LOCK_PREFIX "  xadd      %1,(%2)\n\t"
		     /* tries to transition
			0xffff0001 -> 0x00000000 */
		     "  jz       1f\n"
		     "  call call_rwsem_wake\n"
		     "1:\n\t"
		     "# ending __up_write\n"
		     : "+m" (sem->count), "=d" (tmp)
		     : "a" (sem), "1" (-RWSEM_ACTIVE_WRITE_BIAS)
		     : "memory", "cc");
}
```

1. rwsem_release 什么也没有做。
2. 写者调用该函数释放信号量sem。它与down_write或down_write_trylock配对使用。如果down_write_trylock返回0，不需要调用up_write，因为返回0表示没有获得该读写信号量


``` c
# linux/rwsem.h
/*
 * downgrade write lock to read lock
 */
void downgrade_write(struct rw_semaphore *sem)
{
	/*
	 * lockdep: a downgraded write will live on as a write
	 * dependency.
	 */
	__downgrade_write(sem);
}

# arch/x86/include/asm/rwsem.h
/*
 * downgrade write lock to read lock
 */
static inline void __downgrade_write(struct rw_semaphore *sem)
{
	asm volatile("# beginning __downgrade_write\n\t"
		     LOCK_PREFIX _ASM_ADD "%2,(%1)\n\t"
		     /*
		      * transitions 0xZZZZ0001 -> 0xYYYY0001 (i386)
		      *     0xZZZZZZZZ00000001 -> 0xYYYYYYYY00000001 (x86_64)
		      */
		     "  jns       1f\n\t"
		     "  call call_rwsem_downgrade_wake\n"
		     "1:\n\t"
		     "# ending __downgrade_write\n"
		     : "+m" (sem->count)
		     : "a" (sem), "er" (-RWSEM_WAITING_BIAS)
		     : "memory", "cc");
}

# arch/x86/lib/rwsem_64.S
/* Fix up special calling conventions */
ENTRY(call_rwsem_downgrade_wake)
        save_common_regs
        pushq %rdx
        movq %rax,%rdi
        call rwsem_downgrade_wake
        popq %rdx
        restore_common_regs
        ret
        ENDPROC(call_rwsem_downgrade_wake)
```

该函数用于把写者降级为读者，这有时是必要的。因为写者是排他性的，因此在写者保持读写信号量期间，任何读者或写者都将无法访问该读写信号量保护的共享资源，对于那些当前条件下不需要写访问的写者，降级为读者将，使得等待访问的读者能够立刻访问，从而增加了并发性，提高了效率。
读写信号量适于在读多写少的情况下使用，在linux内核中对进程的内存映像描述结构的访问就使用了读写信号量进行保护。
究竟什么时候使用自旋锁什么时候使用信号量，下面给出建议的方案
当对低开销、短期、中断上下文加锁，优先考虑自旋锁；当对长期、持有锁需要休眠的任务，优先考虑信号量。

#### rwsem 例子

注释未修改。

``` c
/*                                                     
 * $Id: hellop.c,v 1.4 2004/09/26 07:02:43 gregkh Exp $ 
 */                                                    
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/rwsem.h>

#include <linux/wait.h>
#include <linux/sched.h>
#include <asm/uaccess.h>

MODULE_LICENSE("Dual BSD/GPL");

/*                                                        
 * These lines, although not shown in the book,           
 * are needed to make hello.c run properly even when      
 * your kernel has version support enabled                
 */                                                       


#define MAJOR_NUM 0

static ssize_t globalvar_read(struct file *, char *, size_t, loff_t*);
static ssize_t globalvar_write(struct file *, const char *, size_t, loff_t*);

struct file_operations globalvar_fops =
{
        read : globalvar_read,
        write: globalvar_write,
};

static int global_var = 0;
static struct rw_semaphore rwsem;
static wait_queue_head_t outq;
static int flag = 0;

static int __init globalvar_init(void)
{
        int ret;
        ret = register_chrdev(MAJOR_NUM, "globalvar", &globalvar_fops);
        if (ret)
        {
                printk("globalvar register success");
                init_rwsem(&rwsem);   //初始化一个互斥锁
                init_waitqueue_head(&outq); //初始化等待队列outq
                return 0;
        }
        return ret;
}

static void __exit globalvar_exit(void)
{
        unregister_chrdev(MAJOR_NUM, "globalvar");
}

//读设备
static ssize_t globalvar_read(struct file *filp, char *buf, size_t len, loff_t *off)
{
        //等待数据可获得
        if (wait_event_interruptible(outq, flag != 0)) // wait_event_interruptible把进程状态设为TASK_INTERRUPTIBLE，nonexclusive
        {
                return - ERESTARTSYS;
        }

        down_read(&rwsem); //获得信号量，相当于获得锁

        flag = 0;
        if (copy_to_user(buf, &global_var, sizeof(int)))
        {
                up_read(&rwsem);  //如果读取失败，则释放信号量，相当于释放锁
                return -EFAULT;
        }
        up_read(&rwsem); //若读取成功也释放信号量
        return sizeof(int);
}

//写设备
static ssize_t globalvar_write(struct file *filp, const char *buf, size_t len,loff_t *off)
{
        down_write(&rwsem); //获得信号量

        if (copy_from_user(&global_var, buf, sizeof(int)))
        {
                up_write(&rwsem); //写失败后释放锁
                return -EFAULT;
        }
        up_write(&rwsem); //写成功后，也释放信号量
        flag = 1;
        wake_up_interruptible(&outq); //唤醒等待进程，通知已经有数据可以读取
        return sizeof(int);
}

module_init(globalvar_init);
module_exit(globalvar_exit);
```



参考：

https://www.cnblogs.com/mengfanrong/p/4849064.html
