## 7.2 内核同步措施

为了避免并发，防止竞争。内核提供了一组同步方法来提供对共享数据的保护。
Linux使用的同步机制可以说随着内核版本的不断发展而完善。从最初的原子操作，到后来的信号量，从大内核锁到现在的自旋锁。这些同步机制的发展伴随Linux从单处理器到对称多处理器的过度；伴随着从非抢占内核到抢占内核的过度。锁机制越来越有效，也越来越复杂。内核中同步的方法很多，本节主要介绍原子操作、自旋锁和信号量这三种同步措施。

### 7.2.1 原子操作

原子操作可以保证指令以*原子的方式*被执行，也就是执行过程不被打断。例如，第1节提到过的原子方式的加操作，通过把读取和增加变量的行为包含在一个单步中执行，从而防止了竞争发生，保证了操作结果总是一致的（假定i初值是1）：

**内核任务1** | **内核任务2**
-------       | ---
增加i(1-\>2)  |  ---
              |
              | 增加 i(2-\>3)

最后得到的是3，这显然是正确结果。两个原子操作绝对不可能同时地访问同一个变量，这样的加操作也就绝不可能引起竞争。

因此，Linux内核提供了一个专门的atomic\_t类型（一个原子访问计数器），其定义如下：

```c
typedef struct {
        int counter;
}atomic_t;
```

>注：许多读者很疑惑，为何将原子类型这样定义？实际在内核中有许多这样的定义，主要原因是这样可以让GCC在编译的时候加以更加严格的类型检查，防止原子类型变量被误操作（如果为普通类型，则普通的运算符也可以操作）。

Linux内核提供了一些专门的函数和宏，参见表7.1，这些函数和宏作用于atomic\_t类型的变量。

表7.1 Linux中的原子操作

**函数**                     |   **说明**
------                       |   ---
ATOMIC\_INIT(i)              |  在声明一个atomic\_t变量时, 将它初始化为i;
                             |
atomic\_read(v)              |   返回\*v。
                             |
atomic\_set(v,i)             |   把\*v置成i
                             |
atomic\_add(i,v)             |   给\*v增加i
                             |  
atomic\_sub(i,v)             |   从\*v中减去i
                             |
atomic\_sub\_and\_test(i, v) |   从\*v中减去i,如果结果为0，则返回1；否则，返回0
                             |
atomic\_inc(v)               |   把1加到 \*v
                             |
atomic\_dec(v)               |   从\*v减1
                             |
atomic\_dec\_and\_test(v)    |   从\*v减1，如果结果为0，则返回1；否则，返回0
                             |
atomic\_inc\_and\_test(v)    |   把1加到\*v，如果结果为0，则返回1；否则，返回0
                             |
atomic\_add\_negative(i, v)  |   把i加到\*v，如果结果为负，则返回1；否则，返回0
                             |
atomic\_add\_return(i, v)    |   把i加到\*v，并返回相加之后的值
                             |
atomic\_sub\_return(i, v)    |  从\*v中减去i,并返回相减之后的值

下面举例说明这些函数的用法：

定义一个atomic\_c类型的数据很简单，还可以定义时给它设定初值：
```c
atomic_t u; /*定义 u*/

atomic_t v = ATOMIC_INIT(0); /*定义 v 并把它初始化为0*/
```
对其操作：
```c
atomic_set(&v,4) /* v = 4 ( 原子地)*/

atomic_set(2,&v) /* v = v + 2 = 6 (原子地) */

atomic_inc(&v) /* v = v + 1 =7（原子地)*/
```
如果需要读取原子类型atomic_t的值，可以使用atomic_read()来完成：
```c
printk(“%d\n”,atomic_read(&v)); /* 会打印7*/
```
原子整数操作最常见的用途就是实现计数器。使用复杂的锁机制来保护一个单纯的计数器是很笨拙的，所以，开发者最好使用atomic_inc()和atomic_dec()这两个相对来说轻便一点的操作。还可以用原子整数操作原子地执行一个操作并检查结果。一个常见的例子是原子的减操作和检查。
```c
int atomic_dec_and_test(atomic_t*v)
```
这个函数让给定的原子变量减1，如果结果为0，就返回1；否则返回0。

