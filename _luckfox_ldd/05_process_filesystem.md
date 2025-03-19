---
layout: default
title: "05 Process filesystem"
parent: Linux device driver on Luckfox
nav_order: 5
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 05 Process filesystem
Linux OS seperates memory into two parts: user space and kernel space. User space is where the application runs, and kernel space is where the OS runs. The kernel space is protected from user space by a mechanism called memory protection.

To communicate between user space and kernel space, there are several ways:
+ ioctl
+ procfs
+ sysfs
+ netlink
+ configfs
+ debugfs
+ ...

### Introducing an ``procfs``
``procfs``, or the process file system, is a virtual filesystem in Linux that provides a mechanism for the kernel to expose information about processes and other system information to user space. It is typically mounted at ``/proc`` and contains a wealth of information that can be useful for system monitoring, debugging, and performance analysis.

Here is some entry to retrieve info from kernel:
+ ``/proc/devices`` — registered character and block major numbers
+ ``/proc/iomem`` — on-system physical RAM and bus device addresses
+ ``/proc/ioports`` — on-system I/O port addresses (especially for x86 systems)
+ ``/proc/interrupts`` — registered interrupt request numbers
+ ``/proc/softirqs`` — registered soft IRQs
+ ``/proc/swaps`` — currently active swaps
+ ``/proc/kallsyms`` — running kernel symbols, including from loaded modules
+ ``/proc/partitions`` — currently connected block devices and their partitions
+ ``/proc/filesystems`` — currently active filesystem drivers
+ ``/proc/cpuinfo`` — information about the CPU(s) on the system

``procfs`` can also be used to read and modify kernel parameters at runtime through the ``/proc/sys`` directory. This allows for dynamic configuration of kernel settings without needing to reboot the system.

```Note:``` Changes made to certain files in /proc/sys can affect system behavior immediately, so caution is advised when modifying these files.

### Creating ``procfs`` directory
Using this API, we can create a directory under ``/proc/``.
```c
struct proc_dir_entry *proc_mkdir(const char *name, struct proc_dir_entry *parent)
```
+ ``name``: The name of the directory that will be created under /proc.
+ ``parent``: In case the folder needs to be created in a subfolder under /proc a pointer to the same is passed else it can be left as NULL.
### Creating ``procfs`` entry
```c
struct proc_dir_entry *proc_create(const char *name, umode_t mode, struct proc_dir_entry *parent, const struct proc_ops *proc_ops);
```
+ ``name``: The name of the proc entry
+ ``mode``: The access mode for proc entry
+ ``parent``: The name of the parent directory under ``/proc``. If NULL is passed as a parent, the ``/proc`` directory will be set as a parent.
+ ``proc_fops``: The structure in which the file operations for the proc entry will be created.
+ ``Returns``: Pointer to the created proc_dir_entry on success, or NULL on failure.

The ``struct proc_ops`` is new from kernel version 3.10. Looks similar to file operation:
```c
struct proc_ops {
	unsigned int proc_flags;
	int	(*proc_open)(struct inode *, struct file *);
	ssize_t	(*proc_read)(struct file *, char __user *, size_t, loff_t *);
	ssize_t (*proc_read_iter)(struct kiocb *, struct iov_iter *);
	ssize_t	(*proc_write)(struct file *, const char __user *, size_t, loff_t *);
	loff_t	(*proc_lseek)(struct file *, loff_t, int);
	int	(*proc_release)(struct inode *, struct file *);
	__poll_t (*proc_poll)(struct file *, struct poll_table_struct *);
	long	(*proc_ioctl)(struct file *, unsigned int, unsigned long);
#ifdef CONFIG_COMPAT
	long	(*proc_compat_ioctl)(struct file *, unsigned int, unsigned long);
#endif
	int	(*proc_mmap)(struct file *, struct vm_area_struct *);
	unsigned long (*proc_get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
} __randomize_layout;
```
### Remove ``procfs`` directory
```c
void proc_remove(struct proc_dir_entry *parent);
```

### Remove ``procfs`` entry
```c
void remove_proc_entry(const char *name, struct proc_dir_entry *parent);
```

### Note about pros file operation read
It is just not enough when we want to read a file and return the len we have read. It also need to update the offset (``loff_t *``), and next time check where is the offset, otherwise the output will be a non-stop infinite sequence of message like this:
```
[root@luckfox root]# cat /proc/PROCFS_NONAME/PROCFS_ENTRY_NONAME
Message from my procfsMessage from my procfsMessage from my procfsMessage from my procfsMessage from my procfsMessage from my procfsMessage from my procfsMessage from ^C
[root@luckfox root]# 
```
```c
static ssize_t my_proc_read(struct file *file, char __user *buffer, size_t length, loff_t *offset)
{
    char my_msg[] = "Message from my procfs";
	// Check if we have reached the end of the file
	if (*offset >= msg_len) {
		return 0; // End of file
	}
    // Copy the message to user space
    if (copy_to_user(buffer, my_msg, strlen(my_msg))) {
        return -EFAULT; // Error copying to user
    }
    *offset += strlen(my_msg); // Update the offset
    return (ssize_t)strlen(my_msg);
}
```

## Our code
The code located in folder 05_process_filesystem
```bash
# To build ko driver 
make build-05
# To clean ko driver 
make clean-05
# To upload ko driver to our board 
make upload-05
```

Result
```
[root@luckfox root]# insmod pf.ko
[root@luckfox root]# ls /proc/ | grep NONAME
PROCFS_NONAME
[root@luckfox root]# ls /proc/PROCFS_NONAME/
PROCFS_ENTRY_NONAME
[root@luckfox root]# cat /proc/PROCFS_NONAME/PROCFS_ENTRY_NONAME
Message from my procfs
[root@luckfox root]# echo "Hello from user space" > /proc/PROCFS_NONAME/PROCFS_ENTRY_NONAME
[root@luckfox root]# cat /proc/PROCFS_NONAME/PROCFS_ENTRY_NONAME
Hello from user space
```