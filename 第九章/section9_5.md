## 9.5 块驱动程序

&emsp;&emsp;块驱动程序提供了对面向块的设备的访问，这种设备以随机访问的方式传输数据，并且数据总是具有固定大小的块。典型的块设备是磁盘驱动器，当然也有其它类型的块设备。

&emsp;&emsp;块设备和字符设备有很大的区别。比如块设备上可以mount文件系统，而字符设备是不可以的。——显然这是随即访问带来的优势，因为文件系统需要能按块存储数据，同时更需要能随即读写数据。

&emsp;&emsp;另外数据经过块设备相比操作字符设备需要多经历一个 **数据缓冲层(buffer cache
mechanism)**，也就是说应用程序与块设备传递数据时不同于操作字符设备那样直接打交道，而必须经过一个中间缓冲层来存储数据，然后才可使用数据。为什么需要这个“多余的”缓冲层呀？这绝非是画蛇添足之举。

&emsp;&emsp;追根遡源，提高系统整体性能（吞吐量）是其根本原因。系统运行快慢受文件系统访问速度直接影响，而文件系统的访问行为往往是大量无序的，而且常常会重复的访问请求。无序访问请求会让磁头（假设访问磁盘）不断改变方向（比直线运动费时的多）；重复访问又会使得上次读出的数据再被读取（前次的结果被白白浪费了）。

&emsp;&emsp;为了解决上述两个弊端，内核对块设备访问引入了缓冲层，缓冲层作用一是作为数据缓冲区，存储已取得的数据，以便加快访问速度——如果需要从块设备读取的数据已经在缓冲中，则使用缓冲中的数据，避免了耗时的设备操作和I/O操作；二是缓冲将对块设备的I/O访问按照访问扇区的位置进行了优化排序，尽量保证访问时磁头直线移动——这在系统中常被称为 **电梯调度算法（elevator algorithm）**。

&emsp;&emsp;字符驱动程序的接口相对清晰而且易于使用，但是块驱动程序的接口要稍微复杂一些。出现这种情况的原因有两个：一是因为其历史—块驱动程序接口从Linux第一个版本开始就一直存在于每个版本中，并且已经证明很难修改或改进；其二是因为性能，一个慢的字符设备驱动程序虽然不受欢迎，但仍可以接受，但一个慢的块驱动程序将影响整个系统的性能。因此，块驱动程序的接口设计经常受到速度要求的影响。

&emsp;&emsp;本节利用一个示例驱动程序讲述块驱动程序的创建。这个驱动程序称为 Mysbd（My Simple
Block Driver）， Mysbd实现了一个使用系统内存的块设备，从本质上讲，属于一种 RAM
磁盘驱动程序, 因为块驱动程序的复杂性, Mysbd只给出实现块驱动程序的框架。

&emsp;&emsp;Mysbd作为一个内存块设备，对其设备结构的定义如下：

```c
    struct Mysbd_dev {
            void **data; /*存放数据的内存地址*/
            unsigned long size; /*块大小*/
            unsigned int lock; /*用于加锁*/
            unsigned int new_msg; /*标志*/
            struct Mysbd_dev * next; /*链表中下一个元素*/
    };
    extern struct Mysbd_dev *Mysbd; /*设备信息*/
```


### 9.5.1 块驱动程序的注册

&emsp;&emsp;和字符驱动程序一样,内核使用主设备号来标识块驱动程序但块主设备号和字符主设备号是互不相干的。一个主设备号为32的块设备和具有相同主设备号的字符设备可以同时存在，因为它们具有各自独立的主设备号分配空间。

&emsp;&emsp;用来注册和注销块设备驱动程序的函数，与用于字符设备的函数看起来很类似，如下所示:
```c
    #include <Linux/fs.h>
    int register_blkdev(unsigned int major, const char *name,
    struct block_device_operations *bdops);
    int unregister_blkdev(unsigned int major, const char *name);
```
&emsp;&emsp;上述函数中的参数意义和字符设备几乎相同，而且可以通过一样的方式动态赋予主设备号。因此，注册Mysbd的具体程序片段如下：