Linux内核也提供了位原子操作，相应的操作函数有set_bit(nr,addr)，clear_bit(nr,addr)，test_bit(nr,addt)等，它被广泛的应用于内存管理，设备驱动，读者可自行仔细探究。

### 7.2.2 自旋锁

自旋锁是专为防止多处理器并行而引入的一种锁，它在内核中大量应用于中断处理等部分，而对于单处理器来说，可简单采用关闭中断的方式防止中断服务程序的并发执行。

自旋锁最多只能被一个内核任务持有，如果一个内核任务试图请求一个已被持有的自旋锁，那么这个任务就会一直进行忙循环，也就是旋转，等待锁重新可用。如果锁未被持有，请求它的内核任务便能立刻得到它并且继续执行。自旋锁可以在任何时刻防止多于一个的内核任务同时进入临界区，因此这种锁可有效地避免多处理器上并行运行的内核任务竞争共享资源。

事实上，设计自旋锁的初衷是在短期间内进行轻量级的锁定。一个被持有的自旋锁使得请求它的任务在等待锁重新可用期间进行自旋（特别浪费处理器时间），所以自旋锁不应该被持有时间过长。如果需要长时间锁定，最好使用信号量。

自旋锁的定义如下：
```c
typedef struct raw_spinlock {
        unsigned int slock;
} raw_spinlock_t;

typedef struct {  
        raw_spinlock_t raw_lock;
        ... 
} spinlock_t;
```

于是，使用自旋锁的基本形式如下：
```C
DEFINE_SPINLOCK(mr_lock); /* 定义一个自旋锁 */
spin_lock(&mr_lock);
/*临界区*/
spin_unlock(&mr_lock);
```

因为自旋锁在同一时刻最多只能由一个内核任务持有，所以一个时刻只允许有一个任务存在于临界区中。这点很好的满足了对称多处理机器需要的加锁服务。

在单处理器系统上，虽然在编译时锁机制被抛弃掉了，但在上面例子中仍需要关闭中断，以禁止中断服务程序访问共享数据。

自旋锁在内核中有许多变种，如对下半部而言，可以使用spin_lock_bh()来获得特定锁并且关闭下半部执行。开锁操作则由spin_unlock_bh()来执行；如果把临界区的访问从逻辑上可以清晰地分为读和写这种模式，那么可以使用读者/写者自旋锁，它们通过下面方法初始化：
```C
DEFINE_RWLOCK(mr_lock);
```
在读者的代码分支中使用如下函数：

```C
read_lock(&mr_rwlock);
/*只读临界区*/
read_unlock(&mr_rwlock);
```
在写者的代码分支中使用如下函数：
```C
write_lock(&mr_rwlock);
/*写临界区*/
write_unlock(&mr_rwlock);
```

简单的说，自旋锁在内核中主要用来防止多处理器中并行访问临界区，防止内核抢占造成的竞争。另外自旋锁不允许任务睡眠，持有自旋锁的任务睡眠会造成自死锁，这是因为睡眠有可能造成持有锁的内核任务被重新调度，从而再次申请自己已持有的锁。因此自旋锁能够在中断上下文<sup>[3]</sup>中使用。

[3] 所谓中断上下文是指内核在执行一个中断服务程序或下半部时所处的执行环境。

### 7.2.3 信号量

Linux中的信号量是一种睡眠锁。如果有一个任务试图获得一个已被持有的信号量时，信号量会将其推入等待队列，然后让其睡眠。这时处理器获得自由而去执行其它代码。当持有信号量的进程将信号量释放后，在等待队列中的一个任务将被唤醒，从而便可以获得这个信号量。

信号量是在1968年由Edsger Wybe DijKstra提出的，此后它逐渐成为一种常用的锁机制。信号量支持两个原子操作P()和V()，这两个名字来自荷兰语Proberen和Vershogen。前者做测试操作（字面意思是探查），后者叫做增加操作。后来的系统把这两种操作分别叫做down()和up()，Linux也遵从这种叫法。down()操作通过对信号量计数减1来请求获得一个信号量。如果结果是0或大于0，信号量锁被获得，任务就可以进入临界区了。如果结果是负数，任务会被放入等待队列，处理器执行其它任务。down()函数如同一个动词 **“降低（down）”**，一次down()操作就等于获取该信号量。相反，当临界区中的操作完成后，up()操作用来释放信号量，该操作也被称作是 **“提升（upping）”信号量** ，因为它会增加信号量的计数值。如果在该信号量上的等待队列不为空，处于队列中等待的任务在被唤醒的同时会获得该信号量。

