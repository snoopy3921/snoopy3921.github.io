---
layout: default
title: "07 Sys filesystem"
parent: Linux device driver on Luckfox
nav_order: 7
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 07 Sysfs
We have walked through ioctl, profs, all of this is ways to communicate between user space and kernel space. And now is other one call ``sysfs``.

``sysfs`` is a virtual filesystem in Linux that provides a way to interact with kernel objects and their attributes. It is always mounted at ``/sys``. The files in ``sysfs`` contain information about devices and drivers. Some files in ``sysfs`` are even writable, for configuration and control of devices attached to the system.

Doesnt like ``procfs`` which is used to export information about processes, although both ``procfs`` and ``sysfs`` are virtual filesystems, ``sysfs`` is used to export information about specific devices. More detail here:
+ [What is the difference between procfs and sysfs?](https://unix.stackexchange.com/questions/4884/what-is-the-difference-between-procfs-and-sysfs)
+ ``procfs`` is the old one, it is more or less without rules and structure. And at some point it was decided that ``procfs`` was a little too chaotic and a new way was needed.
+ Then ``sysfs`` was created, and the new stuff that was added was put into ``sysfs`` like device information.
+ Interface in ``procfs`` is files, and interface in ``sysfs`` is hierarchical. 

### Kernel Objects
The heart of the ``sysfs`` interface is the ``kernel object``. 

Kernel objects represent various entities managed by the Linux kernel, such as devices, drivers, and subsystems. These objects are exposed to user space through a hierarchical filesystem structure, allowing users and applications to interact with them easily.

Kernel object is represented by ``struct kobject`` and defined in ``<linux/kobject.h>``.

Take a look at koobject structure:
```c
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject	      *parent;
	struct kset		*kset;
	struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;

	ANDROID_KABI_RESERVE(1);
	ANDROID_KABI_RESERVE(2);
	ANDROID_KABI_RESERVE(3);
	ANDROID_KABI_RESERVE(4);
};
```
Some of the important fields are:
+ ``name`` :Name of the kobject. Current kobject is created with this name in sysfs.

+ ``parent`` :This is kobject’s parent. When we create a directory in sysfs for the current kobject, it will create under this parent directory.

+ ``ktype`` :the type associated with a kobject. The ``kobj_type`` structure contains function pointers for operations that can be performed on the kobject, such as showing and storing attributes. This allows for polymorphic behavior, where different types of kobjects can have different behaviors.

+ ``kset`` :This pointer points to the kset (kernel set) that the kobject belongs to. A kset is a collection of kobjects that share a common purpose or functionality. This allows for grouping related kobjects together, which can be useful for managing them ``collectively``.

+ ``sd`` :points to a sysfs directory entry structure that represents this kobject in sysfs.

+ ``kref`` :provides reference counting. It ensures that the kobject is not freed while it is still in use. When the count reaches ``zero``, it will automatically free the kobject.

+ ``entry`` :This is a list entry that allows the kobject to be linked into a list of kobjects. It is part of the Linux kernel's linked list implementation, enabling easy traversal and management of multiple kobjects.

So ``kobject`` is used to create kobject directory in ``/sys``. This is enough. We will not go deep into the kobjects.

### Creating and using ``sysfs``.
There are several steps:
+ Create a directory in ``/sys``.
+ Create ``sysfs`` file.

#### Create a directory in ``/sys``
Using this API:
```c
struct kobject * kobject_create_and_add ( const char * name, struct kobject * parent);
```
+ ``name`` – the name for the kobject

+ ``parent`` – the parent kobject of this kobject, if any.

There are several special parent objects can be passed into this API:
```c
/* The global /sys/kernel/ kobject for people to chain off of */
extern struct kobject *kernel_kobj;
/* The global /sys/kernel/mm/ kobject for people to chain off of */
extern struct kobject *mm_kobj;
/* The global /sys/hypervisor/ kobject for people to chain off of */
extern struct kobject *hypervisor_kobj;
/* The global /sys/power/ kobject for people to chain off of */
extern struct kobject *power_kobj;
/* The global /sys/firmware/ kobject for people to chain off of */
extern struct kobject *firmware_kobj;

/* /sys/fs */
extern struct kobject *fs_kobj;
```
Explain:

+ If you pass ``kernel_kobj`` to the parent argument, it will create the directory under ``/sys/kernel/``. 
+ If you pass ``firmware_kobj`` to the parent argument, it will create the directory under ``/sys/firmware/``. 
+ If you pass ``fs_kobj`` to the parent argument, it will create the directory under ``/sys/fs/``. 
+ If you pass ``NULL`` to the parent argument, it will create the directory under ``/sys/``.

This function creates a kobject structure dynamically and registers it with sysfs. If the kobject was not able to be created, ``NULL`` will be returned.


When you are finished with this structure, call kobject_put and the structure will be dynamically freed when it is no longer being used.

```c
void kobject_put(struct kobject *kobj);
```

### Create ``sysfs`` file
We have created directory for our sysfile, to create sysfile, which is used to interact user space with kernel space through sysfs. 

Sysfile is actually ``sysfs`` attribute, so we call file attribute, there are one value per file attribute. They can be found in the header file ``<linux/sysfs.h>``

#### Create attribute
Kobj_attribute is defined as:
```c
struct kobj_attribute {
	struct attribute attr;
	ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr, char *buf);
	ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count);
};
```
+ ``attr`` – the attribute representing the file to be created
+ ``show`` – the pointer to the function that will be called when the file is read in sysfs
+ ``store`` – the pointer to the function which will be called when the file is written in sysfs.

Each sysfs attribute should have one or both of the two functions defined above.

We can create an attribute using ``__ATTR`` macro, which is defined in ``<linux/sysfs.h>``:
```c
#define __ATTR(_name, _mode, _show, _store) {			\
	.attr = {.name = __stringify(_name),			\
	.mode = VERIFY_OCTAL_PERMISSIONS(_mode) },		\
	.show	= _show,							\
	.store	= _store,						\
}
```
#### Store and Show functions
Each ``sysfs`` attribute should have one or both ``show`` and ``store`` defined.

The ``store`` function will be called whenever we are writing something to the sysfs attribute.

The ``show`` function will be called whenever we are reading the sysfs attribute.
#### Create sysfs file
Using this API to create a single ``sysfs`` file:
```c
int sysfs_create_file ( struct kobject *  kobj, const struct attribute * attr);
```
+ ``kobj`` – object we’re creating for.

+ ``attr`` – attribute descriptor.

Using this API to create a group of attributes:
```c
int sysfs_create_group(struct kobject *kobj, const struct attribute_group *grp_attr);
```
+ ``kobj`` – object we’re creating for.

+ ``grp_attr`` – group of attributes descriptor.

Once you have done with the ``sysfs`` file, you should delete this file using:
```c
void sysfs_remove_file(struct kobject *kobj, const struct attribute *attr);
```
+ ``kobj`` – object we’re creating for.

+ ``attr`` – attribute descriptor.


## Our code
The code located in folder 07_sys_filesystem
```bash
# To build ko driver 
make build-07
# To clean ko driver 
make clean-07
# To upload ko driver to our board 
make upload-07
```
Result
```
[root@luckfox root]# ls | grep sys
sys_fs.ko
[root@luckfox root]# insmod sys_fs.ko
[root@luckfox root]# ls /sys/kernel/ | grep my
my_sys_fs_module
[root@luckfox root]# ls /sys/kernel/my_sys_fs_module/
our_buffer
[root@luckfox root]# echo "Hello" > /sys/kernel/my_sys_fs_module/our_buffer
[root@luckfox root]# cat /sys/kernel/my_sys_fs_module/our_buffer
Hello
[root@luckfox root]# ls -la /sys/kernel/my_sys_fs_module/our_buffer
-rw-rw----    1 root     root          4096 Mar 19 18:02 /sys/kernel/my_sys_fs_module/our_buffer
```
As we can see the mode we have set is 0660, which is -rw-rw---- for our_buffer.