```c
    result = register_blkdev(Mysbd_major, "Mysbd", &Mysbd_bdops);
    if (result < 0) {
            printk(KERN_WARNING "Mysbd: can't get major %d\n",Mysbd_major);
            return result;
    }
    if (Mysbd_major == 0) Mysbd_major = result; /* 动态分配主设备号 */
```

&emsp;&emsp;然而，类似之处到此为止。我们已经看到了一个明显的不同：register\_chrdev()使用一个指向 file\_operations 结构的指针，而 register\_blkdev ( )则使用block\_device\_operations 结构的指针，该结构就是块驱动程序接口，其定义如下： 
```c 
    struct block_device_operations {
    int (*open) (struct inode *, struct file *); /*打开块设备文件*/
    int (*release) (struct inode *, struct file *); /*关闭对块设备文件的最后一个引用 */
    int (*ioctl) (struct inode *, struct file *, unsigned, unsigned long);
    /*在块设备文件上发出ioctl()系统调用*/
    int (*check_media_change) (kdev_t); /*检查介质是否已经变化（如软盘）*/int (*revalidate) (kdev_t); /*检查块设备是否持有有效数据*/
    };
```
&emsp;&emsp;这里列出的 open、release 和 ioctl方法和字符设备的对应方法相同。其它两个方法是块设备所特有的。  

&emsp;&emsp;Mysbd 使用的 bdops 接口定义如下：
```c
    struct block_device_operations Mysbd_bdops = {
            open: Mysbd_open,
            release: Mysbd_release,
            ioctl: Mysbd_ioctl,
            check_media_change: Mysbd_check_change,
            revalidate: Mysbd_revalidate,
    };
```

&emsp;&emsp;请读者注意，block\_device\_operations 接口中没有read 或者write操作。所有涉及到块设备的 I/O 通常由系统进行缓冲处理，用户进程不会对这些设备执行直接的 I/O操作。在用户模式下对块设备的访问，通常隐含在对文件系统的操作当中，而这些操作能够从I/O 缓冲当中获得明显的好处。但是，对块设备的“直接”I/O访问，比如在创建文件系统时的 I/O 操作，也一样要通过 Linux 的缓冲区缓存。为此，内核为块设备提供了一组单独的读写函数generic\_file\_read( )和generic\_file\_write ( )，驱动程序不必理会这些函数。  

&emsp;&emsp;显然，块驱动程序最终必须提供完成实际块 I/O 操作的机制。在 Linux
当中，用于这些 I/O 操作的方法称为“request（请求）”。request
方法同时处理读取和写入操作，因此要复杂一些。我们稍后将讲述 request。  

&emsp;&emsp;但在块设备的注册过程中，我们必须告诉内核实际的 request方法。然而，该方法并不在 block\_device\_operations结构中指定（这出于历史和性能两方面的考虑），相反，该方法和用于该设备的挂起I/O操作队列关联在一起。默认情况下，对每个主设备号并没有这样一个对应的队列。块驱动程序必须通过blk\_init\_queue 初始化这一队列。队列的初始化和清除接口定义如下：

```c  
    #include <Linux/blkdev.h>
    blk_init_queue(request_queue_t *queue, request_fn_proc *request);
    blk_cleanup_queue(request_queue_t *queue);
```

