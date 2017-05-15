## 7.3 生产者-消费者并发实例

  随着人们生活水平的提高，每天早餐基本是牛奶，面包。而在牛奶生产的环节中，生产厂家必须和经销商保持良好的沟通才能使效益最大化，具体说就是生产一批就卖一批，并且只有卖完了，才能生产下一批，这样才能达到供需平衡，否则就有可能造成浪费（供过于求）或者物资短缺（供不应求）。假设现在有一个牛奶生产厂家，他有一个经销商，并且由于资金不足，只有一个仓库。牛奶生产厂家首先生产一批牛奶，并放在仓库里，然后通知经销商来批发。经销商卖完牛奶后，打电话再订购下一批牛奶。牛奶生产厂家接到订单后，才开始生产下一批牛奶。

### 7.3.1 问题分析

  上述问题中，牛奶生产厂家就相当于“生产者”，经销商为“消费者”，仓库则为“公共缓冲区”。问题属于单一生产者，单一消费者，单一公共缓冲区。这属于典型的进程同步问题。生产者和消费者为不同的线程，“公共缓冲区“则为临界区。在同一时刻，只能有一个线程访问临界区。

### 7.3.2 实现机制

  1.数据定义

```c
    #include <linux/init.h>
    #include <linux/module.h>
    #include <linux/semaphore.h>
    #include <linux/sched.h>
    #include <asm/atomic.h>
    #include <linux/delay.h>
    #include <linux/kthread.h>
    #define PRODUCT_NUMS 10

    static struct semaphore sem_producer;
    static struct semaphore sem_consumer;
    static char product[12];
    static atomic_t num;
    static int producer(void *product);
    static int consumer(void *product);
    static int id = 1;
    static int consume_num = 1;
```

  2.生产者线程

```c
    static int producer(void *p)
    {
            char *product=(char *)p;
            int i;
            atomic_inc(&num);
            printk("producer [%d] start... \n", current->pid);
            for (i = 0; i < PRODUCT_NUMS; i++) {
                    down(&sem_producer);
                    snprintf(product,12,"2015-05-%d",id++);
                    printk("producer [%d] produce %s\n",current->pid, product);
                    up(&sem_consumer);
            }
            printk("producer [%d] exit...\n", current->pid);
            return 0;
    }
```

  该函数代表牛奶生产厂家，负责生产十批牛奶，从代码中可以看出，它的执行受制于sem\_producer信号量，当该信号量无法获取时，它将进入睡眠状态，直到信号量可用，它才能继续执行，并且释放sem\_constumer信号量。

  3.消费者线程

```c
    static int consumer(void *p)
    {
            char *product=(char *)p;
            printk("consumer [%d] start...\n", current->pid);
            for (;;) {
                    msleep(100);
                    down_interruptible(&sem_consumer);
                    if(consume_num >= PRODUCT_NUMS * atomic_read(&num))
                            break;
                    printk("consumer [%d] consume %s\n",current->pid, product);
                    consume_num++;
                    memset(product,'\0',12);
                    up(&sem_producer);
            }
            printk("consumer [%d] exit...\n", current->pid);
            return 0;
    }
```

&emsp;&emsp;该函数代表牛奶经销商，负责批发并销售牛奶。只有生产厂家生产了牛奶，下发了批发单，经销商才能批发牛奶。批发之后进行零售。当其零售完后，再向牛奶生产厂家下订货单。

  4.模块插入和删除

```c
    static int procon_init(void)
    {
            printk(KERN_INFO"show producer and consumer\n");
            sema_init(&sem_producer,1);
            sema_init(&sem_consumer,0);
            atomic_set(&num, 0);
            kthread_run(producer,product,"conpro_1");
            kthread_run(consumer,product,"compro_1");
            return 0;
    }
    static void procon_exit(void)
    {
            printk(KERN_INFO"exit producer and consumer\n");
    }

    module_init(procon_init);
    module_exit(procon_exit);
    MODULE_LICENSE("GPL");
    MODULE_DESCRIPTION("producer and consumer Module");
    MODULE_ALIAS("a simplest module");
```

### 7.3.3 具体实现

  对该模块的实际操作步骤如下：

* make 编译模块

* insmod procon.ko 加载模块

* dmesg 观察结果

* rmmod procon 卸载模块

  从结果可以看出，生产者线程首先执行生产一批产品，然后等待消费者线程消费产品。只有消费者消费后，生产者才能再进行生产。生产者严格按照生产顺序生产，消费者也严格按照生产顺序消费。

