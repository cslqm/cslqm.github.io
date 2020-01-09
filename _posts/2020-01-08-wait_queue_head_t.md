---
layout: post
title: "等待队列 wait_queue_head_t"
subtitle: 'wait_queue_head_t'
author: "cslqm"
header-style: text
tags:
  - Linux
---

在开始讲 completions 机制之前，需要先学习一下等待队列。


# 等待队列 wait_queue_head_t

## 作用

等待队列在linux内核中有着举足轻重的作用，很多linux驱动都或多或少涉及到了等待队列。因此，对于linux内核及驱动开发者来说，掌握等待队列是必须课之一。 Linux内核的等待队列是以双循环链表为基础数据结构，与进程调度机制紧密结合，能够用于实现核心的异步事件通知机制。它有两种数据结构：等待队列头（wait_queue_head_t）和等待队列项（wait_queue_t）。等待队列头和等待队列项中都包含一个list_head类型的域作为”连接件”。它通过一个双链表和把等待task的头，和等待的进程列表链接起来。下面具体介绍。

在内核里面，等待队列是有很多用处的，尤其是在中断处理、进程同步、定时等场合。可以使用等待队列在实现阻塞进程的唤醒。它以队列为基础数据结构，与进程调度机制紧密结合，能够用于实现内核中的异步事件通知机制，同步对系统资源的访问等。

## 结构组成

``` c
struct __wait_queue_head {
  spinlock_t lock;
  struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```

| 成员 | 描述 |
| :-: |  :-: |
| lock |  在对task_list与操作的过程中，使用该锁实现对等待队列的互斥访问 |
| task_list | 双向循环链表，存放等待的进程 |

## 操作/常用函数

Linux-2.6提供如下关于等待队列的操作:
(1) 定义"等待队列头"
``` c
    wait_queue_head_t my_queue;
```

(2) 初始化"等待队列头"
```c
    init_waitqueue_head(&my_queue);
    定义和初始化的快捷方式:
    DECLARE_WAIT_QUEUE_HEAD(my_queue);  
```

(3) 定义等待队列
``` c
    DECLARE_WAITQUEUE(name, tsk);
    定义并初始化一个名为 name 的等待队列(wait_queue_t);
```

(4) 添加/移除等待队列
```c
    void fastcall add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
    void fastcall remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait);
    add_wait_queue()用于将等待队列wait添加到等待队列头q指向的等待队列链表中，而remove_wait_queue()用于将等待队列wait从附属的等待队列头q指向的等待队列链表中移除。
```

(5) 等待事件
``` c
    wait_event(queue, condition);
    wait_event_interruptible(queue, condition);
    wait_event_timeout(queue, condition, timeout);
    wait_event_interruptible_timeout(queue, condition, timeout);
    等待第一个参数queue作为等待队列头的等待队列被唤醒，而且第二个参数condition必须满足，否则阻塞。wait_event()和wait_event_interruptible()的区别在于后者可以被信号打断，而前者不能。加上timeout后的宏意味着阻塞等待的超时时间，以jiffy为单位，在第三个参数的timeout到达时，不论condition是否满足，均返回。
```

(6) 唤醒队列
``` c
    void wake_up(wait_queue_head_t *queue);
    void wake_up_interruptible(wait_queue_head_t *queue);
    上述操作会唤醒以queue作为等待队列头的所有等待队列对应的进程。
    wake_up()               <--->    wait_event()
                                     wait_event_timeout()
    wake_up_interruptible() <--->    wait_event_interruptible()  
                                     wait_event_interruptible_timeout()

    wake_up() 可以唤醒处于TASK_INTERRUPTIBLE和TASK_UNINTERRUPTIBLE的进程
    wake_up_interruptble() 只能唤醒处于TASK_INTERRUPTIBLE的进程。
```

(7) 在等待队列上睡眠
``` c
    sleep_on(wait_queue_head_t *q);
    interruptible_sleep_on(wait_queue_head_t *q);
 
    sleep_on()函数的作用就是将当前进程的状态置成TASK_UNINTERRUPTIBLE，定义一个等待队列，并把它添加到等待队列头q，直到支援获得，q引导的等待队列被唤醒。
    interruptible_sleep_on()与sleep_on()函数类似，其作用是将目前进程的状态置成TASK_INTERRUPTIBLE，并定义一个等待队列，之后把它附属到等待队列头q，直到资源可获得，q引导的等待队列被唤醒或者进程收到信号。  

    sleep_on()               <--->   wake_up()
    interruptible_sleep_on() <--->   wake_up_interruptible()
```

