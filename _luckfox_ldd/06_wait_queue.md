---
layout: default
title: "06 Wait queue"
parent: Linux device driver on Luckfox
nav_order: 6
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
nav_enabled: false
---
## 06 Waitqueue
We could have read and write to the driver already, but the CPU will not happy if we do read and write continously and not reasonally. Sometimes the device buffer is full and not ready to accept data or it is processing data and not ready to send. In such cases, driver should (by default) ``block`` the process,
putting it to ``sleep`` until the request can proceed.

There are several ways of handling sleeping and waking up in Linux, each suited to different needs. Waitqueue is also one of the methods to handle that case.

The waitqueue is a queue that its elements is waitting :). 

To make a process sleep, just put it into waitqueue. However, a couple of rules that you must keep in mind to be able to code sleeps in a safe manner(avoiding race condition):
+ Never sleep when you are running in an atomic context.

{: .note }
An atomic context is simply a state where multiple steps must be performed without any sort of concurrent access. What that means, with regard to sleeping, is that your driver cannot sleep while holding a spinlock, seqlock, or RCU lock.

+ Things have changed while process is sleeping, so we have to check our condition after wake up.

There are 3 important steps in Waitqueue:
1. Initializing Waitqueue
2. Queuing (Put the Task to sleep until the event comes)
3.  Waking Up Queued Task

Waitqueue is defined in ``<linux/wait.h>`` with a structure of
type ``wait_queue_head_t``. A wait queue head can
be defined and initialized statically or dynamically:
+ Statically:
```c
DECLARE_WAIT_QUEUE_HEAD(name);
```
+ Dynamically:
```c
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
```

There are several macros are used to put process to waitqueue:
+ ``wait_event``
+ ``wait_event_timeout``
+ ``wait_event_cmd``
+ ``wait_event_interruptible``
+ ``wait_event_interruptible_timeout``
+ ``wait_event_killable``

### ``wait_event``
Sleep until a condition gets true.
```c
wait_event(wq, condition);
```
+ ``wq`` – the waitqueue to wait on
+ ``condition`` – a C expression for the event to wait for

Processes are put by wait_event is ``TASK_UNINTERRUPTIBLE``. 

### ``wait_event_timeout``
Sleep until a condition gets true or a timeout elapses
```c
wait_event_timeout(wq, condition, timeout);
```
+ ``wq`` – the waitqueue to wait on
+ ``condition`` – a C expression for the event to wait for
+ ``timeout`` –  timeout, in jiffies (same as tick in RTOSes)

Processes are put by wait_event is ``TASK_UNINTERRUPTIBLE``. 

If the timeout expires before the condition is met, the process is woken up regardless of the condition's state. The ``return value`` indicates whether the condition was met or if the timeout occurred.

``Return Value``

The macro returns the number of jiffies remaining before the timeout expires. If the condition was met before the timeout, it returns a positive value (the remaining time). If the timeout expired, it returns 0.

### ``wait_event_cmd``
Sleep until a condition gets true
```c
wait_event_cmd(wq, condition, cmd1, cmd2);
```
+ ``wq`` –  the waitqueue to wait on
+ ``condtion`` – a C expression for the event to wait for
+ ``cmd1`` – the command will be executed before sleep
+ ``cmd2`` – the command will be executed after sleep

Processes are put by wait_event is ``TASK_UNINTERRUPTIBLE``. 

### ``wait_event_interruptible``
Sleep until a condition gets true, with the added capability of allowing the wait to be interrupted by signals.

This is particularly useful in scenarios where you want to wait for an event but also want to ensure that the process can be woken up by user-space signals (like ``SIGINT`` or ``SIGTERM``).
```c
wait_event_interruptible(wq, condition);
```
+ ``wq`` –  the waitqueue to wait on
+ ``condtion`` – a C expression for the event to wait for

Processes are put by wait_event is ``TASK_INTERRUPTIBLE``. 

The ``return value`` will indicate that the wait was interrupted by a signal.

``Return Value``
+ The macro returns 0 if the condition evaluated to true and the process was woken up normally.
+ If the wait was interrupted by a signal, it returns ``-ERESTARTSYS``, which indicates that the system call was interrupted and should be restarted.

