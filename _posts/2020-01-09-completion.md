---
layout: post
title: "并发：completion 机制"
subtitle: 'completion'
author: "cslqm"
header-style: text
tags:
  - Linux
---

之前了解了 rw_sem 和等待队列，本篇继续按照 ldd 的顺序讲一下 completion。

	所有源代码来自 2.6.32.27

## completions 基本场景

在内核编程中常有这样的场景，在当前线程中创建一个线程，并且等待它完成之后再继续执行。通常可以用信号量来解决它，也可以用completion机制来解决。

为什么用completions ，它比信号量好在哪？
1. 使用completion比使用信号量简单。
2. 使用completion可以一次性唤醒所有等待进程，而用信号量会比较麻烦。

	The basic summary is that we had this (fairly common) way of waiting for certain events by having a locked semaphore on the stack of the waiter, and then having the waiter do a "down()" which caused it to block until the thing it was waiting for did an "up()".
	This works fairly well, but it has a really small (and quite unlikely) race on SMP, that is not so much a race of the idea itself, as of the implementation of the semaphores. We could have fixed the semaphores, but there were a few reasons not to:
	the semaphores are optimized (on purpose) for the non-contention case. The "wait for completion" usage has the opposite default case the semaphores are quite involved and architecture-specific, exactly due to this optimization. Trying to change them is painful as hell.


内核编程中常见的一种模式是，在当前线程之外初始化某个活动，然后等待该活动的结束。

这个活动可能是，创建一个新的内核线程或者新的用户空间进程、对一个已有进程的某个请求，或者某种类型的硬件动作，等等。

在这种情况下，我们可以使用信号量来同步这两个任务。然而，内核中提供了另外一种机制——completion接口。

Completion是一种轻量级的机制，他允许一个线程告诉另一个线程某个工作已经完成。

## completion 结构

``` c
# linux/completion.h
/**
 * struct completion - structure used to maintain state for a "completion"
 *
 * This is the opaque structure used to maintain the state for a "completion".
 * Completions currently use a FIFO to queue threads that have to wait for
 * the "completion" event.
 *
 * See also:  complete(), wait_for_completion() (and friends _timeout,
 * _interruptible, _interruptible_timeout, and _killable), init_completion(),
 * and macros DECLARE_COMPLETION(), DECLARE_COMPLETION_ONSTACK(), and
 * INIT_COMPLETION().
 */
struct completion {
	unsigned int done;
	wait_queue_head_t wait;
};
```

1. done 是用于同步的原子量
2. wait 是等待队列。

## completion 初始化
和信号量一样，初始化分为静态初始化和动态初始化两种情况。

### 静态初始化

DECLARE_COMPLETION

``` c
# linux/completion.h
#define DECLARE_COMPLETION(work) \
	struct completion work = COMPLETION_INITIALIZER(work)

#define COMPLETION_INITIALIZER(work) \
	{ 0, __WAIT_QUEUE_HEAD_INITIALIZER((work).wait) }

# linux/wait.h
#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {				\
	.lock		= __SPIN_LOCK_UNLOCKED(name.lock),		\
	.task_list	= { &(name).task_list, &(name).task_list } }
```

### 动态初始化

init_completion

``` c
struct  completion my_comp;

init_waitqueue_head(&my_comp);
```

## 函数列表

关于 completion 常用的函数基本如下。

``` c
extern void wait_for_completion(struct completion *);
extern int wait_for_completion_interruptible(struct completion *x);
extern int wait_for_completion_killable(struct completion *x);
extern unsigned long wait_for_completion_timeout(struct completion *x,
						   unsigned long timeout);
extern unsigned long wait_for_completion_interruptible_timeout(
			struct completion *x, unsigned long timeout);
extern bool try_wait_for_completion(struct completion *x);
extern bool completion_done(struct completion *x);

extern void complete(struct completion *);
extern void complete_all(struct completion *);
```