## 初始化

(1)

``` c
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);

#define init_waitqueue_head(q)				\
	do {						\
		static struct lock_class_key __key;	\
							\
		__init_waitqueue_head((q), &__key);	\
	} while (0)

void __init_waitqueue_head(wait_queue_head_t *q, struct lock_class_key *key)
{
	spin_lock_init(&q->lock);          //初始化自旋锁
	lockdep_set_class(&q->lock, key);  //初始化锁映射信息，具体代码看不懂，只知道可以检测是否发生死锁
	INIT_LIST_HEAD(&q->task_list);     //初始化链表
}
```
直接定义并初始化。init_waitqueue_head()函数会将自旋锁初始化为未锁，等待队列初始化为空的双向循环链表。

(2)

``` c
DECLARE_WAIT_QUEUE_HEAD(my_queue);

#define DECLARE_WAIT_QUEUE_HEAD(name) \
	wait_queue_head_t name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {				\
	.lock		= __SPIN_LOCK_UNLOCKED(name.lock),		\
	.task_list	= { &(name).task_list, &(name).task_list } }
```

定义并初始化，相当于(1)。

(3)定义等待队列：

``` c
DECLARE_WAITQUEUE(name,tsk);

#define DECLARE_WAITQUEUE(name, tsk)					\
	wait_queue_t name = __WAITQUEUE_INITIALIZER(name, tsk)

#define __WAITQUEUE_INITIALIZER(name, tsk) {				\
	.private	= tsk,						\
	.func		= default_wake_function,			\
	.task_list	= { NULL, NULL } }
```

注意此处是定义一个wait_queue_t类型的变量name，并将其private与设置为tsk。wait_queue_t类型定义如下

``` c
typedef struct __wait_queue wait_queue_t;

struct __wait_queue {
	unsigned int flags;
#define WQ_FLAG_EXCLUSIVE	0x01
	void *private;
	wait_queue_func_t func;
	struct list_head task_list;
};
```

其中 flags 域指明该等待的进程是互斥进程还是非互斥进程。其中 0 是非互斥进程，WQ_FLAG_EXCLUSIVE(0x01)是互斥进程。

等待队列(wait_queue_t)和等待对列头(wait_queue_head_t)的区别是等待队列是等待队列头的成员。也就是说等待队列头的 task_list 域链接的成员就是等待队列类型的(wait_queue_t)。

![等待队列](/img/in-post/等待队列.png)


## (从等待队列头中)添加／移出等待队列

add_wait_queue
``` c
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
	unsigned long flags;

	wait->flags &= ~WQ_FLAG_EXCLUSIVE;  // ~WQ_FLAG_EXCLUSIVE 位运算 表示非互斥
	spin_lock_irqsave(&q->lock, flags); // 需要修改队列成员，申请锁
	__add_wait_queue(q, wait);          // 添加到等待队列
	spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(add_wait_queue);

static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
	list_add(&new->task_list, &head->task_list);
}
```

设置等待的进程为非互斥进程，并将其添加进等待队列头(q)的队头中。

``` c
void add_wait_queue_exclusive(wait_queue_head_t *q, wait_queue_t *wait)
{
	unsigned long flags;

	wait->flags |= WQ_FLAG_EXCLUSIVE;   // 互斥进程 WQ_FLAG_EXCLUSIVE
	spin_lock_irqsave(&q->lock, flags); // 需要修改等待队列
	__add_wait_queue_tail(q, wait);     // 修改成员队列
	spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(add_wait_queue_exclusive);

static inline void __add_wait_queue_tail(wait_queue_head_t *head,
						wait_queue_t *new)
{
	list_add_tail(&new->task_list, &head->task_list);
}
```

该函数也和add_wait_queue()函数功能基本一样，只不过它是将等待的进程(wait)设置为互斥进程。

___


remove_wait_queue()函数

``` c
# linux/wait.h
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
	unsigned long flags;

	spin_lock_irqsave(&q->lock, flags);  // 需要修改队列，申请锁
	__remove_wait_queue(q, wait);        // 删除
	spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(remove_wait_queue);

static inline void __remove_wait_queue(wait_queue_head_t *head,
							wait_queue_t *old)
{
	list_del(&old->task_list);
}
```

