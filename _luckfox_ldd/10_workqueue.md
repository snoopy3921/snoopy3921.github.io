---
layout: default
title: "10 workqueue"
parent: Linux device driver on Luckfox
nav_order: 10
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 10 Workqueue
A small recap about interrupt IRQ, when handling an interrupt, if the job that need to be handled seems big, then Linux kernel divides into 2 part to handle that job, called ``Top halves`` and ``Bottom halves``. And we have walked through how to handle top halves with gpio driver using input interrupt. Now we will talk about how to handle bottom halves with workqueue.

Workqueue is one of mechanisms of bottom halves.

We may heard about tasklet, but it is marked as deprecation. Because, they have some limitations, such as not being able to sleep and being less flexible compared to other mechanisms. That covers some points about workqueue:
+ ``Process Context``: Workqueues run in the context of a kernel thread, which means they can sleep and perform operations that require blocking, such as memory allocation or I/O operations.
+ ``Schedulable``: You can schedule work to be done later, allowing interrupt handlers to return quickly and improving system responsiveness.
+ ``Concurrency``: Workqueues can be configured to run on multiple threads, allowing for parallel execution of tasks. This can improve performance, especially on multi-core systems.

There are two ways to implement Workqueue in the Linux kernel.

+ Using global workqueue 
+ Creating Own workqueue

## Dive into workqueue mechanism


<a href="https://www.cnblogs.com/LoyenWang/p/13185451.html">https://www.cnblogs.com/LoyenWang/p/13185451.html</a>

<a href="https://habr.com/en/companies/embox/articles/244155/">https://habr.com/en/companies/embox/articles/244155/</a>

<a href="https://litux.nl/mirror/kerneldevelopment/0672327201/ch07lev1sec4.html">https://litux.nl/mirror/kerneldevelopment/0672327201/ch07lev1sec4.html</a>


### Workqueue key components
+ ``Work (or work item)`` is the struct that holds the pointer to the function to be executed asynchronously. Defined as ``struct work_struct``.
+ ``Work queue``: a queue of work items. Defined as ``struct workqueue_struct``.
+ ``Worker``: is the struct that holds a special-purpose thread which execute the functions from the work queue, one after the other. Or we can say ``Worker`` processes ``Works(or work items)``. But in general there is no strict, sequential implementation: after all, here there is suppression, sleep, waiting, etc. Defined as ``struct worker``

![Worker thread](/assets/images/worker_thread.png)

+ ``Worker pool``: is a group of ``Worker``, it also holds shared resources, provides ``Workers`` abilities to process different ``Work item``. Defined as ``struct worker_pool``.There are 2 types of ``worker pool``:
  + Normal worker pool, used for general works.
  + Unbound worker pool, used for works type WQ_UNBOUND.
+ ``Pool workqueue``: is a "bridge" connects ``workqueue`` and ``worker_pool``. Defined as ``struct pool_workqueue``. 

Here is a pic about workqueue subsystem:

![Workqueue subsystem](/assets/images/workqueue_subsystem.png)

There are 2 types of workqueue:
+ ``Bind``: That workqueue created will be bound to specific CPU core(processor) to run. 
+ ``Unbound``: The work queue is not bound to a processor. The flag must be specified when creating the WQ_UNBOUND work queue. Core threads(jobs) can move between CPU cores(processors).

By default, the kernel creates several work queues (users can also create them):
1. ``system_wq``: If ``work item`` execution time is short, use this queue. This is also known as ``global_workqueue``.
2. ``system_highpri_mq``: High priority job queue.
3. ``system_long_wq``: If ``work item`` execution time is long, use this queue.
4. ``system_unbound_wq``: The core threads of this work queue are not tied to a specific processor(CPU).
5. ``system_freezable_wq``: This work queue is used to freeze when pausing ``work item``.
6. ``system_power_efficient_wq``: This work queue is chosen to sacrifice performance for saving energy.
7. ``system_freezable_power_efficient_wq``: Combination of ``system_freezable_wq`` and ``system_power_efficient_wq``.

### Workqueue creation

```c
DECLARE(_DELAYED)_WORK(name, void (*function)(struct work_struct *work)); /* Statically at linking time*/
INIT(_DELAYED)_WORK(_work, _func);                                        /* Dynamically at running time*/
PREPARE(_DELAYED)_WORK(_work, _func);                                     /* Change the function being executed*/
```