### ``wait_event_interruptible_timeout``
Sleep until a condition gets true or a timeout elapses, with the added capability of being interrupted by signals.
```c
wait_event_interruptible_timeout(wq, condition, timeout);
```
+ ``wq`` – the waitqueue to wait on
+ ``condition`` – a C expression for the event to wait for
+ ``timeout`` –  timeout, in jiffies (same as tick in RTOSes)

Processes are put by wait_event is ``TASK_INTERRUPTIBLE``. 

The ``return value`` indicates whether the condition was met or if the timeout occurred.

``Return Value``
+ The macro returns the number of jiffies remaining before the timeout expires. If the condition was met before the timeout, it returns a positive value (the remaining time). 
+ If the timeout expired, it returns 0.
+ If the wait was interrupted by a signal, it returns ``-ERESTARTSYS``, indicating that the system call was interrupted and should be restarted.

### ``wait_event_killable``
Sleep until a condition gets true. It also allow the wait to be interrupted by signals. This is similar to wait_event_interruptible, but it is specifically designed to handle situations where you want to ensure that the process can be killed (terminated) if a signal is received.
```c 
wait_event_killable(wq, condition);
```
+ ``wq`` – the waitqueue to wait on
+ ``condition`` – a C expression for the event to wait for

Processes are put by wait_event is ``TASK_KILLABLE``. 

If the signal that interrupts the wait is a termination signal (like ``SIGKILL`` or ``SIGTERM``), the process will be terminated by the kernel.

The return value will indicate that the wait was interrupted by a signal.

``Return Value``
+ The macro returns 0 if the condition was met and the process was woken up normally.
+ If the wait was interrupted by a signal, it returns -ERESTARTSYS, which indicates that the system call was interrupted and should be restarted.

To wake processes in waitqueue, we can use the below function:
+ ``wake_up``
+ ``wake_up_all``
+ ``wake_up_interruptible``
+ ``wake_up_sync`` and ``wake_up_interruptible_sync``

### ``wake_up``
Wakes up only sleeping processes which are in non-interruptible sleep.

```c
wake_up(&wq);
```
+ ``wq`` – the waitqueue to wake up

### ``wake_up_all``
Wakes up all sleeping processes, regardless of whether they are in interruptible or non-interruptible sleep.

```c
wake_up_all(&wq);
```
+ ``wq`` – the waitqueue to wake up

### ``wake_up_interruptible``
Wakes up only sleeping processes which are in interruptible sleep.

```c
wake_up_interruptible(&wq);
```
+ ``wq`` – the waitqueue to wake up

### ``wake_up_sync`` and ``wake_up_interruptible_sync``
Normally, when calling ``wake_up`` can cause an immediate reschedule to happen, which could result in context switching to a different process before the current process (the one that called ``wake_up``) continues execution. It may not make sense to perform a context switch because the current process is about to sleep anyway. So calling ``wake_up_sync`` and ``wake_up_interruptible_sync`` guarantees that current process runs.

```c
wake_up_sync(&wq);
```
```c
wake_up_interruptible_sync(&wq);
```
+ ``wq`` – the waitqueue to wake up

{: .note }

The awakened processes could run immediately on a different processor, so these functions should not be expected to provide mutual exclusion. To avoid this situation, additional synchronization mechanisms (like spinlocks or mutexes) should be used.

## Our code
The code located in folder 06_wait_queue
```bash
# To build ko driver 
make build-06
# To clean ko driver 
make clean-06
# To upload ko driver to our board 
make upload-06
```
Result
```
[root@luckfox root]# insmod wq.ko
[root@luckfox root]# dmesg | tail -5
[27577.170630] 8
[27633.778206] Wq driver initializing...
[27633.782324] Thread Created successfully
[27633.782349] Wq module inserted successfully.
[27633.784268] Waiting For Event...
[root@luckfox root]# ls /dev/ | grep wq
wq_dev
[root@luckfox root]# cat /dev/wq_dev
[root@luckfox root]# cat /dev/wq_dev
[root@luckfox root]# cat /dev/wq_dev
[root@luckfox root]# cat /dev/wq_dev
[root@luckfox root]# dmesg | tail -5
[27862.400515] Wq File Opened...!!!
[27862.400627] Wq File Read
[27862.400648] Event Came From Read Function - 4
[27862.400657] Waiting For Event...
[27862.400672] Wq File Closed...!!!
```
I have created a thread, and put the thread into wait queue, every time we read the char device, it will unblock that thread with ``wait_event_interruptible``, increase the ``read_count``, and then reset the ``condition`` to 0. 