## 睡眠等待

wait_for_completion

``` c
void __sched wait_for_completion(struct completion *x)
{
	wait_for_common(x, MAX_SCHEDULE_TIMEOUT, TASK_UNINTERRUPTIBLE);
}
EXPORT_SYMBOL(wait_for_completion);

static long __sched
wait_for_common(struct completion *x, long timeout, int state)
{
	might_sleep();

	spin_lock_irq(&x->wait.lock);  // 修改等待队列，申请锁
	timeout = do_wait_for_common(x, timeout, state);  // 调用 do_wait_for_common
	spin_unlock_irq(&x->wait.lock);
	return timeout;
}

static inline long __sched
do_wait_for_common(struct completion *x, long timeout, int state)
{
	if (!x->done) {
		DECLARE_WAITQUEUE(wait, current);     // 初始化队列

		wait.flags |= WQ_FLAG_EXCLUSIVE;
		__add_wait_queue_tail(&x->wait, &wait);    //将事件加入等待队列
		do {
			if (signal_pending_state(state, current)) {
				timeout = -ERESTARTSYS;
				break;
			}
			__set_current_state(state);           //设置当前进程状态
			spin_unlock_irq(&x->wait.lock);       //需要调度其他程序，解锁
			timeout = schedule_timeout(timeout);  //开始睡眠
			spin_lock_irq(&x->wait.lock);
		} while (!x->done && timeout);            //是否可以结束循环
		__remove_wait_queue(&x->wait, &wait);     //进程被唤醒，所以删除队列中的记录
		if (!x->done)
			return timeout;
	}
	x->done--;
	return timeout ?: 1;
}
```

___

wait_for_completion_interruptible

``` c
int __sched wait_for_completion_interruptible(struct completion *x)
{
	long t = wait_for_common(x, MAX_SCHEDULE_TIMEOUT, TASK_INTERRUPTIBLE);
	if (t == -ERESTARTSYS)
		return t;
	return 0;
}
EXPORT_SYMBOL(wait_for_completion_interruptible);
```

同 wait_for_completion 基本一样，可中断。

___


wait_for_completion_killable

``` c
int __sched wait_for_completion_killable(struct completion *x)
{
	long t = wait_for_common(x, MAX_SCHEDULE_TIMEOUT, TASK_KILLABLE);
	if (t == -ERESTARTSYS)
		return t;
	return 0;
}
EXPORT_SYMBOL(wait_for_completion_killable);
```

___


wait_for_completion_timeout

``` c
unsigned long __sched
wait_for_completion_timeout(struct completion *x, unsigned long timeout)
{
	return wait_for_common(x, timeout, TASK_UNINTERRUPTIBLE);
}
EXPORT_SYMBOL(wait_for_completion_timeout);
```

___

wait_for_completion_interruptible_timeout

``` c
unsigned long __sched
wait_for_completion_interruptible_timeout(struct completion *x,
					  unsigned long timeout)
{
	return wait_for_common(x, timeout, TASK_INTERRUPTIBLE);
}
EXPORT_SYMBOL(wait_for_completion_interruptible_timeout);
```

___

try_wait_for_completion

``` c
bool try_wait_for_completion(struct completion *x)
{
	unsigned long flags;
	int ret = 1;

	spin_lock_irqsave(&x->wait.lock, flags);
	if (!x->done)
		ret = 0;
	else
		x->done--;
	spin_unlock_irqrestore(&x->wait.lock, flags);
	return ret;
}
EXPORT_SYMBOL(try_wait_for_completion);
```

## 唤醒睡眠

``` c
extern bool completion_done(struct completion *x);

extern void complete(struct completion *);
extern void complete_all(struct completion *);
```

