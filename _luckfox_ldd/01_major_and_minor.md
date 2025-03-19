---
layout: default
title: "01 Major and minor"
parent: Linux device driver on Luckfox
nav_order: 1
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
nav_enabled: false
---
## 01 Major and minor

Devices when connect to linux system located in /dev. Linux treats devices as files, so we call device files interchangeably.

Each device file has its own major and minor number(device files are created by device drivers). The ``major number`` identifies the driver associated with the device, while the ``minor number`` is used to distinguish between different devices or instances of the same type managed by that driver.

<center>
<img src="/assets/images/major_and_minor.png" 
alt="Luckfox pico ultra - Wifi version" />
</center>

```
+---------------------+
|   User-Space App    |
|  (e.g., open /dev)  |
+----------+----------+
           |
+---------------------+
|     Device File     |
|  (e.g., /dev/fb0)   |
+----------+----------+
           |
+---------------------+
|     Major/Minor     |
|    (e.g., 250 1 )   |
+----------+----------+
           |
+---------------------+
|   Device Driver     |
|  (e.g., my_driver)  |
+----------+----------+
           |
+---------------------+
|       Hardware      |
|(e.g., oled display) |
+----------+----------+
```

+ Device structure type: (defined in kernel header ``<linux/types.h>``)
```c
dev_t // contains both major & minor numbers
```

+ Macros: (defined in kernel header ``<linux/kdev_t.h>``)
```c
MAJOR(dev_t dev) // extracts the major number from dev
MINOR(dev_t dev) // extracts the minor number from dev
MKDEV(int major, int minor) // creates the dev from major & minor
```


We can allocate the major and minor numbers in two ways.

1. Statically allocating
2. Dynamically allocating

#### Statically allocating
This API (defined in kernel header ``<linux/fs.h>``) registers the cnt number of device file numbers starting from first, with the name. Return 0 if the allocation was successfully.

```c
int register_chrdev_region(dev_t first, unsigned int count, char *name);
```
Example
```c
static dev_t first; // Global variable for the first device number
first = MKDEV(250, 1); // Major = 150, Minor = 1

register_chrdev_region(first, 1, "my_device"); // Register 1 device started from first dev with the name my_device
```

#### Dynamically allocating
If we don’t want the fixed major and minor numbers so use this method will allocate the major number dynamically to your driver which is available.
```c
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name);
```
Example
```c
static dev_t first; // Global variable for the first device number

alloc_chrdev_region(&first, 0, 1, "my_device"); // Alloc major number for first dev with 1 device, the minor number starts from 0 and the name is my_device
```

#### Unregister the Major and Minor Number
Regardless of how you allocate your device numbers, you should free them when they are no longer in use. Device numbers are freed with:

```c
void unregister_chrdev_region(dev_t first, unsigned int count);
```
And usually, we should do this in the module’s cleanup function.


## Our code
The code located in folder 01_major_and_minor
```bash
# To build ko driver 
make build-01
# To clean ko driver 
make clean-01 
# To upload ko driver to our board 
make upload-01 
```

Result

1. Statically
```
[root@luckfox root]# insmod mam.ko
[root@luckfox root]# dmesg | tail -5
[ 5111.010099] 8
[ 5111.011428] 8
[ 5126.626149] 8
[ 5134.642838] Mam driver initializing...
[ 5134.642870] Mam driver registered, Major 235, Minor 1
[root@luckfox root]# cat /proc/devices | grep mam
235 mam_dev
```
2. Dynamically
```
[root@luckfox root]# insmod mam.ko
[root@luckfox root]# dmesg | tail -5
[ 5134.642870] Mam driver registered, Major 235, Minor 1
[ 6436.434407] 8
[ 6454.953887] Mam driver unregistered
[ 6460.058224] Mam driver initializing...
[ 6460.058263] Mam driver registered, Major 242, Minor 0
[root@luckfox root]# cat /proc/devices | grep mam
242 mam_dev
[root@luckfox root]#
```