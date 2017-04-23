##**3.3 Linux系统中进程的组织方式**

在一个系统中，通常可拥有数十个、数百个乃至数千个进程，相应地就有这么多PCB。为了能有效地对它们加以管理，应该用适当的方式将这些PCB组织起来。

###**3.3.1进程链表**

为了对给定类型的进程（例如在可运行状态的所有进程）进行有效的搜索，内核建立了几个进程链表。每个进程链表由指向进程PCB的指针组成。在task_struct结构中有如下的定义：

    struct task_struct {
    …
	    struct list_head tasks;
        char comm[TASK_COMM_LEN];/*可执行程序的名字（带路径）*/
    …
    }

因此，如图3.8的一个双向循环链表把所有进程联系起来，我们叫它为进程链表。

![](http://i.imgur.com/mopL3yO.png)

链表的头和尾都为init_task。init_task是0号进程的PCB，0号这个进程永远不会被撤消，它的PCB被静态地分配到内核数据段中，也就是说init_task的PCB是预先由编译器分配的，在运行的过程中保持不变，而其它PCB是在运行的过程中，由系统根据当前的内存状况随机分配的，撤消时再归还给系统。

自己可编写一个内核模块，打印进程的PID和进程名，模块中主要函数的代码如下：

    static int print_pid( void) 
    {
        struct task_struct *task,*p;
        struct list_head *pos;
        int count=0;
        printk("Hello World enter begin:\n");
        task=&init_task;
        list_for_each(pos,&task->tasks)
               {
               p=list_entry(pos, struct task_struct, tasks);
               count++;
               printk("%d--->%s\n",p->pid,p->comm);
               }
      printk(the number of process is:%d\n",count);
      return 0;
    }

需要注意的是，在一个拥有大量进程的系统中通过重复来遍历所有的进程是非常耗费时的。

###**3.3.2哈希表**

在有些情况下，内核必须能根据进程的PID导出对应的PCB。顺序扫描进程链表并检查PCB的pid域是可行但相当低效的。为了加速查找，引入了哈希表,于是要有一个哈希函数把PID转换成表的索引，Linux用一个叫做pid_hashfn的宏实现：

    #define pid_hashfn(x) \ 
    ((((x) >> 8) ^ (x)) & (PIDHASH_SZ - 1))

其中，PIDHASH_SZ为表中元素的最大个数，通过pid_hashfn这个函数，可以把进程的PID均匀地散列在哈希表中。
	
对于一个给定的pid，可以通过find_task_by_pid()函数快速地找到对应的进程:

    static inline struct task_struct *find_task_by_pid(int pid)
    {
        struct task_struct *p, **htable = &pidhash[pid_hashfn(pid)];
        for(p = *htable; p && p->pid != pid; p = p->pidhash_next);
               
        return p;
    }

其中pidhash是哈希表,  其定义为:`struct task_struct  *pidhash[PIDHASH_SZ]`

在数据结构课程中我们已经了解到，哈希函数并不总能确保PID与表的索引一一对应，两个不同的PID散列到相同的索引称为冲突。

Linux利用链地址法来处理冲突的PID，也就是说，每一表项是由冲突的PID组成的双向链表， task_struct 结构中有两个域pidhash_next 和 pidhash_pprev域来实现这个链表，同一链表中pid由小到大排列。如图3.9所示。

![](http://i.imgur.com/uWyaXD4.png)

###**3.3.3 就绪队列**

当内核要寻找一个新的进程在CPU上运行时，必须只考虑处于就绪状态的进程，因为扫描整个进程链表是相当低效的，所以把可运行状态的进程组成一个双向循环链表，也叫就绪（runqueue）。

就绪队列容纳了系统中所有准备运行的进程， 在task_struct结构中定义了双向链表。

    struct task_struct{
    …
    struct list_head run_list;
    …
    }


就绪队列的定义以及相关操作在/kernel/sched.c文件中：

    static LIST_HEAD(runqueue_head);  /*定义就绪队列头指针为runqueue_head*/

    add_to_runqueue()函数向就绪队列中插入进程的PCB。
    static inline void add_to_runqueue(struct task_struct * p)
    {       list_add_tail(&p->run_list, &runqueue_head);
        nr_running++;  /*就绪进程数加1*/
    };

    move_last_runqueue()函数从就绪队列中删除进程的PCB。
    static inline void move_last_runqueue(struct task_struct * p)
    {
       list_del(&p->run_list);
      list_add_tail(&p->run_list, &runqueue_head);
    };

以上讲述的进程组织方式，实际上大都是数据结构中数据的组织方式，因此，读者在阅读本书或者源代码的过程中，首先抓住事物的本质，找出熟悉的知识，然后，再去体会或应用已有的知识解决问题。

另外，以上是Linux 2.4中就绪队列简单的组织方式。为了让读者尽可能先掌握原理，因此本章在讨论相关内容的时候将会去繁存简，将其中最核心的调度理论和算法做以阐述。

###**3.3.4 等待队列**

如前说述，睡眠有两种相关的进程状态:TASK_INTERRUPTIBLE和TASK_ UNINTERRUPTIBLE。它们的唯一区别是处于TASK_UNINTERRUPTIBLE的进程会忽略信号，而处于TASK_INTERRUPTIBLE状态的进程如果接收到一个信号会被提前唤醒并响应该信号。两种状态的进程位于同一个等待队列上，等待某些事件，不能够运行。

等待队列在内核中有很多用途，尤其对中断处理、进程同步及定时用处更大。因为这些内容在以后的章节中讨论，我们只在这里说明，进程必须经常等待某些事件的发生，例如，等待一个磁盘操作的终止，等待释放系统资源，或等待时间走过固定的间隔。等待队列实现在事件上的条件等待，也就是说，希望等待特定事件的进程把自己放进合适的等待队列，并放弃控制权。因此，等待队队列是一组睡眠的进程，当某一条件变为真时，由内核唤醒它们。

**1. 等待队列的数据结构：**
  
在include/linux/wait.h中，对等待队列的定义如下：

    struct __wait_queue {
	     unsigned int flags;
       #define WQ_FLAG_EXCLUSIVE	0x01
	    void *private;
	    wait_queue_func_t func;
	    struct list_head task_list;
    };

    typedef struct _ _wait_queue wait_queue_t;

在内核代码中，以两个下划线为开头的标识符一般都是内核内部定义的。typefdef对内部定义重新封装。

在这个结构中，最主要的域是task_list,它把处于睡眠状态的进程链接成双向链表。睡眠是暂时的，把它唤醒继续运行才是目的。为此，设置了func域，该域指向唤醒函数，用以把等待队列中的进程唤醒：

    typedef int (*wait_queue_func_t)(wait_queue_t *wait, unsigned mode, int flags, void *key);

如何唤醒等待队列中的进程，还需进一步根据等待的原因进行归类。比如，因为争夺某个临界资源，有一组进程由此睡眠，那么在唤醒时，是把这一组全部唤醒还是唤醒其中一个？如果全部唤醒，但实际上只能有一个进程使用临界资源，其他进程还得继续回去睡眠，因此仅唤醒等待队列中的一个进程才有意义。结构中的flag域就是为了区分睡眠时的互斥进程和非互斥进程。对于互斥进程，flag的取值为1（#define WQ_FLAG_EXCLUSIVE	0x01），反之，取值为0。还有一个private域，是传递给func函数的参数。

**2.等待队列头**

  每个等待队列都有一个等待队列头（wait queue head），定义如下：

    struct _ _wait_queue_head {
         spinlock_t lock;
         struct list_head task_list;
    };
    typedef struct _ _wait_queue_head wait_queue_head_t;

因为等待队列是由中断处理程序和主要内核函数修改的，因此必须对其双向链表保护以免对其进行同时访问，因为同时访问会导致不可预测的后果（参见第七章）。通过lock自旋锁域进行同步，而task_list域是等待进程链表的头。如图3.10是等待队列以及队列头形成的双向链表。
![](http://i.imgur.com/dghsQ16.png)

**3. 等待队列的操作**
  
在使用一个等待队列前，首先对等待队列头和等待队列进行初始化，wait.h中定义了如下宏：


    #define __WAIT_QUEUE_HEAD_INITIALIZER(name) {                           \
      .lock           = __SPIN_LOCK_UNLOCKED(name.lock),              \
     .task_list      = { &(name).task_list, &(name).task_list } }
  
    #define DECLARE_WAIT_QUEUE_HEAD(name) \
       wait_queue_head_t name = __WAIT_QUEUE_HEAD_INITIALIZER(name)

这两个宏声明初并始化等待队列头name.

初始化等待队列中的一个元素，则调用如下函数：

    static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
    {
        q->flags = 0;
        q->private = p;
        q->func = default_wake_function;
    }

default_wake_function( )唤醒睡眠非互斥进程p，然后从等待队列链表中将其删除。
  
定义了一个等待进程后，必须把它插入等待队列。add_wait_queue( )把一个非互斥进程插入等待队列链表的第一个位置。add_wait_queue_exclusive( )把一个互斥进程插入等待队列链表的最后一个位置。remove_wait_queue( )从等待队列链表中删除一个进程。waitqueue_active( )检查一个给定的等待队列是否为空。

如何让等待特定条件的进程去睡眠，内核提供了多个函数。下面介绍最基本的睡眠函数sleep_on():

    void sleep_on(wait_queue_head_t *wq)
    {
         wait_queue_t wait;
         init_waitqueue_entry(&wait, current);
         current->state = TASK_UNINTERRUPTIBLE;
         add_wait_queue(wq,&wait); /*  wq指向当前队列的头  */
         schedule( );
         remove_wait_queue(wq, &wait);
    }

该函数把当前进程的状态设置为TASK_UNINTERRUPTIBLE，并把它插入到特定的等待队列。然后，调用调度程序，而调度程序重新调度另一个进程开始执行。当睡眠的进程被唤醒时，调度程序接着执行sleep_on( )，也就是紧接schedule（）的remove_wait_queue（）函数，把该进程从等待队列中删除。

如果要让等待的进程唤醒，就调用唤醒函数wake_up（），它让待唤醒的进程进入TASK_RUNNING状态。内核代码中，wake_up 定义为一个宏，实际上等价于下列代码片段：

    void wake_up(wait_queue_head_t *q)
    {
         struct list_head *tmp;
         wait_queue_t *curr;
         list_for_each(tmp, &q->task_list) {
              curr = list_entry(tmp, wait_queue_t, task_list);
              if (curr->func(curr, 
    TASK_INTERRUPTIBLE|TASK_UNINTERRUPTIBLE, 
                  0, NULL) && curr->flags)
                  break;
         }
    }

list_for_each宏扫描双向链表q->task_list中的所有项，即等待队列中的所有进程。对每一项，list_entry宏都计算wait_queue_t型变量（curr）对应的地址。这个变量的func域存放wake_up函数的地址，它试图唤醒由等待队列中的task_list域标识的进程。如果一个进程已经被有效地唤醒（函数返回1）并且进程是互斥的（curr->flags等于1），循环结束。因为所有的非互斥进程总是在双向链表的开始位置，而所有的互斥进程在双向链表的尾部，所以函数总是先唤醒非互斥进程然后再唤醒互斥进程。