+ ``name``: The name of the “work_struct” structure that has to be created.
+ ``function``: The function to be scheduled in this workqueue.

And then you can add ``work item`` to specific ``workqueue``(``system_wq``, ``system_highpri_mq``, ...).
```c
bool queue_work(struct workqueue_struct *wq,   struct work_struct *work);
bool queue_delayed_work(struct workqueue_struct *wq, struct delayed_work *dwork, unsigned long delay); /* Add workitem to queue after delay time */
```

#### **schedule_work**
Put work item to global workqueue(``system_wq``)
```c
bool schedule_work( struct work_struct *work );
```
+ ``work``: ``work item``(job) to be done
+ ``Return``: zero if work was already on the kernel-global workqueue and non-zero otherwise.

#### **schedule_delayed_work**
Put work item to global workqueue(``system_wq``) after delay time.
```c
bool schedule_delayed_work( struct delayed_work *dwork, unsigned long delay );
```
+ ``dwork``: delay-``work item``(delay job) to be done
+ ``delay``: number of jiffies to wait or 0 for immediate execution
+ ``Return``: zero if work was already on the kernel-global workqueue and non-zero otherwise.

#### **schedule_work_on**
Put work item to global workqueue(``system_wq``) on a specific CPU.
```c
bool schedule_work_on( int cpu, struct work_struct *work );
```
+ ``cpu``: CPU to put the work task on
+ ``work``: ``work item``(job) to be done

#### **schedule_delayed_work_on**
Put work item to global workqueue(``system_wq``) on a specific CPU after delay time.
```c
bool schedule_delayed_work_on(int cpu, struct delayed_work *dwork, unsigned long delay);
```
+ ``cpu``: CPU to put the work task on
+ ``dwork``: delay-``work item``(delay job) to be done
+ ``delay``: number of jiffies to wait or 0 for immediate execution

#### Delete work from workqueue
Wait for a work to finish executing the last queueing instance.
```c
bool flush_work( struct work_struct *work );
```
+ ``work``: the work to flush
+ ``Return``: ``true`` if ``flush_work()`` waited for the work to finish execution, ``false`` if it was already idle. 

And for a delayed work

Wait for a dwork to finish executing the last queueing.
```c
bool flush_delayed_work(struct delayed_work * dwork);
```
+ ``dwork``: the delayed work to flush
+ ``Return``: ``true`` if ``flush_delayed_work()`` waited for the dwork to finish execution, ``false`` if it was already idle.

#### Cancel work from workqueue
Cancel a work and wait for it to finish.
```c
bool cancel_work_sync(struct work_struct * work);
```
+ ``work``: the work to cancel
+ ``Return``: ``true`` if work was pending, ``false`` otherwise.

And for delayed work

Cancel a delayed work and wait for it to finish.
```c
bool cancel_delayed_work_sync(struct delayed_work * dwork);
```
+ ``dwork``: the delayed work to cancel.
+ ``Return``: ``true`` if dwork was pending, ``false`` otherwise.

#### Check the workqueue
Find out whether a work item is currently pending.
```c
work_pending(work)
```
+ ``work``: The work item in question


Find out whether a delayable work item is currently pending
```c
delayed_work_pending(work)
```
+ ``work``: The work item in question


## Our code
The code located in folder 10_workqueue
```bash
# To build ko driver 
make build-10
# To clean ko driver 
make clean-10
# To upload ko driver to our board 
make upload-10
```
```
[root@luckfox root]# insmod ki_wq_drv.ko
[root@luckfox root]# dmesg | tail
[    8.744105] rwnx_send_sm_connect_req drv_vif_index:0 connect to Ae809810(8) channel:2412 auth_type:0
[    9.627080] of_dma_request_slave_channel: dma-names property of node '/serial@ff4b0000' missing or empty
[    9.627116] dw-apb-uart ff4b0000.serial: failed to request DMA, use interrupt mode
[  141.689537] 8
[  142.342961] 8
[  151.469222] 8
[  159.327201] ki_drv driver initializing...
[  159.328421] ki_drv module inserted successfully.
[  159.330231] Executing dynamic workqueue function
[  159.330254] Executing static workqueue function
[root@luckfox root]#
```