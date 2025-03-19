---
layout: default
title: "04 Input Output Control"
parent: Linux device driver on Luckfox
nav_order: 4
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 04 Input Output Control
Linux OS seperates memory into two parts: user space and kernel space. User space is where the application runs, and kernel space is where the OS runs. The kernel space is protected from user space by a mechanism called memory protection.

To communicate between user space and kernel space, there are several ways:
+ ioctl
+ procfs
+ sysfs
+ netlink
+ configfs
+ debugfs
+ ...

### Introducing an ``ioctl``

Input-output control (ioctl, in short) is a common operation or system call available with most of the driver categories. It provides a way for user space applications to communicate with device drivers. It allows user space to send control commands to the driver.

#### Purpose of using ``ioctl``
+ Configure device parameters.
+ Control device behavior.
+ Retrieve device status. 
+ It can handle a wide range of operations, such as setting device modes, getting device information, or performing specific actions.

The ``ioctl`` function is nothing but a simple function with switch...case implementation over the command implementing the corresponding functionalities.

There are some steps involved in implementing of ``ioctl`` in Linux Device Drivers:
+ Create ``ioctl`` command in the driver
+ Write the ``ioctl`` function in the driver
+ Create ``ioctl`` command in a Userspace application
+ Use the ``ioctl`` system call in a Userspace

#### 1. Create ``ioctl`` command in the driver
We need to know macro structure of creating command:
```c
#define _IO(type, number)         // For commands without data
#define _IOR(type, number, data)  // For commands that read data
#define _IOW(type, number, data)  // For commands that write data
#define _IOWR(type, number, data) // For commands that read and write data
```
Where:
+ ``_IO``: Used for commands that do not require any data to be passed to or from the driver.
+ ``_IOR``: Used for commands that read data from the driver to user space.
+ ``_IOW``: Used for commands that write data from user space to the driver.
+ ``_IOWR``: Used for commands that both read and write data.

and:

+ ``type``: A unique character that identifies the driver or subsystem. This helps avoid conflicts between different drivers.
+ ``number``: A unique number for the command within the driver.
+ ``data``: The type of data being passed (e.g., int, struct, etc.).

Example:
```c
#include <linux/ioctl.h>

typedef struct
{
    int day, month, year;
} date_t;

// Define a unique character for the driver
#define MY_IOCTL_MAGIC 'M'

// Define command codes
#define MY_IOCTL_SET_VALUE _IOW(MY_IOCTL_MAGIC, 1, int)  // Command to set a value
#define MY_IOCTL_GET_VALUE _IOR(MY_IOCTL_MAGIC, 2, int)  // Command to get a value
#define MY_IOCTL_RESET     _IO(MY_IOCTL_MAGIC, 3)         // Command to reset the device

#define MY_IOCTL_GET_DATE _IOR(MY_IOCTL_MAGIC, 4, date_t* )  // Command to get a date
```

#### 2. Write the ``ioctl`` function in the driver
In the ``struct file_operations`` has an attribute ``unlocked_ioctl``. We need to implement that funtion to handle the ``ioctl`` requests.

In my code, i wrote the funtion ``my_ioctl`` and then link it to ``unlocked_iotcl``.
```c
// File operations structure
static struct file_operations fops = {
    .owner = THIS_MODULE,
    .unlocked_ioctl = my_ioctl,
    // Other file operations (open, read, write, etc.) can be added here
};
```

Below is the prototype of the function:
```c
#if (LINUX_VERSION_CODE < KERNEL_VERSION(2,6,35))
      int my_ioctl(struct inode *i, struct file *f, unsigned int cmd, unsigned long arg){}
#else
      long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg){}
#endif
```
Where:

+ ``inode``: is the inode number of the file being worked on.
+ ``file``: is the file pointer to the file that was passed by the application.
+ ``cmd``: is the ioctl command that was called from the userspace.
+ ``arg``: are the arguments passed from the userspace

#### 3. Create ``ioctl`` command in a Userspace application
Just define the ioctl command like how we defined it in the driver.
```c
// Define a unique character for the driver
#define MY_IOCTL_MAGIC 'M'

// Define command codes
#define MY_IOCTL_SET_VALUE _IOW(MY_IOCTL_MAGIC, 1, int)  // Command to set a value
#define MY_IOCTL_GET_VALUE _IOR(MY_IOCTL_MAGIC, 2, int)  // Command to get a value
#define MY_IOCTL_RESET     _IO(MY_IOCTL_MAGIC, 3)         // Command to reset the device

#define MY_IOCTL_GET_DATE _IOR(MY_IOCTL_MAGIC, 4, date_t* )  // Command to get a date
```

### 4. Use the ``ioctl`` system call in a Userspace
After define command codes and a data structure that matches what the driver expects, open the device file, in userspace we use ioctl to send and receive data.

```c
#include <sys/ioctl.h>
int ioctl(int fd, unsigned long cmd, ...);
```
+ ``fd`` is file descriptor
+ ``cmd`` is cmd 
+ ``...`` can be arguments


## Our code
The code located in folder 04_input_output_control
```bash
# To build ko driver 
make build-04
# To clean ko driver 
make clean-04
# To upload ko driver to our board 
make upload-04
```

Result
```
[root@luckfox root]# ls | grep -e user -e io
ioctrl.ko
user_app
[root@luckfox root]# insmod ioctrl.ko
[root@luckfox root]# ./user_app
*******User App IOCTL*******

Opening Driver
Reading Value from Driver
Day 13, month 3, year 2025
Closing Driver
```
