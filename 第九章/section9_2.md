## 9.2 设备驱动程序框架

由于设备种类繁多，相应的设备驱动程序也非常之多。尽管设备驱动程序是内核的一部分，但设备驱动程序的开发往往由很多人来完成，如业余编程高手、设备厂商等。为了让设备驱动程序的开发建立在规范的基础上，就必须在驱动程序和内核之间有一个严格定义和管理的接口，例如SVR4提出了DDI/DDK规范，其含义就是设备与驱动程序接口／设备驱动程序与内核接口（Device-Driver  
Interface／Driver-Kernel  
Interface）。通过这个规范，可以规范设备驱动程序与内核之间的接口。

Linux的设备驱动程序与外接的接口与DDI/DKI规范相似，可以分为三部分：

1. 驱动程序与内核的接口，这是通过数据结构file\_operations来完成的。

2. 驱动程序与系统引导的接口，这部分利用驱动程序对设备进行初始化。

3. 驱动程序与设备的接口，这部分描述了驱动程序如何与设备进行交互，这与具体设备密切相关。

其中第一点是驱动程序的核心部分，在此给予具体分析，至于后面两点，在具体的驱动程序中将会涉及到。

从上一章虚拟文件系统的介绍可以看出，设备是被纳入到文件系统的框架之下的，如第八章的图8.1。从上图9.1可以看出，用户进程是通过file结构与文件或者设备进行交互的，file结构的简化定义（具体定义于include/linux/fs.h）如下：

```c
struct file {
        ...
        const struct file_operations *f_op;
        ...
}
```

其中struct file\_operations是驱动程序主要关注的对象，其结构如下：

```c
struct file_operations{
        int (*open) (struct inode *, struct file *); /*打开*/
        int (*close) (struct inode *, struct file *); /*关闭*/
        loff_t (*llseek) (struct file *, loff\_t, int); /*修改文件当前的读写位置*/  
        ssize_t (*read) (struct file *, char *, size_t, loff_t *);
        /*从设备中同步读取数据*/  
        ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
        /*向设备中发送数据*/  
        int (*mmap) (struct file *, struct vm_area_struct *);
        /*将设备的内存映射到进程地址 空间*/  
        int(*ioctl) (struct inode *, struct file *,unsigned int ,unsigned long);
        /*执行设备上的I/O控制命令*/
        unsigned int (*poll) (struct file *,struct poll_table_struct *);
        /*轮询，判断是否可以进行非阻塞的读取或者写入*/  
        ...
}
```

可以看出，file\_operations结构中对文件操作的函数只给出了定义，至于实现，就留给具体的驱动程序完成，下面为字符设备驱动程序打开、读、写以及I/O控制函数的模板：

```c
static int char_open(struct inode *inode,struct file *file)  
{
        ...
}

ssize_t xxx_read(struct file *filp, char _user *buf, size_t
count, loff_t *f_pos)
{
        ...
        copy_to_user(buf,...,count);
        ...
}

ssize_t xxxwrite(struct file *filp, char _user *buf, size_t
count, loff_t *f_pos)
{
        ...
        copy_from_user(...,buf,count);
        ...
}

int xxx_ioctl(struct inode *inode,struct file *filp,unsigned int cmd,unsigned
long arg)
{

        switch(cmd)
        {
        case xxx_cmd1:
        ...
        break;

        case xxx_cmd2:
        ...
        break;
        default: /*不能支持的命令*/
        }
        return –enotty;
        }
```

在这些设备驱动函数中，filp是文件结构的指针,count是要读的字节数，f\_pos是读的位置相对于文件开头的偏移，buf是用户空间的内存地址，该地址在内核空间不能直接读写，因此要调用copy\_from\_user\(\)和copy\_to\_user\(\)函数进行跨空间的数据拷贝：

这两个函数的原型如下：

```c
unsigned long copy_from_user(void *to,count void _user *from,unsigned
long count);

unsigned long copy_to_user(void _user *to,count void *from,unsigned long
count);
```

通过以上的介绍，使读者对驱动程序的框架有一个初步了解，以以上框架为模板，看一个简单的字符驱动程序。

例 9-1 简单字符驱动程序mycdev.c

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/types.h>
#include <linux/fs.h>
#include <linux/mm.h>
#include <linux/sched.h>
#include <linux/cdev.h>
#include <asm/io.h>
#include <asm/switch_to.h>
#include <asm/uaccess.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");

#define MYCDEV_MAJOR 231 /\*给定的主设备号\*/
#define MYCDEV_SIZE 1024

static int mycdev_open(struct inode *inode, struct file *fp)
{
        return 0;
}

static int mycdev_release(struct inode *inode, struct file *fp)
{
        return 0;
}

static ssize_t mycdev_read(struct file *fp, char __user *buf, size_t size, loff_t *pos)
{
        unsigned long p = *pos;
        unsigned int count = size;
        char kernel_buf[MYCDEV_SIZE] = "This is mycdev!";
        int i;
        if (p >= MYCDEV_SIZE)
                return -1;
        if (count > MYCDEV_SIZE)
                count = MYCDEV_SIZE - p;
        if (copy_to_user(buf, kernel_buf, count) != 0) {
                printk("read error!\n");
        return -1;
}

        printk("reader: %d bytes was read...\n", count);
        return count;
}

static ssize_t mycdev_write(struct file *fp, const char __user *buf,
size_t size, loff_t *pos)
{
        return size;
}

/* 填充 mycdev的 file operation 结构*/

static const struct file_operations mycdev_fops =
{
        .owner = THIS_MODULE,
        .read = mycdev_read,
        .write = mycdev_write,
        .open = mycdev_open,
        .release = mycdev_release,
};

/*模块初始化函数*/

static int __init mycdev_init(void)
{
        int ret;
        printk("mycdev module is staring..\n");
        ret=register_chrdev(MYCDEV_MAJOR,"my dev",&mycdev_fops);
        /*注册驱动程序*/
        if (ret<0){
                printk("register failed..\n");
                return 0;
        }
        else{
                printk("register success..\n");
        }
        return 0;
}

/*模块卸载函数*/

static void __exit mycdev_exit(void)
{
        printk("mycdev module is leaving..\n");
        unregister_chrdev(MYCDEV_MAJOR,"my_cdev"); /*注销驱动程序*/
}

module_init(mycdev_init);

module_exit(mycdev_exit);
```

上机调试该程序，并按如下步骤进行模块的插入：

\(1\) 通过make编译mycdev.c模块，并把mycdev.ko插入到内核；

\(2\) 通过cat /proc/devices  
查看系统中未使用的字符设备主设备号，比如当前231未使用；

\(3\) 创建设备文件结点：sudo mknod /dev/mycdev c 231 0；具体使用方法通过man  
mknod命令查看；

\(4\) 修改设备文件权限：sudo chmod 777 /dev/mycdev；

\(5\) 通过dmesg查看日志信息

\(6\) 以上成功完成后，编写用户态测试程序；运行该程序查看结果；

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdlib.h>

int main()
{
        int testdev;
        int i, ret;
        char buf[10];

        testdev = open("/dev/mycdev", O_RDWR);

        if (testdev == -1) {
        printf("cannot open file.\n");
        exit(1);
}

if (ret = read(testdev, buf, 10) < 10) {
        printf("read error!\n");
        exit(1);
}

for (i = 0; i < 10; i++)
        printf("%d\n", buf[i]);
        close(testdev);
        return 0;
}
```



