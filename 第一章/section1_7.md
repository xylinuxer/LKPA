## **1.7  Linux 内核中链表的实现及应用**

&emsp;&emsp;链表是Linux内核中最简单、最常用的一种数据结构。与数组相比，链表中可以动态插入或删除元素，在编译时不必知道要创建的元素个数。也因为链表中每个元素的创建时间各不相同，因此，它们在内存无需占用连续的内存单元，因为单元的不连续，因此各元素需要通过某种方式被链接在一起，于是，每个元素都包含一个指向下一个元素的指针，当有元素加入链表或者从链表中删除元素时，只需要调整下一个节点的指针就可以了。

### **1.7.1 链表的演化**

&emsp;&emsp;在C 语言中，一个基本的双向链表定义如下：

	struct my_list {
		void *mydata;
		struct my_list *next;
		struct my_list *prev;
	}

![](http://i.imgur.com/d3wmDTM.png)
   
&emsp;&emsp;图1.6是一双链表，通过前趋（prev）和后继（next）两个指针域，就可以从两个方向遍历双链表，这使得遍历链表的代价减少。如果打乱前驱、后继的依赖关系，就可以构成"二叉树"；如果再让首节点的前趋指向链表尾节点、尾节点的后继指向首节点（如图1.6中虚线部分），就构成了循环链表；如果设计更多的指针域，就可以构成各种复杂的树状数据结构。

&emsp;&emsp;如果减少一个指针域，就退化成单链表，如果只能对链表的首尾进行插入或删除操作，就演变为队结构，如果只能对链表的头进行插入或删除操作，就退化为栈结构。
  
### **1.7.2. 链表的定义和操作**

&emsp;&emsp;如上所述，在众多数据结构中，选取双向链表作为基本数据结构，并将其嵌入到其他数据结构中，从而可以演化出其他复杂数据结构。Linux内核实现方式与众不同，对链表给出了一种抽象的定义。

**1. 链表的定义**

	struct list_head {
		struct list_head *next, *prev;
	};   
                                           
&emsp;&emsp;这个不含数据域的链表，可以嵌入到任何结构中，例如可以按如下方式定义含有数据域的链表： 

	struct my_list{ 
		void *mydata; 
		struct list_head list;
	};  
 
&emsp;&emsp;在此，进一步说明几点：

&emsp;&emsp;1）list域隐藏了链表的指针特性。

&emsp;&emsp;2）struct list_head可以位于结构的任何位置，可以给其起任何名字

&emsp;&emsp;3）在一个结构中可以有多个list 域

&emsp;&emsp;以struct list_head为基本对象，对链表进行插入、删除、合并以及遍历等各种操作。

**2 链表的声明和初始化宏**

&emsp;&emsp;实际上， struct list_head只定义了链表节点，并没有专门定义链表头，那么一个链表结构是如何建立起来的？内核代码list.h中定义了两个宏：

	#define LIST_HEAD_INIT(name) { &(name), &(name) }	/*仅初始化*/
	
	#define LIST_HEAD(name) \
		struct list_head name = LIST_HEAD_INIT(name)    /*声明并初始化*/

&emsp;&emsp;如果我们要申明并初始化自己的链表头mylist，则直接调用LIST_HEAD：

	LIST_HEAD(mylist_head)

&emsp;&emsp;调用之后，mylist\_head的next、prev指针都初始化为指向自己，这样，我们就有了一个空链表，如何判断链表是否为空，自己写一下这个简单的函数list_empty ，也就是让头指针的next指向自己。


**3 在链表中增加一个结点**

&emsp;&emsp;list.h中增加节点的函数为：

	static inline void __list_add();
	static inline void list_add();
	static inline void list_add_tail();

&emsp;&emsp;在内核代码中，函数名前加两个下划线表示内部函数，第一个函数的具体代码如下：

	static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
	{
		next->prev = new;
		new->next = next;
		new->prev = prev;
		prev->next = new;
	}

&emsp;&emsp;调用这个内部函数以分别在链表头和尾增加节点：

	static inline void list_add(struct list_head *new, struct list_head *head)
	{
		__list_add(new, head, head->next);
	}

&emsp;&emsp;该函数向指定链表的head节点后插入new节点。因为链表是循环的，而且通常没有首尾节点的概念，所以可以将任何节点传递给head。但是如果传递最后一个元素传给head，那么该函数可以用来实现一个栈。


	static inline void list_add_tail(struct list_head *new, struct list_head *head)
	{
		__list_add(new, head->prev, head);
	}


&emsp;&emsp;该函数向指定链表的head节点前插入new节点。和list_add()函数类似，因为链表是环形的，而且可以将任何节点传递给head。但是如果传递第一个元素给head那么，该函数可以用来实现一个队列。

&emsp;&emsp;另外，对函数名前面的staitic inline 关键字给予说明。“static”加在函数前，表示这个函数是静态函数，所谓静态函数，实际上是对函数作用域的限制，指该函数的作用域仅局限于本文件。所以说，static具有信息隐藏的作用。而关键字"inline“加在函数前，说明这个函数对编译程序是可见的，也就是说，编译程序在调用这个函数时就立即展开该函数。所以，关键字inline 必须与函数定义体放在一起才能使函数成为内联。inline函数一般放在头文件中。

&emsp;&emsp;关于节点的删除，将结合后面的例子给予具体说明。至于节点的搬移和合并，读者自行分析，在此不一一讨论，下面主要分析链表的遍历。


**4. 遍历链表**

&emsp;&emsp;list.h中定义了如下遍历链表本的宏：

	#define list_for_each(pos, head) \
		for (pos = (head)->next; pos != (head); pos = pos->next)

&emsp;&emsp;这种遍历仅仅是找到一个个节点在链表中的偏移位置pos，如图1.7所示。

![](http://i.imgur.com/X4InkHq.png)

&emsp;&emsp;问题在于，如何通过pos获得节点的起始地址，从而可以引用节点中的域？ 于是 list.h中定义了晦涩难懂的list_entry（）宏：

	#define list_entry(ptr, type, member) \
		container_of(ptr, type, member)



&emsp;&emsp;指针ptr指向结构体type中的成员member；通过指针ptr，返回结构体type的起始地址，也就是list_entry返回指向type类型的指针，如图1.8所示。

![](http://i.imgur.com/L6Bc8tu.png)

&emsp;&emsp;进一步仔细分析list_entry（）宏：

((unsigned long) &(type *)0)->member)把0地址转化为type结构的指针，然后获取该结构中member域的指针，也就是获得了member在type结构中的偏移量。其中(char *)(ptr)求出的是ptr的绝对地址，二者相减，于是得到type类型结构体的起始地址，如图1.9所示。

![](http://i.imgur.com/8NMNQjb.png)

&emsp;&emsp;到此，我们对链表的实现机制有初步了解，更多的函数和实现查看[include/linux/list.h ](include/linux/list.h )中的代码。如何应用这些函数，下面举例说明。尽管list.h是内核代码中的头文件，但我们稍加修改后也可以把它移植到用户空间使用。

### **1.7.3 链表的应用**

&emsp;&emsp;[linclude/linux/list.h](linclude/linux/list.h)中的函数和宏，是一组精心设计的接口，有比较完整的注释和清晰的思路，请详细阅读该文件中的代码。

&emsp;&emsp;下面编写一个linux 内核模块，用以创建、增加、删除和遍历一个双向链表。

	#include <linux/kernel.h>
	#include <linux/module.h>
	#include <linux/slab.h>
	#include <linux/list.h>

	MODULE_LICENSE("GPL");
	MODULE_AUTHOR("XIYOU");

	#define N 10   //链表节点
	struct numlist {
		int num;//数据
		struct list_head list;//指向双联表前后节点的指针
	};

	struct numlist numhead;//头节点

	static int __init doublelist_init(void)
	{
		//初始化头节点
		struct numlist *listnode;//每次申请链表节点时所用的指针
		struct list_head *pos;
		struct numlist *p;
		int i;

		printk("doublelist is starting...\n");
		INIT_LIST_HEAD(&numhead.list);

		//建立N个节点，依次加入到链表当中
		for (i = 0; i < N; i++) {
			listnode = (struct numlist *)kmalloc(sizeof(struct numlist), GFP_KERNEL); // kmalloc（）在内核空间申请内存，类似于malloc,参见第四章
			listnode->num = i+1;
			list_add_tail(&listnode->list, &numhead.list);
			printk("Node %d has added to the doublelist...\n", i+1);
		}

		//遍历链表
		i = 1;
		list_for_each(pos, &numhead.list) {
			p = list_entry(pos, struct numlist, list);
			printk("Node %d's data:%d\n", i, p->num);
			i++;
		}
	
		return 0;
	}

	static void __exit doublelist_exit(void)
	{
		struct list_head *pos, *n;
		struct numlist *p;
		int i;
	
		//依次删除N个节点
		i = 1;
		list_for_each_safe(pos, n, &numhead.list) {  //为了安全删除节点而进行的遍历
			list_del(pos);//从双链表中删除当前节点
			p = list_entry(pos, struct numlist, list);//得到当前数据节点的首地址，即指针
			kfree(p);//释放该数据节点所占空间
			printk("Node %d has removed from the doublelist...\n", i++);
		}
		printk("doublelist is exiting..\n");
    	}

	module_init(doublelist_init);
	module_exit(doublelist_exit);

&emsp;&emsp;说明：关于删除元素的安全性问题

&emsp;&emsp;在上面的代码中，为什么不调用list\_for\_each（）宏而调用 list\_for\_each_safe（）进行删除前的遍历？具体看删除函数的源代码：

	static inline void __list_del(struct list_head * prev, struct list_head * next)
	{
		next->prev = prev;
		prev->next = next;
	}

	static inline void list_del(struct list_head *entry)
	{
		__list_del(entry->prev, entry->next);
		entry->next = LIST_POISON1;
		entry->prev = LIST_POISON2;
	}

&emsp;&emsp;可以看出，当执行删除操作的时候， 被删除的节点的两个指针被指向一个固定的位置（LIST\_POISON1和LIST\_POISON2是内核空间的两个地址）。而list\_for\_each(pos, head)中的pos指针在遍历过程中向后移动，即pos = pos->next，如果执行了list\_del()操作，pos将指向这个固定位置的next, prev,而此时的next, prev没有任何指向了，必然出错。

&emsp;&emsp;而list\_for\_each_safe(p, n, head) 宏解决了上面的问题：

	#define list_for_each(pos, head) \
		for (pos = (head)->next; pos != (head); pos = pos->next)

&emsp;&emsp;它采用了一个同pos同样类型的指针n 来暂存将要被删除的节点指针pos，从而使得删除操作不影响pos指针！ 

&emsp;&emsp;这里要说明的是，哈希表也是链表的一种衍生，在list.h中也有相关的代码，在此不仔细讨论，读者可自行分析。