在等待的资源或事件满足时，进程被唤醒，使用该函数被从等待头中删除。


## 等待事件

wait_event()宏

``` c
# linux/wait.h
/**
 * wait_event - sleep until a condition gets true
 * @wq: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_UNINTERRUPTIBLE) until the
 * @condition evaluates to true. The @condition is checked each time
 * the waitqueue @wq is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 */
#define wait_event(wq, condition) 					\
do {									\
	if (condition)	 						\
		break;							\
	__wait_event(wq, condition);					\
} while (0)

#define __wait_event(wq, condition) 					\
do {									\
	DEFINE_WAIT(__wait);						\
									\
	for (;;) {							\
		prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);	\
		if (condition)						\
			break;						\
		schedule();						\
	}								\
	finish_wait(&wq, &__wait);					\
} while (0)

// prepare_to_wait 修改当前进程状态为 wait，配置未不可中断
// if (condition)  跳出死循环条件是否满足
// schedule 开始睡眠
// finish_wait 修改当前进程状态为 TASK_RUNNING，将过去等待的事件从队列中移除

# kernel/wait.h
/*
 * Note: we use "set_current_state()" _after_ the wait-queue add,
 * because we need a memory barrier there on SMP, so that any
 * wake-function that tests for the wait-queue being active
 * will be guaranteed to see waitqueue addition _or_ subsequent
 * tests in this thread will see the wakeup having taken place.
 *
 * The spin_unlock() itself is semi-permeable and only protects
 * one way (it only protects stuff inside the critical region and
 * stops them from bleeding out - it would still allow subsequent
 * loads to move into the critical region).
 */
void
prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
{
	unsigned long flags;

	wait->flags &= ~WQ_FLAG_EXCLUSIVE;  // 非互斥
	spin_lock_irqsave(&q->lock, flags); // 需要修改队列
	if (list_empty(&wait->task_list))   // 等待队列为空 = 如果要修改的实际不存在于队列中
		__add_wait_queue(q, wait);      // 添加事件
	set_current_state(state);           // 修改状态
	spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(prepare_to_wait);

static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
	list_add(&new->task_list, &head->task_list);
}

# linux/sched.h
#define set_current_state(state_value)		\
	set_mb(current->state, (state_value))

# 
#define set_mb(var, value) do { var = value; barrier(); } while (0)
```

在等待会列中睡眠直到condition为真。在等待的期间，进程会被置为TASK_UNINTERRUPTIBLE进入睡眠，直到condition变量变为真。每次进程被唤醒的时候都会检查condition的值.

___

wait_event_interruptible()函数

``` c
/**
 * wait_event_interruptible - sleep until a condition gets true
 * @wq: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_INTERRUPTIBLE) until the
 * @condition evaluates to true or a signal is received.
 * The @condition is checked each time the waitqueue @wq is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 *
 * The function will return -ERESTARTSYS if it was interrupted by a
 * signal and 0 if @condition evaluated to true.
 */
#define wait_event_interruptible(wq, condition)				\
({									\
	int __ret = 0;							\
	if (!(condition))						\
		__wait_event_interruptible(wq, condition, __ret);	\
	__ret;								\
})

#define __wait_event_interruptible(wq, condition, ret)			\
do {									\
	DEFINE_WAIT(__wait);						\
									\
	for (;;) {							\
		prepare_to_wait(&wq, &__wait, TASK_INTERRUPTIBLE);	\
		if (condition)						\
			break;						\
		if (!signal_pending(current)) {				\
			schedule();					\
			continue;					\
		}							\
		ret = -ERESTARTSYS;					\
		break;							\
	}								\
	finish_wait(&wq, &__wait);					\
} while (0)
```

和wait_event()的区别是调用该宏在等待的过程中当前进程会被设置为TASK_INTERRUPTIBLE状态.在每次被唤醒的时候,首先检查condition是否为真,如果为真则返回,否则检查如果进程是被信号唤醒,会返回-ERESTARTSYS错误码.如果是condition为真,则返回0.

___

wait_event_timeout()宏