信号量具有睡眠特性，这使得信号量适用于锁会被长时间持有的情况，因此只能在进程上下文中使用，而不能在中断上下文中使用，因为中断上下文是不能被调度的；另外当任务持有信号量时，不可以再持有自旋锁。下面说明信号量的定义和使用。

信号量的定义

内核中对信号量的定义如下：
```c
struct semaphore {
        spinlock_t lock;
        unsigned int count;
        struct list_head wait_list;  
}
```

其各个域的含义如下：

**count:**

存放unsigned
int类型的一个值。如果该值大于0，那么资源就是空闲的，也就是说，该资源现在可以使用。相反，如果count等于0，那么信号量是忙的，但没有进程等待这个被保护的资源。最后，如果count为负数，则资源是不可用的，并至少有一个进程等待资源。

**lock:**

自旋锁。这个字段是在linux内核高版本中加入的，为了防止多处理器并行造成错误。

**wait_list:**

存放等待队列链表的地址，当前等待资源的所有睡眠进程都放在这个链表中。当然，如果count大于或等于0，等待队列就为空。

>注意：在使用信号量的时候，不要直接访问semaphore结构体内的成员，而要使用内核提供的函数来操作。

1. 相关操作函数

为了满足各种不同的需求，Linux内核提供了丰富的操作函数，对于获取信号量的操作，有down(),down\_interrputible()等函数，其中down()的实现代码为：

```c
void down(struct semaphore *sem)
{
        unsigned long flags;
        spin_lock_irqsave(&sem->lock, flags);/*加锁，使信号量的操作在关闭中断的状态下进行，防止多处理器并发操作造成错误*/
        if (sem->count > 0)) /*如果信号量可用，则将引用计数减1 */
        	sem->count--;
        else /*如果无信号量可用，则调用_down()函数进入睡眠等待状态 */
        __down(sem);
        spin_unlock_irqrestore(&sem->lock, flags); /*对信号量的操作解锁*/
}
```

像down\_interrputible()等其他一些获取信号量的函数与down()类似，在此不一一解释。从上面的分析也可以看出，如果无信号量可用，则当前进程进入睡眠状态。其中的\_down()函数调用\_down\_common()，这是各种down操作的统一函数：

```c
static inline int __sched__down_common(struct semaphore *sem, long state,long timeout)
{
        struct task_struct *task = current;
        struct semaphore_waiter waiter;
        list_add_tail(&waiter.list, &sem->wait_list); /*将当前进程添加到信号量sem的等待队列的队尾*/
        waiter.task = task;
        waiter.up = 0;
        for (;;) {
                if (signal_pending_state(state, task)) /*如果当前进程被信号唤醒，则返回*/
        	       goto interrupted;
                if (timeout <= 0) /*如果等待超时，则返回*/
        	       goto timed_out;
                __set_task_state(task, state); /* 设置进程状态*/
                spin_unlock_irq(&sem->lock); /* 释放自选锁 */
                timeout = schedule_timeout(timeout); /*执行进程切换*/
                spin_lock_irq(&sem->lock); /*当进程被唤醒时，如果再次进入获取信号量操作，则对其进行加锁*/
                if (waiter.up) /*如果进程是被信号量等待队列的其他进程唤醒，则返回*/
                return 0;
        }
        timed_out: /* 进程获取信号量等待超时，返回 */
        list_del(&waiter.list);
        return -ETIME;
        interrupted: /*进程等待获取信号量时被信号中断，返回*/
        list_del(&waiter.list);
        return -EINTR;
}
```

其中semaphore_waiter的定义为：

```c
struct semaphore_waiter {
       struct list_head list;
       struct task_struct* task;
       int up;
};
```

对于不同的信号量获取函数，传递给\_down\_common()函数的参数是不同的，对于down()操作调用\_down\_common()的形式为：

```c
__down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
```

