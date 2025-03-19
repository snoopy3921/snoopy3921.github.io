---
layout: default
title: "03 File operations"
parent: Linux device driver on Luckfox
nav_order: 3
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 03 File operations
So, our device file is created. To perform action on it like open, close, read, write. We need to register some structures to the driver.

Each device in the kernel is represented by a structure called ``file``(yes, our device file), which is defined in the file ``<linux/fs.h>``. This structure is used exclusively by the kernel and is never utilized by user-space applications. It is important to note that this is completely different from ``FILE``, which is defined by the ``glibc library`` and is not used anywhere in the kernel. The name of the structure can be misleading because it represents an abstraction of an open file, rather than a file on disk, which is represented by the ``inode`` structure.

| Aspect  | file (Device Driver) |  FILE (glibc) |
| ------------- | ------------- |  ------------- |
| Context  | Kernel space (device interaction)  |  User space (file I/O operations)  |
| Structure  | struct file (kernel structure)  |  FILE (glibc structure)  |
| Purpose  | Represents device files for hardware access  |  Represents file streams for buffered I/O  |
| Operations  | Handled by kernel through file operations  |  Handled by glibc functions (e.g., fopen, fread)  |
| Buffering  | Typically unbuffered (direct device access)  |  Buffered I/O for efficiency |
| Error Handling  | Kernel-level error handling  |  User-level error handling (e.g., ferror)  |

Until now, our driver has created device files with major and minor numbers. However, these files do not have any operations associated with them, so we cannot do anything other than observe their existence.

There are 3 steps:

1. Fill in a file operations structure (struct file_operations fops) with the desired file operations (``my_open``, ``my_close``, ``my_read``, ``my_write``, …)
2. Initialize the character device structure ``(struct cdev c_dev)`` using ``cdev_init()``.
3. Register the character device structure with the kernel using ``cdev_add()``.
+ Both ``cdev_init()`` and ``cdev_add()`` are declared in ``<linux/cdev.h>``.

Take a look at ``read()`` and ``write()`` operation:
```c
ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
```
So note that the return type is ssize_t, not int.

My code file the cdev is created statically. But you can create it dynamically by these APIs:
```c
struct cdev *my_cdev = cdev_alloc( );
my_cdev->ops = &my_fops;
```

To remove a char device from the system:
```c
void cdev_del(struct cdev *dev);
```

## Our code
The code located in folder 03_file_operations
```bash
# To build ko driver 
make build-03
# To clean ko driver 
make clean-03 
# To upload ko driver to our board 
make upload-03
```

Result
```
[root@luckfox root]# ls
dfc.ko  fo.ko   mam.ko  ofd.ko
[root@luckfox root]# insmod fo.ko
[root@luckfox root]# cat /proc/devices | grep fo
241 fo
[root@luckfox root]# ls -l /dev/ | grep fo
crw-------    1 root     root      241,   0 Mar 10 20:00 fo_dev
[root@luckfox root]# echo "Hi" > /dev/fo_dev
[root@luckfox root]# dmesg | tail -5
[108593.967023] Fo driver initializing...
[108593.967727] Fo module inserted successfully.
[108754.968437] Fo driver: open()
[108754.968495] Fo driver: write()
[108754.968516] Fo driver: close()
[root@luckfox root]# cat /dev/fo_dev
[root@luckfox root]# dmesg | tail -8
[108593.967023] Fo driver initializing...
[108593.967727] Fo module inserted successfully.
[108754.968437] Fo driver: open()
[108754.968495] Fo driver: write()
[108754.968516] Fo driver: close()
[108822.493281] Fo driver: open()
[108822.493392] Fo driver: read()
[108822.493410] Fo driver: close()
```
As we write "Hi" to our device file name ``fo_dev``, that makes driver ``open`` fo_dev, ``write`` to fo_dev and ``close`` fo_dev. And we can read what is in ``fo_dev`` after writting by cat /dev/fo_dev, that makes driver ``open`` fo_dev, ``read`` to fo_dev and ``close`` fo_dev, but reading shows nothing. Next chapter we will discover more.