## **1.5 Linux内核模块编程入门**

&emsp;&emsp;内核模块是Linux内核向外部提供的一个插口，其全称为动态可加载内核模块（Loadable Kernel Module，LKM），我们简称为模块。Linux内核之所以提供模块机制，是因为它本身是一个单内核（monolithic kernel）。单内核的最大优点是效率高，因为所有的内容都集成在一起，但其缺点是可扩展性和可维护性相对较差，模块机制就是为了弥补这一缺陷。

### **1.5.1	什么是模块**

&emsp;&emsp;模块是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行，这与运行在用户空间的进程是不同的。模块通常由一组函数和数据结构组成，用来实现一种文件系统、一个驱动程序或其他内核上层的功能。

### **1.5.2	编写一个简单的模块**

&emsp;&emsp;模块和内核都在内核空间运行，模块编程在一定意义上说就是内核编程。因为内核版本的每次变化，其中的某些函数名也会相应地发生变化，因此模块编程与内核版本密切相关。我们在本书中所涉及的内核编程，基于的内核为2.6.x，对于其他版本，还需要做一些适当调整。

**1．程序举例**

      #include <linux/module.h>
      #include <linux/kernel.h>
      #include <linux/init.h>

      static int __init lkp_init(void)
      {
	      printk("<1>Hello, World! from the kernel space...\n");
	      return 0;
      }

      static void __exit lkp_exit(void)
      {
	      printk("<1>Goodbye, World! leaving kernel space...\n");
      }

      module_init(lkp_init);
      module_exit(lkp_exit);
      MODULE_LICENSE("GPL");

**2．说明**

&emsp;&emsp;（1）module.h头文件中包含了对模块的结构定义以及模块的版本控制，任何模块程序的编写都要包含这个头文件; 头文件kernel.h包含了常用的内核函数；而头文件init.h包含了宏\_init和\_exit，宏_init告诉编译程序相关的函数和变量仅用于初始化，编译程序将标有 _init的所有代码存储到特殊的内存段中，初始化结束后就释放这段内存。

&emsp;&emsp;（2）函数lkp\_init ()是模块的初始化函数，函数lkp\_cleanup ()是模块的退出和清理函数。

&emsp;&emsp;（3）我们在这里使用了printk()函数，该函数是由内核定义的，功能与C库中的printf()类似，它把要打印的信息输出到终端或系统日志。字符串中的<1>是输出的级别，表示立即在终端输出。

&emsp;&emsp;（4）函数module\_init()和cleanup\_exit()是模块编程中最基本也是必须的两个函数。module\_init()向内核注册模块所提供的新功能，而cleanup_exit()注销由模块提供的所有功能。

&emsp;&emsp;（5）最后一句告诉内核该模块具有GNU公共许可证。

**3．编译模块**

&emsp;&emsp;假定我们给前面的程序起名为“hellomod.c”，只有超级用户才能加载和卸载模块。对于2.6内核的模块，其Makefile文件的基本内容如下：

	#Makefile3.10
	obj-m := hellomod.o                                          #产生hellomod 模块的目标文件
	CURRENT_PATH := $(shell pwd)                                 #模块所在的当前路径
	LINUX_KERNEL := $(shell uname -r)                            #Linux内核源代码的当前版本
	LINUX_KERNEL_PATH := /usr/src/kernels/$(LINUX_KERNEL)        #Linux内核源代码的绝对路径
	all:
		make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules   #编译模块
	clean:
		make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean     #清理
  
&emsp;&emsp;上面的Makefile中使用了 obj-m := 这个赋值语句，其含义说明要使用目标文件hellomod.o建立一个模块，最后生成的模块名是helloworld.ko，如果你有一个名为module.ko的模块依赖于两个文件 file1.o和file2.o，那么我们可以使用module-obj扩展，如下所示
 
	obj-m := module.o 
	module-objs := file1.o file2.o

&emsp;&emsp;关于Makefile的具体编写方法，请参考相关书籍。

&emsp;&emsp;最后，用make命令运行Makefile

**4.运行代码**  
&emsp;&emsp;当编译好模块，我们就可以将新的模块插入到内核中，这可以用insmod命令来实现，如下所示：

	insmod hellomod.ko

&emsp;&emsp;然后，可以用lsmod命令检查模块是否正确插入到内核中了：

&emsp;&emsp;模块的输出由printk()来产生。该函数默认打印系统文件/var/log/messages的内容。快速浏览这些消息可输入如下命令：

	tail /var/log/messages

&emsp;&emsp;这一命令打印日志文件的最后10行内容，可以看到我们的初始化信息：.
	
	...
	Mar  6 10:35:55  lkp1  kernel: Hello,World! from the kernel space...

&emsp;&emsp;使用rmmod命令，加上我们在insmod中看到的模块名，可以从内核中移除该模块（还可以看到退出时显示的信息）。如下所示：

	rmmod hellomod

&emsp;&emsp;同样，输出的内容也在日志文件中，如下所示：

	...
	Mar  6 12:00:05  lkp1  kernel: Hello,World! from the kernel space...