可见，使用down()操作获取信号量的时候，如果信号量不可获取，则进程进入睡眠等待状态，并且不可被信号中断。而down\_interruptible()调用\_down\_common()的形式为：
```c
_down_common(sem, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
```
可见，进程在等待获取信号量的时候时可以被信号打断的。其他获取信号量的操作函数形式基本类似。

当进程获取信号量，访问完临界区之后，需要释放信号量。释放信号量操作函数为up()：

```c
void up(struct semaphore *sem)
{
        unsigned long flags;
        spin_lock_irqsave(&sem->lock, flags); /* 对信号操作进行加锁 */
        if list_empty(&sem->wait_list) /*如果该信号量的等待队列为空，则释放信号量 */
                sem->count++;
        else /* 否则唤醒该信号量的等待队列队头的进程 */
                __up(sem);
        spin_unlock_irqrestore(&sem->lock, flags); /*对信号量操作进行解锁 */
}
```

2．信号量的使用

要使用信号量，需要包含头文件 < linux/semaphore.h >，其中有信号量的定义和操作函数。

 (1).信号量的创建和初始化

内核提供了多种方法来创建信号量。如果要定义一个互斥信号量，则可以使用：

DECLARE\_MUTEX(name);

参数name为要定义的信号量的名字。如果信号量已经被定义，只需将其进行初始化，则可以使用以下函数：

```c
sema_init(struct semaphore *sem, int val); /*
  初始化信号量sem的使用者数量为val */
init_MUTEX(sem); /*初始化信号量sem为未锁定的互斥信号量 */
init_MUTEX_LOCKED(sem); /* 初始化信号量sem为锁定的互斥信号量 */
```

(2).信号量的使用

 信号量的一般使用形式为：

```c
static DECLARE_MUTEX(mr_sem);/*声明并初始化互斥信号量*/
if(down_interruptible(&mr_sem))
/*信号被接收 , 信号量还未获取*/
/*临界区…*/
	up(&mr_sem);
```


函数down\_interruptible()试图获取指定的信号量，如果获取失败，它将以TASK\_INTERRUPTIBLE状态睡眠。回忆第三章的内容，这种进程状态意味着任务可以被信号唤醒，一般来说这是件好事。如果进程在等待获取信号量的时候接收到了信号，那么该进程就会被唤醒，而函数down\_interruptible()会返回–EINTR，说明这次任务没有获得所需资源。另一方面，如果down\_interruptible(
)正常结束并得到了需要的资源，就返回0。

同自旋锁一样，信号量在内核中也有许多变种，比如读者－写者信号量等，这里不做一一介绍。下表是信号量的操作函数列表：

函数                       |     描述                          
------                     |    ---                           
                           |                    
down(struct semaphore \*); | 获取信号量，如果不可获取，
                           | 则进入不可中断睡眠状态（目前已经不建议使用）。
                           |
down\_interruptible        | 获取信号量，如果不可获取，               
(struct semaphore \*);     | 则进入可中断睡眠状态。
                           |
down\_killable             | 获取信号量，如果不可获取，
(struct semaphore \*);     | 则进入可被致命信号中断的睡眠状态。 
                           |
down\_trylock              |  尝试获取信号量，
(struct semaphore \*);     |  如果不能获取，则立刻返回。
                           |
down\_timeout              | 在给定时间（jiffies）
(struct semaphore \*,      | 内获取信号量，如果不能够获取，
long jiffies);             | 则返回。
                           | 
up(struct semaphore \*);   | 释放信号量。 

3． 信号量和自旋锁区别

了解何时使用自旋锁，何时使用信号量对编写优良代码很重要，但是多数情况下，不需要太多的考虑，因为在中断上下文中只能使用自旋锁，而在任务睡眠时只能使用信号量。表7.2给出了自旋锁与信号量的对比

表7.2 自旋锁与信号量对比

需求                     | 建议的加锁方法
------                   | ---
低开销加锁               | 优先使用自旋锁
                         |
短期锁定                 | 优先使用自旋锁
                         |
长期加锁                 | 优先使用信号量
                         | 
中断上下文中加锁         | 使用自旋锁
                         |
持有锁时需要睡眠、调度   | 使用信号量