&emsp;&emsp;blk\_init\_queue()函数建立请求队列，并将该驱动程序的 request函数（通过第二个参数传递）关联到队列。在模块的清除阶段，调blk\_cleanup\_queue() 函数。Mysbd 驱动程序使用下面的代码行初始化它的队列：  
```c
blk\_init\_queue(BLK\_DEFAULT\_QUEUE(major), Mysbd\_request);  
```
&emsp;&emsp;每个设备有一个默认使用的请求队列，必要时，可使用 BLK\_DEFAULT\_QUEUE(major)宏得到该默认队列。这个宏在 blk\_dev\_struct 结构形成的全局数组（该数组名为blk\_dev）中搜索得到对应的默认队列。blk\_dev数组由内核维护，并可通过主设备号索引。blk\_dev\_struct 结构定义如下：
```c
    struct blk_dev_struct {
            request_queue_t request_queue;
            queue_proc* queue;
            void* data;
    };
```
&emsp;&emsp;request\_queue 域包含了初始化之后的 I/O
请求队列，我们将很快看到该队列的域。此外，还有个函数指针queue，当这个指针为非空时，就调用这个函数来找到具体设备的请求队列，这是为具有同一主设备号的多种同类设备而设的一个域。data
域可由驱动程序使用，以便保存一些私有数据，但很少有驱动程序使用该域。

&emsp;&emsp;图 9.7说明了注册和注销一个驱动程序模块时所要调用的函数及数据结构关系图：

1.  加载模块时，insmod命令调用init\_module()函数，该函数调用register\_blkdev()和blk\_init\_queue()分别进行驱动程序的注册和请求队列的初始化。

2.  register\_blkdev()把块驱动程序接口block\_device\_operations加入blkdevs[]表中。

3.  blk\_init\_queue()初始化一个默认的请求队列，将其放入blk\_dev[]表中，并将该驱动程序的
    request 函数关联到该队列。

4.  卸载模块时，rmmod命令调用cleanup\_module()函数，该函数调用unregister\_blkdev()和blk\_cleanup\_queue()分别进行驱动程序的注销和请求队列的清除。

<div align=center>
<img src="图9_7.jpg"/>        
</div>

<div align=center>
图 9.7：注册块设备驱动程序        
</div>


### 9.5.2 块设备请求

&emsp;&emsp;在内核安排一次数据传输时，它首先在一个表中对该请求排队，并以最大化系统性能为原则进行排序。然后，请求队列被传递到驱动程序的
&emsp;&emsp;request 函数，该函数的原型如下： 
```c 
    void request_fn (request_queue_t *queue);
```
&emsp;&emsp;块设备的读写操作都是由request()函数完成。对于具体的块设备，函数request()当然是不同的。所有的读写请求都存储在request结构的链表中。内核定义了request结构中，下面只列出与驱动程序相关的域：

```c
    struct request {
            ...
            kdev_t rq_dev; /*请求所访问的设备*/
            int cmd; /* 要执行的操作，READ 或 WRITE */
            unsigned long sector; /* 本次请求要传输的第一个扇区编号*/
            unsigned long current_nr_sectors; /*当前请求要传输的扇区数目 */
            char * buffer; /*数据要被写入或者要被读出的缓冲区。 */
            struct buffer_head * bh; /*本次请求对应的缓冲区链表的第一个缓冲区，即缓冲区
            头。*/
            ...
    };
```


&emsp;&emsp;内核定义了一个CURRENT宏，它是一个指向当前请求的指针，request()函数利用该宏检查当前的请求。

&emsp;&emsp;下面给出一个并不进行实际数据传输的最小 request 函数，应该如下定义：
```c
    void Mysbd _request(request_queue_t *q)
    {
            while(1) {
                    INIT_REQUEST;
                    printk("<1>request %p: cmd %i sec %li (nr. %li)\n", CURRENT,
                    CURRENT->cmd,
                    CURRENT->sector,
                    CURRENT->current_nr_sectors);
                    end_request(1); /* 成功 */
            }
    }
```

&emsp;&emsp;request()函数从INIT\_REQUEST宏命令开始（它定义在blk.h中），它对请求队列进行检查，保证请求队列中至少有一个请求在等待处理。如果没有请求（即CURRENT > = 0），INIT\_REQUEST宏命令将使request()函数返回，任务结束。

