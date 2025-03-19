---
layout: default
title: "02 Device file creation"
parent: Linux device driver on Luckfox
nav_order: 2
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
nav_enabled: false
---
## 02 Device File Creation
At the previous part we have registered a device with the kernel. So kernel can manage it, but in for a device the user want to control, we need to create a device file. 

All device files are stored in ``/dev`` directory

There are two ways to create a device file:
1. Manually using mknod command
2. Automatically using ``udev`` deamon

#### Manually using mknod command
The basic syntax for the ``mknod`` command is:
```bash
mknod [options] <filename> <type> <major> <minor>
```
+ ``<filename>``: The name of the device file you want to create (e.g., /dev/my_device).
+ ``<type>``: The type of device:
  + Use c for character devices 
  + Use b for block devices.
+ ``<major>``: The major number assigned to the device.
+ ``<minor>``: The minor number assigned to the device.

Navigate to our board.
```
[root@luckfox root]# cat /proc/devices | grep mam
242 mam_dev
[root@luckfox root]# mknod /dev/mam_dev c 242 0
[root@luckfox root]# ls -l /dev/ | grep mam
crw-r--r--    1 root     root      242,   0 Mar  9 11:03 mam_dev
```

To remove the device file, use the ``rm`` command. 

{: .note }
The ``rmmod`` does not remove the device file.
```
[root@luckfox root]# rmmod mam.ko
[root@luckfox root]# ls -l /dev/ | grep mam
crw-r--r--    1 root     root      242,   0 Mar  9 11:03 mam_dev
[root@luckfox root]# rm /dev/mam_dev
[root@luckfox root]# ls -l /dev/ | grep mam
[root@luckfox root]#
```

{: .note }
Anyone can create the device file using this method.

{: .note }
You can create the device file even before loading the driver.


#### Automatically using udev deamon

When a device is connected to the system (e.g., a USB drive, a network card), the Linux kernel detects this hardware and populates information about the device in the ``/sys`` filesystem.

And then, the user space need to interpret it and take an appropriate action. In most Linux desktop systems, the `udev` daemon picks up that information and accordingly creates the device files.

How `udev` works:

`udev` is a device manager for the Linux kernel that dynamically creates and removes device nodes in the `/dev` directory based on the devices that are present in the system. Here’s how `udev` operates:

1. **Device Detection**: 
   - When a device is connected to the system (e.g., a USB device, a new hard drive, etc.), the kernel generates an event indicating that a new device has been detected.

2. **Event Handling**: 
   - The `udev` daemon listens for these events. When it receives a notification about a new device, it processes the event.

3. **Device Node Creation**: 
   - Based on the rules defined in `/etc/udev/rules.d/` and `/lib/udev/rules.d/`, `udev` creates the appropriate device file in `/dev`. The rules specify how to name the device files, what permissions to set, and other attributes.

4. **Device Removal**: 
   - When a device is disconnected or unregistered, `udev` automatically removes the corresponding device file from `/dev`.

These following steps will create a device file:
1. Include the header file linux/device.h and linux/kdev_t.h
2. Register major and minor number for driver.
2. Create the struct Class
3. Create Device with the class which is created by the above step

#### Create the class
This API creates a struct class and return the pointer to that struct class. The structure is created under ``/sys/class/``.
```c
struct class * class_create(struct module *owner, const char *name);
```
+ ``owner`` – pointer to the module that is to “own” this struct class
+ ``name`` – pointer to a string for the name of this class
+ Use ``IS_ERR(struct class *)`` to check error.
+ If error is indicated by IS_ERR, use ``PTR_ERR(struct class *)``.
#### Destroy the class
```c
void class_destroy (struct class * cls);
```

#### Create device
Alright, till now we had driver registered with major and minor numbers, we had class created, last one is create a device file.

Using this API, a struct device will be created and registered to the specified class. That struct device is managed internally by the kernel, so dont care it much. Care about ``devt`` and ``class``.
```c
struct device *device_create(struct *class, struct device *parent, dev_t devt, void * drvdata, const char *fmt, ...);
```
+ ``class`` – pointer to the struct class that this device should be registered to

+ ``parent`` – pointer to the parent struct device of this new device, if any

+ ``devt`` – the dev_t for the char device to be added

+ ``drvdata`` – the data to be added to the device for callbacks

+ ``fmt`` – string for the device’s name

+ ``...`` – variable arguments

+ Use ``IS_ERR(struct device *)`` to check error.
#### Destroy the device
```c
void device_destroy (struct class * class, dev_t devt);
```

## Our code
The code located in folder 02_device_file_creation
```bash
# To build ko driver 
make build-02
# To clean ko driver 
make clean-02
# To upload ko driver to our board 
make upload-02 
```

Result

```
[root@luckfox root]# ls | grep dfc
dfc.ko
[root@luckfox root]# insmod dfc.ko
[root@luckfox root]# lsmod | grep dfc
dfc                      956  0
[root@luckfox root]# cat /proc/devices | grep dfc
242 dfc
[root@luckfox root]# ls -l /dev/ | grep dfc
crw-------    1 root     root      242,   0 Mar  9 14:19 dfc_dev
[root@luckfox root]# rmmod dfc.ko
[root@luckfox root]# dmesg | tail -3
[ 1713.987720] Dfc driver initializing...
[ 1713.988927] Dfc module inserted successfully.
[ 1731.771590] Dfc driver unregistered
```