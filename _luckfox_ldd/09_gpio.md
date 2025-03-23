---
layout: default
title: "09 gpio"
parent: Linux device driver on Luckfox
nav_order: 9
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 09 GPIO driver
GPIO in Linux system is controlled by the Kernel GPIO subsystem. 

### GPIO APIs
Include ``<linux/gpio.h>`` for control GPIO in linux driver.

#### **gpio_is_valid**
To check if a GPIO is valid.
```c
bool gpio_is_valid(int gpio_number);
```
+ ``gpio_number`` : GPIO that you are planning to use
+ ``Return`` : false if it is not valid otherwise, it returns true.


#### **gpio_request**
Request the GPIO
```c
int gpio_request(unsigned gpio, const char *label);
```
+ ``gpio`` : GPIO that you are planning to use
+ ``label`` :  label used by the kernel for the GPIO in sysfs. You can provide any string that can be seen in ``/sys/kernel/debug/gpio``. Do ``cat /sys/kernel/debug/gpio``. You can see the GPIO assigned to the particular GPIO.
+ ``Return`` : 0 if success and a negative number if failure.


Request one GPIO
```c
int gpio_request_one(unsigned gpio, unsigned long flags, const char *label);
```

Request multiple GPIOs
```c
int gpio_request_array(struct gpio *array, size_t num); 
```

#### **gpio_export**
For debugging purposes, you can export the GPIO which is allocated to the sysfs
```c
int gpio_export(unsigned int gpio, bool direction_may_change);
```
+ ``gpio`` : GPIO that you want to export
+ ``direction_may_change`` :  This parameter controls whether user space is allowed to change the direction of the GPIO
  + true – Can change
  + false – Can’t change.
+ ``Return`` : zero on success, else an error.

{: .note }
When you export a GPIO for example gpio72 with the option direction_may_change true, you can see it in ``/sys/class/gpio/gpio72/direction``.

{: .note }
Once you export the GPIO, you can see the GPIO in ``/sys/class/gpio/*``. There you can the GPIO’s value.

#### **gpio_unexport**
Unexport the exported GPIO
```c
void gpio_unexport(unsigned int gpio);
```
+ ``gpio`` : GPIO that you want to unexport

#### **Set the direction of the GPIO**
Set the GPIO direction (output or input)

```c
int  gpio_direction_input(unsigned gpio);
```

```c
int  gpio_direction_output(unsigned gpio, int value);
```

+ ``gpio`` : GPIO that you want to set the direction.
+ ``value`` : The value of the GPIO once the output direction is effective.
+ ``Return`` : zero on success, else an error.

#### **Set and get value of the GPIO**
Once you set the GPIO direction as an output, use this API can set the value of GPIO
```c
void gpio_set_value(unsigned gpio, int value);
```
+ ``gpio`` :  GPIO that you want to change the value.
+ ``value`` : value to set to the GPIO. ``0 – Low``, ``1 – High``.

Once you set the GPIO direction as an input, use this API can get the value of GPIO
```c
int  gpio_get_value(unsigned gpio);
```
+ ``gpio`` :  GPIO that you want to get the value.
+ ``Return``: GPIO value. ``0 – Low``, ``1 – High``.

#### **GPIO Interrupt (IRQ)**
To use interrupt in general, we need to know the ``IRQ line``, and ``IRQ line`` is hardware-dependent. Use this API to get the ``IRQ line`` for GPIO number
```c
int gpio_to_irq(unsigned gpio);
```
+ ``gpio`` : GPIO number you want to use with interrupt input.
+ ``Return`` : IRQ line

Beside flags for use with normal interrupt, GPIO interrupt also has some flags:
+ ``IRQF_TRIGGER_NONE``	
+ ``IRQF_TRIGGER_RISING``	
+ ``IRQF_TRIGGER_FALLING``	
+ ``IRQF_TRIGGER_HIGH``	
+ ``IRQF_TRIGGER_LOW``	

#### **gpio_free**
Once you have done with the GPIO, then you can release the GPIO which you have allocated previously
```c
void gpio_free(unsigned int gpio);
```
+ ``gpio`` : GPIO number you want release.

Release multiple GPIOs.
```c
void gpio_free_array(struct gpio *array, size_t num);
```