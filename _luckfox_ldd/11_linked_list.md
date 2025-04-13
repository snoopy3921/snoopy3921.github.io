---
layout: default
title: "11 linked list"
parent: Linux device driver on Luckfox
nav_order: 11
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 11 Linked list
From basic OS to very advanced OS, linked list is a fundamental data structure. Therefore in Linux kernel, linked list is used in many places. To reduce the amount of duplicated code, the kernel developers have created a standard implementation of circular, doubly-linked lists; others needing to manipulate lists are encouraged to use this facility, introduced in version 2.1.45 of the kernel.

This is structure of type ``list_head``, defined in ``<linux/list.h>``:
```c
struct list_head {
    struct list_head *next, *prev;
};
```
This structure helps link the elements of the list together. To use linked list with more complex node, just embed this struct into that node struct. For example: 
```c
struct my_list_node {
    struct list_head list;
    int value; 
};
```
And the linked list has it header, the header must be a standalone list_head. And empty therefor has only header, and number of items is zero.

![Linked list](/assets/linked_list.png)

To create a linked list, first is to create head node(header). These lines is to declare and init head node:

```c
struct list_head my_list;

INIT_LIST_HEAD(&my_list);
```

Or alternatively can do so, they are the same:

```c
LIST_HEAD(my_list);
```

This is because Linux defines macro ``LIST_HEAD`` internally like this:

```c
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```
{: .note }
A note here is now to work with linked list, we actually work with list_head, and we do all actions on this structure, not in our user-defined node. And linked list is circular.

### Create a node
I want to create a node with value, and my_list_node is the structure i've created.
```c
struct my_list_node{
     struct list_head list;     //linux kernel list implementation
     int val;
};
```
Then 
```c
my_list_node new_node; /*Declare my node*/

INIT_LIST_HEAD(&new_node.list) /*Init list_head for my node*/

new_node.val = 10 /*Add value for that node*/
```

### Add node to linked list after (head)node
After creating our node. We use this inline function to do add our node to linked list named ``my_list`` above:
```c
void list_add(struct list_head *new, struct list_head *head);
```
Params are the list_head, and this will insert ``new`` entry after ``head``. 

In our case:
```c
list_add(&new_node.list, &my_list);
```

If you pass a list_head structure that happens to be in the middle of the list somewhere, the new entry will go immediately after it. Since Linux lists are circular, the head of the list is not generally different from any other entry.

This API is useful for implementing ``stack``

### Add node to linked list before (head)node
```c
void list_add_tail(struct list_head *new, struct list_head *head);
```
Params are the list_head, and this will insert ``new`` entry before ``head``. 

In our case:
```c
list_add_tail(&new_node.list, &my_list);
```
### Remove node from linked list 
It will remove the entry node from the list. But it doesn’t free any memory space allocated for the entry node.
```c
void list_del(struct list_head *entry);
```

### Move Node in linked list 
It will remove that node and again insert it after the ``head`` node.
```c
inline void list_move(struct list_head *list, struct list_head *head);
```
Or insert it before the ``head`` node after remove.
```c
inline void list_move_tail(struct list_head *list, struct list_head *head);
```

### Test the linked list
1. Is this entry is the last:
```c
inline int list_is_last(const struct list_head *list, const struct list_head *head);
```
2. Is this list empty:
```c
inline int list_empty(const struct list_head *head);
```
3. Is this list singular:
```c
inline int list_is_singular(const struct list_head *head);
```

All of aboves return 1 if true, and 0 otherwise.

### Traverse Linked List
To walk through linked list, we actually walk through entries. And this API used to get the struct for that entry.
```c
list_entry(ptr, type, member);
```
+ ``ptr`` – the struct list_head pointer.

+ ``type`` – the type of the struct this is embedded in.

+ ``member`` – the name of the list_head within the struct.

Returns the structure that embeds entry.

In our case:
```c
struct my_list_node *node_ptr = list_entry(&my_list.list, struct my_list_node, list);
```

## Our code
The code located in folder 11_linked_list
```bash
# To build ko driver 
make build-11
# To clean ko driver 
make clean-11
# To upload ko driver to our board 
make upload-11
```
```
[root@luckfox root]# insmod ki_linked_list_drv.ko
[root@luckfox root]# dmesg | tail -5
[  268.755055] 8
[  282.025045] ki_drv driver initializing...
[  282.026587] ki_drv module inserted successfully.
[  282.028320] Executing dynamic workqueue function
[  282.028343] Executing static workqueue function
[root@luckfox root]# cat /dev/ki_drv_dev
[root@luckfox root]# dmesg | tail -12
[  353.834662] Linked list start -
[  353.834675] - 0
[  353.834697] - 1
[  353.834704] - 2
[  353.834711] - 3
[  353.834718] - 4
[  353.834724] - 5
[  353.834729] - 6
[  353.834735] - 7
[  353.834740] - 8
[  353.834745] - 9
[  353.834749]
```