``` c
#define wait_event_timeout(wq, condition, timeout)			\
({									\
	long __ret = timeout;						\
	if (!(condition)) 						\
		__wait_event_timeout(wq, condition, __ret);		\
	__ret;								\
})

#define __wait_event_timeout(wq, condition, ret)			\
do {									\
	DEFINE_WAIT(__wait);						\
									\
	for (;;) {							\
		prepare_to_wait(&wq, &__wait, TASK_UNINTERRUPTIBLE);	\
		if (condition)						\
			break;						\
		ret = schedule_timeout(ret);				\
		if (!ret)						\
			break;						\
	}								\
	finish_wait(&wq, &__wait);					\
} while (0)
```

也与wait_event()类似.不过如果所给的睡眠时间为负数则立即返回.如果在睡眠期间被唤醒,且condition为真则返回剩余的睡眠时间,否则继续睡眠直到到达或超过给定的睡眠时间,然后返回0.

___

wait_event_interruptible_timeout()宏

``` c
#define wait_event_interruptible_timeout(wq, condition, timeout)	\
({									\
	long __ret = timeout;						\
	if (!(condition))						\
		__wait_event_interruptible_timeout(wq, condition, __ret); \
	__ret;								\
})

#define __wait_event_interruptible_timeout(wq, condition, ret)		\
do {									\
	DEFINE_WAIT(__wait);						\
									\
	for (;;) {							\
		prepare_to_wait(&wq, &__wait, TASK_INTERRUPTIBLE);	\
		if (condition)						\
			break;						\
		if (!signal_pending(current)) {				\
			ret = schedule_timeout(ret);			\
			if (!ret)					\
				break;					\
			continue;					\
		}							\
		ret = -ERESTARTSYS;					\
		break;							\
	}								\
	finish_wait(&wq, &__wait);					\
} while (0)
```

与wait_event_timeout()类似,不过如果在睡眠期间被信号打断则返回ERESTARTSYS错误码.

___

wait_event_interruptible_exclusive()宏

``` c
#define wait_event_interruptible_exclusive(wq, condition)		\
({									\
	int __ret = 0;							\
	if (!(condition))						\
		__wait_event_interruptible_exclusive(wq, condition, __ret);\
	__ret;								\
})

#define __wait_event_interruptible_exclusive(wq, condition, ret)	\
do {									\
	DEFINE_WAIT(__wait);						\
									\
	for (;;) {							\
		prepare_to_wait_exclusive(&wq, &__wait,			\
					TASK_INTERRUPTIBLE);		\
		if (condition) {					\
			finish_wait(&wq, &__wait);			\
			break;						\
		}							\
		if (!signal_pending(current)) {				\
			schedule();					\
			continue;					\
		}							\
		ret = -ERESTARTSYS;					\
		abort_exclusive_wait(&wq, &__wait, 			\
				TASK_INTERRUPTIBLE, NULL);		\
		break;							\
	}								\
} while (0)
```

同样和wait_event_interruptible()一样,不过该睡眠的进程是一个互斥进程.

## 唤醒队列

wake_up()函数

``` c
#define wake_up(x)			__wake_up(x, TASK_NORMAL, 1, NULL)

/**
 * __wake_up - wake up threads blocked on a waitqueue.
 * @q: the waitqueue
 * @mode: which threads
 * @nr_exclusive: how many wake-one or wake-many threads to wake up
 * @key: is directly passed to the wakeup function
 *
 * It may be assumed that this function implies a write memory barrier before
 * changing the task state if and only if any tasks are woken up.
 */
void __wake_up(wait_queue_head_t *q, unsigned int mode,
			int nr_exclusive, void *key)
{
	unsigned long flags;

	spin_lock_irqsave(&q->lock, flags);
	__wake_up_common(q, mode, nr_exclusive, 0, key);
	spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL(__wake_up);

/*
 * The core wakeup function. Non-exclusive wakeups (nr_exclusive == 0) just
 * wake everything up. If it's an exclusive wakeup (nr_exclusive == small +ve
 * number) then we wake all the non-exclusive tasks and one exclusive task.
 *
 * There are circumstances in which we can try to wake a task which has already
 * started to run but is not in state TASK_RUNNING. try_to_wake_up() returns
 * zero in this (rare) case, and we handle it by continuing to scan the queue.
 */
static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
			int nr_exclusive, int wake_flags, void *key)
{
	wait_queue_t *curr, *next;

	list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
		unsigned flags = curr->flags;

		if (curr->func(curr, mode, wake_flags, key) &&
				(flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
			break;
	}
}
```