``` c
bool completion_done(struct completion *x)
{
	unsigned long flags;
	int ret = 1;

	spin_lock_irqsave(&x->wait.lock, flags); // 此时加锁放置有其他进程修改等待队列
	if (!x->done)
		ret = 0;
	spin_unlock_irqrestore(&x->wait.lock, flags);
	return ret;
}
EXPORT_SYMBOL(completion_done);
```

___


complete 唤醒一个等待进程

``` c
void complete(struct completion *x)
{
	unsigned long flags;

	spin_lock_irqsave(&x->wait.lock, flags);  // 此时加锁放置有其他进程修改等待队列
	x->done++;
	__wake_up_common(&x->wait, TASK_NORMAL, 1, 0, NULL);
	spin_unlock_irqrestore(&x->wait.lock, flags);
}
EXPORT_SYMBOL(complete);

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

``` c
typedef struct __wait_queue wait_queue_t;

struct __wait_queue {
    // 激活后是否继续激活下一个entry。候选值为WQ_FLAG_EXCLUSIVE。一般设置为0。
    // 当等待队列所有entry的flags==0时，等待队列所有entry都会被激活。所以就会有惊群现象。
	unsigned int flags;
#define WQ_FLAG_EXCLUSIVE	0x01
	void *private;        
	wait_queue_func_t func;   // 函数指针 唤醒函数，默认为 try_to_wake_up
	struct list_head task_list;
};
```

___

complete_all 唤醒所有等待进程

``` c
void complete_all(struct completion *x)
{
	unsigned long flags;

	spin_lock_irqsave(&x->wait.lock, flags); // 此时加锁放置有其他进程修改等待队列
	x->done += UINT_MAX/2;
	__wake_up_common(&x->wait, TASK_NORMAL, 0, 0, NULL);
	spin_unlock_irqrestore(&x->wait.lock, flags);
}
EXPORT_SYMBOL(complete_all);
```

由于completion的实现方式，即使complete在wait_for_competion之前调用，也可以正常工作。


## 例子

ldd 上的例子

``` c
/*                                                     
 * $Id: hellop.c,v 1.4 2004/09/26 07:02:43 gregkh Exp $ 
 */                                                    
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/sched.h>
#include <linux/kernel.h> /* printk() */
#include <linux/types.h>  /* size_t */
#include <linux/completion.h>

MODULE_LICENSE("Dual BSD/GPL");

/*                                                        
 * These lines, although not shown in the book,           
 * are needed to make hello.c run properly even when      
 * your kernel has version support enabled                
 */                                                       

static int hello_major = 0;

DECLARE_COMPLETION(comp);

ssize_t hello_read (struct file *filp, char __user *buf, size_t count, loff_t *pos)
{
        printk(KERN_DEBUG "process %i (%s) going to sleep\n",
                        current->pid, current->comm);
        wait_for_completion(&comp);
        printk(KERN_DEBUG "awoken %i (%s)\n", current->pid, current->comm);
        return 0; /* EOF */
}

ssize_t hello_write (struct file *filp, const char __user *buf, size_t count,
                loff_t *pos)
{
        printk(KERN_DEBUG "process %i (%s) awakening the readers...\n",
                        current->pid, current->comm);
        complete(&comp);
        return count; /* succeed, to avoid retrial */
}


struct file_operations hello_fops = {
        .owner = THIS_MODULE,
        .read =  hello_read,
        .write = hello_write,
};


int hello_init(void)
{
        int result;

        /*
         * Register your major, and accept a dynamic number
         */
        result = register_chrdev(hello_major, "hello", &hello_fops);
        if (result < 0)
                return result;
        if (hello_major == 0)
                hello_major = result; /* dynamic */
        return 0;
}

void hello_cleanup(void)
{
        unregister_chrdev(hello_major, "hello");
}

module_init(hello_init);
module_exit(hello_cleanup);
```
1. insmod ./hello.ko
2. cat /proc/hello | grep hello 获取编号
3. mknod /dev/hello c 编号 1
4. cat /dev/hello &
5. echo "ok" >> /dev/hello