&emsp;&emsp;假定队列中至少有一个请求，request()函数现在应处理队列中的第一个请求，当处理完请求后，request()函数将调用end\_request()函数。如果成功地完成了读写操作，应该用参数值1调用end\_request()函数；如果读写操作不成功，以参数值0调用end\_request()函数。如果队列中还有其他请求，将CURRENT指针设为指向下一个请求。执行end\_request()函数后，request()函数回到循环的起点，对下一个请求重复上面的处理过程。

&emsp;&emsp;Mysbd 设备中能够完成实际工作的 request 函数如下：  
```c
    void Mysbd_request(request_queue_t *q)
    {
            Mysbd_Dev *device;
            int status;
            while(1) {
                    INIT_REQUEST; /* 当请求队列为空时返回 */
                    /* 检查我们正在用的是哪一个设备*/
                    device = Mysbd_locate_device (CURRENT);
                    if (device == NULL) {
                            end_request(0);
                            continue;
                    }
            /* 数据传送并进行清理 */
            spin_lock(&device->lock);
            status = Mysbd_transfer(device, CURRENT);
            spin_unlock(&device->lock);
            end_request(status);
            }
    }
```
&emsp;&emsp;上面的代码和前面给出的空 request
函数几乎没有什么不同，该函数本身集中于请求队列的管理上，而将实际的工作交给其它函数完成。第一个函数是
Mysbd\_locate\_device()，它检索请求当中的设备编号，并找出正确的 Mysbd\_Dev
结构；第二个函数是Mysbd\_transfer()，它完成实际的I/O请求：

```c
    static int Mysbd_transfer(Mysbd_Dev *device, const struct request *req)
    {
            int size; /*要传送的数据大小*/
            u8 *ptr; /*指向存放数据的内存起始地址*/
            ...
            /* 进行传送 */
            switch(req->cmd) {
            case READ:memcpy(req->buffer, ptr, size); /*从 Mysbd到缓冲区 */
            return 1;
            case WRITE:
            memcpy(ptr, req->buffer, size); /* 从缓冲区到 Mysbd */
            return 1;
            default:
            /* 不可能发生 */
            return 0;
            }
    }
```
&emsp;&emsp;因为 Mysbd 只是一个 RAM 磁盘，因此，该设备的“数据传输”只是一个 memcpy 调用而已。

&emsp;&emsp;块设备驱动程序初始化时，由驱动程序的init()完成。为了引导内核时调用init()，需要在blk\_dev\_init()函数中增加一行代码Mysbd\_init()。

&emsp;&emsp;块设备驱动程序初始化的工作主要包括：

（1）检查硬件是否存在；

（2）登记主设备号；

（3）利用register\_blkdev()函数对设备进行注册：

（4）将块设备驱动程序的数据容量传递给缓冲区：
```c
    #define Mysbd_HARDS_SIZE 512
    #define Mysbd_BLOCK_SIZE 1024
    static int Mysbd_hard = Mysbd_HARDS_SIZE;
    static int Mysbd_soft = Mysbd_BLOCK_SIZE;
    hardsect_size[Mysbd_MAJOR] = &Mysbd_hard;
    blksize_size[Mysbd_MAJOR] = &Mysbd_soft;
```

&emsp;&emsp;在块设备驱动程序内核编译时，应把下列宏加到blk.h文件中：
```c
    #define MAJOR_NR Mysbd_MAJOR
    #define DEVICE_NAME “Mysbd”
    #define DEVICE_REQUEST Mysbd_request
    #define DEVICE_NR(device) (MINOR(device))
    #define DEVICE_ON(device)
    #define DEVICE_OFF(device)
```
（5）将request()函数的地址传递给内核：
```c
    blk_dev[Mysbd_MAJOR].request_fn = DEVICE_REQUEST;
```

&emsp;&emsp;以上只是一个简单的内存块设备驱动程序的示例,实际块驱动程序要比这复杂多，比较相近的例子如上一章讨论的romfs文件系统，在此不进一步讨论。

&emsp;&emsp;关于驱动程序更详细的内容请参看《Linux设备驱动程序》一书。