唤醒等待队列.可唤醒处于 TASK_INTERRUPTIBLE 和 TASK_UNINTERUPTIBLE 状态的进程,和wait_event/wait_event_timeout成对使用.

被唤醒的进程，都会检查自己等待的条件是否满足，满足的进程会修改自己的状态为 running。如果条件不满足会继续睡眠。等待下次唤醒（睡眠的进程可能支持可中断，所以并发所有的唤醒都是由类似函数唤醒）。

___

wake_up_interruptible()函数:

``` c
#define wake_up_interruptible(x)	__wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
```

和wake_up()唯一的区别是它只能唤醒TASK_INTERRUPTIBLE状态的进程.,与wait_event_interruptible/wait_event_interruptible_timeout/ wait_event_interruptible_exclusive成对使用. 

___

``` c
#define wake_up_all(x)			__wake_up(x, TASK_NORMAL, 0, NULL)

#define wake_up_interruptible_nr(x, nr)	__wake_up(x, TASK_INTERRUPTIBLE, nr, NULL)
#define wake_up_interruptible_all(x)	__wake_up(x, TASK_INTERRUPTIBLE, 0, NULL)
```

这些也基本都和wake_up/wake_up_interruptible一样.

## 在等待队列上睡眠

sleep_on函数

``` c
void __sched sleep_on(wait_queue_head_t *q)
{
	sleep_on_common(q, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
EXPORT_SYMBOL(sleep_on);

static long __sched
sleep_on_common(wait_queue_head_t *q, int state, long timeout)
{
	unsigned long flags;
	wait_queue_t wait;

	init_waitqueue_entry(&wait, current);

	__set_current_state(state);

	spin_lock_irqsave(&q->lock, flags);
	__add_wait_queue(q, &wait);
	spin_unlock(&q->lock);
	timeout = schedule_timeout(timeout);
	spin_lock_irq(&q->lock);
	__remove_wait_queue(q, &wait);
	spin_unlock_irqrestore(&q->lock, flags);

	return timeout;
}

```

该函数的作用是定义一个等待队列(wait)，并将当前进程添加到等待队列中(wait)，然后将当前进程的状态置为TASK_UNINTERRUPTIBLE，并将等待队列(wait)添加到等待队列头(q)中。之后就被挂起直到资源可以获取，才被从等待队列头(q)中唤醒，从等待队列头中移出。在被挂起等待资源期间，该进程不能被信号唤醒。

___

sleep_on_timeout()函数

``` c
long __sched sleep_on_timeout(wait_queue_head_t *q, long timeout)
{
	return sleep_on_common(q, TASK_UNINTERRUPTIBLE, timeout);
}
EXPORT_SYMBOL(sleep_on_timeout);
```

与sleep_on()函数的区别在于调用该函数时，如果在指定的时间内(timeout)没有获得等待的资源就会返回。实际上是调用schedule_timeout()函数实现的。值得注意的是如果所给的睡眠时间(timeout)小于0，则不会睡眠。该函数返回的是真正的睡眠时间。

___

interruptible_sleep_on()函数

``` c
void __sched interruptible_sleep_on(wait_queue_head_t *q)
{
	sleep_on_common(q, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
EXPORT_SYMBOL(interruptible_sleep_on);
```

该函数和sleep_on()函数唯一的区别是将当前进程的状态置为TASK_INTERRUPTINLE，这意味在睡眠如果该进程收到信号则会被唤醒。

___

interruptible_sleep_on_timeout()函数

``` c
long __sched
interruptible_sleep_on_timeout(wait_queue_head_t *q, long timeout)
{
	return sleep_on_common(q, TASK_INTERRUPTIBLE, timeout);
}
EXPORT_SYMBOL(interruptible_sleep_on_timeout);
```
类似于sleep_on_timeout()函数。进程在睡眠中可能在等待的时间没有到达就被信号打断而被唤醒，也可能是等待的时间到达而被唤醒。

以上四个函数都是让进程在等待队列上睡眠，不过是小有诧异而已。在实际用的过程中，根据需要选择合适的函数使用就是了。例如在对软驱数据的读写中，如果设备没有就绪则调用sleep_on()函数睡眠直到数据可读(可写)，在打开串口的时候，如果串口端口处于关闭状态则调用interruptible_sleep_on()函数尝试等待其打开。在声卡驱动中，读取声音数据时，如果没有数据可读，就会等待足够常的时间直到可读取。


## 例子

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