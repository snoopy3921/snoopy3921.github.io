---
layout: default
title: "12 kernel timer"
parent: Linux device driver on Luckfox
nav_order: 12
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 12 Kernel timer
Timer is the crucial part of every embedded system, some applications require a timer to be used as a clock source, others require a timer to be used as a counter,... Any way, timer manages flow of time, it increases timer unit(tick in RTOS, or in jiffies Linux). And kernel timer is just a piece of code that help managing list of timers. Kernel timer requires hardware timer from SoC, same as MCU implements timers. 

Kernel timer is defined in ``<linux/timer.h>``. A timer is executed only once — timers aren’t cyclic. Therefore, to make it cyclic, we need to re-enable it every time it expires. Kernel timers are described by the ``timer_list`` structure:
```c  
struct timer_list {
	/*
	 * All fields that change during normal runtime grouped to the
	 * same cacheline
	 */
	struct hlist_node	entry;
	unsigned long		expires;
	void			(*function)(struct timer_list *);
	u32			flags;
};
```

### Initialize a kernel timer
With new kernel, to init the timer, we need to call ``timer_setup()`` function.
```c
timer_setup(timer, callback, flags)
```
+ ``timer`` – the timer to be initialized

+ ``function`` – Callback function to be called when the timer expires. In this callback function, the argument will be struct timer_list *.

+ ``data`` – data has to be given to the callback function
### Start a kernel timer
Calling this API to start timer that was initialized.
```c
void add_timer(struct timer_list *timer);
```
### Modify a kernel timer's timeout
```c
int mod_timer(struct timer_list *timer, unsigned long expires);
```
``mod_timer(timer, expires)`` is equivalent to:

1. del_timer(timer);
2. timer->expires = expires;
3. add_timer(timer);

### Stop a kernel timer
Calling this API will deactivate a timer. This works on both active and inactive timers.
```c
int del_timer(struct timer_list * timer);
```
Return:
+ 0 – del_timer of an inactive timer
+ 1 – del_timer of an active timer

Same as ``del_timer``, del_timer_sync also delete the timer, but it also wait for the handler to finish. This purpose is to avoid race condition. 
```c
int del_timer_sync(struct timer_list * timer);
```
For example. When callback function has some shared datas, and we are programming multi-thread. The ``del_timer_sync`` should be used to avoid such situation where, after module exit and all shared datas are considered as deleted but the callback still in progress. This can lead to accessing invalid memory.

### Check kernel timer status
This will tell whether a given timer is currently pending, or not.
```c
int timer_pending(const struct timer_list * timer);
```
Return:
+ 0 – timer is not pending
+ 1 – timer is pending

## Our code
We will use kernel timer to trigger callback funtion every 300ms, and toggle the led.
The code located in folder 12_kernel_timer
```bash
# To build ko driver 
make build-12
# To clean ko driver 
make clean-12
# To upload ko driver to our board 
make upload-12
```
```
[root@luckfox root]# insmod ki_kernel_timer_drv.ko
[root@luckfox root]# dmesg | tail -10
[  133.065870] Timer Callback function Called [5]
[  133.375890] Timer Callback function Called [6]
[  133.685882] Timer Callback function Called [7]
[  133.995873] Timer Callback function Called [8]
[  134.305894] Timer Callback function Called [9]
[  134.615872] Timer Callback function Called [10]
[  134.925875] Timer Callback function Called [11]
[  135.235870] Timer Callback function Called [12]
[  135.545870] Timer Callback function Called [13]
[  135.855892] Timer Callback function Called [14]
```

<iframe width="720" height="480" src="https://www.youtube.com/embed/swz0aPNnm0A" frameborder="0" allow="autoplay" allowfullscreen></iframe>

