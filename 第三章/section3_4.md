## **3.4 进程调度**

&emsp;&emsp;在多进程的操作系统中，进程调度是一个全局性、关键性的问题，它对系统的总体设计、系统的的实现、功能设置以及各个方面的性能都有着决定性的影响。进程调度算法的设计，还对系统的复杂性有着极大的影响，常常会由于实现的复杂程度而在功能与性能方面作出必要的权衡和让步。在Linux 2.6中为了提高性能，对调度算法进行了大幅度改进，其实现复杂度也随之增加。为了简单起见，这里以Linux 2.4中的调度算法来说明进程调度原理。后面附加说明Linux 2.6中进程调度的改进方法。

### **3.4.1基本原理**

&emsp;&emsp;从前面我们可以看到，进程运行时需要各种各样的系统资源，如内存、文件、打印机和最宝贵的CPU等等，所以说，调度的实质就是资源的分配。系统通过不同的调度算法来实现这种资源的分配。通常来说，选择什么样的调度算法取决于的资源分配的策略， 一个好的调度算法应当考虑以下几个方面：

&emsp;&emsp;（1）公平：保证每个进程得到合理的CPU时间。

&emsp;&emsp;（2）高效：使CPU保持忙碌状态，即总是有进程在CPU上运行。

&emsp;&emsp;（3）响应时间：使交互用户的响应时间尽可能短。

&emsp;&emsp;（4）周转时间：使批处理用户等待输出的时间尽可能短。

&emsp;&emsp;（5）吞吐量：使单位时间内处理的进程数量尽可能多。

&emsp;&emsp;很显然，这5个目标不可能同时达到，所以，不同的操作系统会在这几个方面中作出相应的取舍，从而确定自己的调度算法，例如Unix采用动态优先数调度、BSD采用多级反馈队列调度、Windows采用抢先多任务调度等等。

下面来了解一下主要的调度算法及其基本原理：

**1．时间片轮转调度算法**

&emsp;&emsp;时间片（Time Slice）就是分配给进程运行的一段时间。

&emsp;&emsp;在分时系统中，为了保证人机交互的及时性，系统使每个进程依次地按时间片轮流地执行，此时应采用时间片轮转法进行调度。在通常的轮转法中，系统将所有的可运行（即就绪）进程按先来先服务的原则，排成一个队列，每次调度时把CPU分配给队首进程，并令其执行一个时间片。时间片的大小从几ms到几百ms不等。当执行的时间片用完时，系统发出信号，通知调度程序，调度程序便据此信号来停止该进程的执行，并将它送到运行队列的末尾，等待下一次执行；然后，把处理机分配给就绪队列中新的队首进程，同时也让它执行一个时间片。这样就可以保证就绪队列中的所有进程，在一个给定的时间（人所能接受的等待时间）内，均能获得一时间片的处理机执行时间。

**2．优先权调度算法**

&emsp;&emsp;为了照顾到紧迫型进程在进入系统后便能获得优先处理，引入了最高优先权调度算法。当将该算法用于进程调度时，系统将把处理机分配给运行队列中优先权最高的进程，这时，又可进一步把该算法分成两种方式：

&emsp;&emsp;(1) 非抢占式优先权算法（又称不可剥夺调度：Nonpreemptive Scheduling）

&emsp;&emsp;在这种方式下，系统一旦将处理机（CPU）分配给运行队列中优先权最高的进程后，该进程便一直执行下去，直至完成；或因发生某事件使该进程放弃处理机时，系统方可将处理机分配给另一个优先权高的进程。这种调度算法主要用于批处理系统中，也可用于某些对实时性要求不严的实时系统中。

&emsp;&emsp;(2) 抢占式优先权调度算法（又称可剥夺调度：Preemptive Scheduling）

&emsp;&emsp;该算法的本质就是系统中当前运行的进程永远是可运行进程中优先权最高的那个。

&emsp;&emsp;在这种方式下，系统同样是把处理机分配给优先权最高的进程，使之执行。但是只要一出现了另一个优先权更高的进程时，调度程序就暂停原最高优先权进程的执行，而将处理机分配给新出现的优先权最高的进程，即剥夺当前进程的运行。因此，在采用这种调度算法时，每当出现一新的可运行进程，就将它和当前运行进程进行优先权比较，如果高于当前进程，将触发进程调度。

&emsp;&emsp;这种方式的优先权调度算法，能更好的满足紧迫进程的要求，故而常用于实时性要求比较严格的系统中，以及对性能要求较高的批处理和分时系统中。Linux目前也采用这种调度算法。

**3．多级反馈队列调度**

&emsp;&emsp;这是一种折衷调度算法。其本质是综合了时间片轮转调度和抢占式优先权调度的优点，即优先权高的进程先运行给定的时间片，相同优先权的进程轮流运行给定的时间片。

**4．实时调度**

&emsp;&emsp;最后我们来看一下实时系统中的调度。什么叫实时系统，就是系统对外部事件有求必应、尽快响应。在实时系统中存在有若干个实时进程或任务，它们用来反应或控制某个（些）外部事件，往往带有某种程度的紧迫性，因而一般采用抢占式调度方式。

### **3.4.2时间片**

&emsp;&emsp;时间片表明进程在被抢占前所能持续运行的时间。调度策略必须规定一个默认的时间片，但这并不是件简单的事。时间片过长会导致系统对交互的响应表现欠佳；让人觉得系统无法并发执行应用程序。时间片太短会明显增大进程切换带来的处理器时间，因为肯定会有相当的一部分系统时间用在进程切换上，而用来运行的时间片却很短。从上面的争论中可以看出，任何长时间片都将导致系统交互表现欠佳。很多操作系统中都特别重视这点，所以默认的时间片很短——如20毫秒。

&emsp;&emsp;Linux调度程序提高交互式程序的优先级，让它们运行得更频繁。于是，调度程序提供较长的默认时间片给交互式程序。此外，Linux调度程序还根据进程的优先级动态调整分配给它的时间片。从而保证了优先级高的进程，也应该是重要性高的进程，执行的频率高，执行时间长。通过实现这样一种动态调整优先级和时间片长度的机制，Linux调度性能不但非常稳定而且也很强健。

### **3.4.3 Linux进程调度时机**

&emsp;&emsp;Linux的调度程序是一个叫Schedule（）的函数，这个函数被调用的频率很高，由它来决定是否要进行进程的切换，如果要切换的话，切换到哪个进程等等。我们先来看在什么情况下要执行调度程序，Linux调度时机主要有：

&emsp;&emsp;(1)	进程状态转换的时刻：进程终止、进程睡眠；

&emsp;&emsp;(2)	当前进程的时间片用完时；

&emsp;&emsp;(3)	设备驱动程序运行时；

&emsp;&emsp;(4)	从内核态返回到用户态时；

&emsp;&emsp;时机(1)，进程要调用sleep_on（）或exit（）等函数时，这些函数会主动调用调度程序。

&emsp;&emsp;时机(2)，由于进程的时间片用完时要放弃CPU，因此也是主动调用调度程序。

&emsp;&emsp;时机(3)，当设备驱动程序执行长而重复的任务时，直接调用调度程序。在每次反复循环中，驱动程序都检查调度标志，如果必要，则调用调度程序schedule()主动放弃CPU。

&emsp;&emsp;时机(4)，不管是从中断、异常还是系统调用返回，都要对调度标志进行检测，如果必要，则调用调用调度程序。那么，为什么从系统调用返回时要调用调度程序呢？这当然是从效率考虑。从系统调用返回意味着要离开内核态而返回到用户态，而状态的转换要花费一定的时间，因此，在返回到用户态前，系统把在内核态该处理的事全部做完。
  
### **3.4.4 进程调度的依据**

&emsp;&emsp;调度程序运行时，要在所有处于可运行状态的进程之中选择最值得运行的进程投入运行。选择进程的依据是什么呢？在进程的task_struct结构中有以下几个与调度相关的域：

&emsp;&emsp;（1）need_resched：调度标志，以决定是否调用schedule( )函数。

&emsp;&emsp;（2）counter: 进程处于可运行状态时所剩余的时钟节拍数，每次时钟中断到来时，这个值就减1。这个值将作为进程调度的依据，因此，也把这个域叫做进程的“动态优先级”，这就巧妙地把时间片和优先级结合起来。

&emsp;&emsp;(3）nice: 进程的基本优先级，或叫做“静态优先级”。它的值决定counter的初值。这个域包含的值在－20～19之间；负值对应“高优先级”进程，正数对应“低优先级”进程。缺省值0对应普通进程。这个值也可以由用户通过nice系统调用进行改变。
 
&emsp;&emsp;（4）policy: 调度的类型，允许的取值是：
  
&emsp;&emsp; SCHED_FIFO

&emsp;&emsp;先入先出的实时进程。

&emsp;&emsp; SCHED_RR

&emsp;&emsp;时间片轮转的实时进程。当调度程序把CPU分配给一个进程时，把这个进程的PCB就放在运行队列的末尾。这种策略确保了把CPU时间公平地分配给具有相同优先级的所有SCHED_RR实时进程。


&emsp;&emsp; SCHED_OTHER

&emsp;&emsp;普通的分时进程。
		
&emsp;&emsp;（5）rt_priority: 实时进程的优先级
     
&emsp;&emsp;这里要说明的是，与其他分时操作系统一样，Linux的时间单位是“时钟节拍”，Linux设计者将一个时钟节拍定义为10ms（在内核2.6版以后最小可可以定义为1ms）。在这里，我们把counter叫做进程的时间片，系统用时钟节拍数来表示，例如，若counter为2，则分配给该进程的时间片就为2个时钟节拍，也就是2*10ms=20ms。

&emsp;&emsp;以下代码片段取自Linux 2.4。

&emsp;&emsp;Linux中有一个goodness（）函数用来衡量一个处于可运行状态的进程值得运行的程度。该函数综合使用了上面提到的几域，给每个处于可运行状态的进程赋予一个权值（weight），调度程序以这个权值作为选择进程的唯一依据。函数主体如下（为了便于理解，笔者对函数做了一些改写和简化，只考虑单处理机的情况）：
  
    static inline int goodness(struct task_struct * p, struct mm_struct *this_mm)
    {	int weight;     ／* 权值，作为衡量进程是否运行的唯一依据 *

       weight=-1;   
       if (p->policy&SCHED_YIELD)  
        goto out;  /*如果该进程愿意“礼让（yield）”，则让其权值为－1 *／
	 switch(p->policy)  
	{
		/* 实时进程*/
		case SCHED_FIFO:
		case SCHED_RR:
			weight = 1000 + p->rt_priority;

		/* 普通进程 */
		case SCHED_OTHER:
		    {	weight = p->counter;
		      if(!weight)
			   goto out
			/* 做细微的调整*/
			if (p->mm=this_mm||!p->mm)
					weight = weight+1;
		       weight+=20-p->nice;				
			}	
       }
    out:
    return weight;   /*返回权值*/
    }

&emsp;&emsp;其中，在sched.h中对调度策略定义如下：

    #define SCHED_OTHER             0
    #define SCHED_FIFO              1
    #define SCHED_RR                2
    #define SCHED_YIELD             0x10

&emsp;&emsp;这个函数比较很简单。首先，根据policy区分实时进程和普通进程。实时进程的权值取决于其实时优先级，其至少是1000，与conter和nice无关。普通进程的权值需特别说明两点：

&emsp;&emsp;（1）	为什么进行细微的调整？如果p->mm为空，则意味着该进程无用户空间（例如内核线程），则无需切换到用户空间。如果p->mm=this_mm，则说明该进程的用户空间就是当前进程的用户空间，该进程完全有可能再次得到运行。对于以上两种情况，都给其权值加1，算是对它们小小的奖励。

&emsp;&emsp;（2）	进程的优先级nice是从早期Unix沿用下来的负向优先级，其数值标志“谦让”的程度，其值越大，就表示其越“谦让”，也就是优先级越低，其取值范围为－20～＋19，因此，（20-p->nice）的取值范围就是0～40。可以看出，普通进程的权值不仅考虑了其剩余的时间片，还考虑了其优先级，优先级越高，其权值越大。

&emsp;&emsp;有了衡量进程是否应该运行的标准，选择进程就是轻而易举的事情了，弱肉强食，谁的权值大谁就先运行。

&emsp;&emsp;根据进程调度的依据，调度程序就可以控制系统中的所有处于可运行状态的进程并在它们之间进行选择。

### **3.4.5 调度函数schedule( )的实现**

&emsp;&emsp;调度程序在内核中就是一个函数，为了讨论方便，我们同样对其进行了简化，略其对SMP的实现部分。
    
    asmlinkage void schedule(void)
    {
      struct task_struct *prev, *next, *p; ／* prev表示调度之前的进程, 
     next表示调度之后的进程 *／  
    struct list_head *tmp;    /* 定义一个临时指针，指向双向链表*/
    int this_cpu, c;           

      if (!current->active_mm) BUG();/*如果当前进程的的active_mm为空，出错*／
    need_resched_back:             
           prev = current;         ／*让prev成为当前进程 *／
           this_cpu = prev->processor;

    if (in_interrupt()) {／*如果schedule是在中断服务程序内部执行，
    就说明发生了错误*／
           printk("Scheduling in interrupt\n");
              BUG();
        }
       release_kernel_lock(prev, this_cpu); /*释放全局内核锁，
    并开this_cpu的中断*／
       spin_lock_irq(&runqueue_lock); ／*锁住运行队列，并且同时关中断*/
        if (prev->policy == SCHED_RR) ／*将一个时间片用完的SCHED_RR实时
               goto move_rr_last;      进程放到队列的末尾 *／
     move_rr_back:
        switch (prev->state) {     ／*根据prev的状态做相应的处理*／
               case TASK_INTERRUPTIBLE:  /*此状态表明该进程可以被信号中断*/
                        if (signal_pending(prev)) { /*如果该进程有未处理的
    信号，则让其变为可运行状态*/
                               prev->state = TASK_RUNNING;
                                break;
                        }
                 default:     ／*如果为可中断的等待状态或僵死状态*／
                        del_from_runqueue(prev); ／*从运行队列中删除*／
                case TASK_RUNNING:;／*如果为可运行状态，继续处理*／
         }
         prev->need_resched = 0;
 
     ／*下面是调度程序的正文 *／
    repeat_schedule:    ／*真正开始选择值得运行的进程*／
        next = idle_task(this_cpu); /*缺省选择空闲进程*/
       c = -1000;
     if (prev->state == TASK_RUNNING)
          goto still_running;
    still_running_back:
    list_for_each(tmp, &runqueue_head) { ／*遍历运行队列*／
       p = list_entry(tmp, struct task_struct, run_list);
     if (can_schedule(p, this_cpu)) { ／*单CPU中，该函数总返回1*／       
             int weight = goodness(p, this_cpu, prev->active_mm);
               if (weight > c)
                   c = weight, next = p;
           }
     }          
               
    ／* 如果c为0，说明运行队列中所有进程的权值都为0，也就是分配给各个进程的时间片都已用完，需重新计算各个进程的时间片 *／ 
 
    if  (!c) {
             struct task_struct *p;
             spin_unlock_irq(&runqueue_lock);／*锁住运行队列*／
              read_lock(&tasklist_lock);  ／* 锁住进程的双向链表*／
             for_each_task(p)            ／* 对系统中的每个进程*／
             p->counter = (p->counter >> 1) + NICE_TO_TICKS(p->nice);
             read_unlock(&tasklist_lock);
                 spin_lock_irq(&runqueue_lock);
               goto repeat_schedule;
        }

     spin_unlock_irq(&runqueue_lock);／*对运行队列解锁，并开中断*／

       if (prev == next) {     /*如果选中的进程就是原来的进程*/
            prev->policy &= ~SCHED_YIELD;
               goto same_process;
      }

          ／* 下面开始进行进程切换*／
       kstat.context_swtch++; ／*统计上下文切换的次数*／
   
        {
               struct mm_struct *mm = next->mm;
               struct mm_struct *oldmm = prev->active_mm;
              if (!mm) {  ／*如果是内核线程，则借用prev的地址空间*／
                      if (next->active_mm) BUG();
                      next->active_mm = oldmm;
                 
              } else { ／*如果是一般进程，则切换到next的用户空间*／
                       if (next->active_mm != mm) BUG();
                       switch_mm(oldmm, mm, next, this_cpu);
              }

            if (!prev->mm) { ／*如果切换出去的是内核线程*／
                  prev->active_mm = NULL;／*归还它所借用的地址空间*／
                    mmdrop(oldmm);     ／*mm_struct中的共享计数减1*／
               }
        }
    
        switch_to(prev, next, prev); ／*进程的真正切换，即堆栈的切换*／
        __schedule_tail(prev);  ／*置prev->policy的SCHED_YIELD为0 *／

    same_process:
       reacquire_kernel_lock(current);／*针对SMP*／
        if (current->need_resched)    ／*如果调度标志被置位*／
               goto need_resched_back; ／*重新开始调度*／
        return;
    }


&emsp;&emsp;以上就是调度程序的主要内容，为了对该程序形成一个清晰的思路，我们对其再给出进一步的解释：

&emsp;&emsp;(1) 如果当前进程既没有自己的地址空间，也没有向别的进程借用地址空间，那肯定出错。另外， 如果schedule()在中断服务程序内部执行,那也出错.

&emsp;&emsp;(2)	对当前进程做相关处理，为选择下一个进程做好准备。当前进程就是正在运行着的进程,可是，当进入schedule()时,其状态却不一定是TASK_RUNNIG，例如，在exit()系统调用中，当前进程的状态可能已被改为TASK_ZOMBE；又例如，在wait4()系统调用中，当前进程的状态可能被置为TASK_INTERRUPTIBLE。因此，如果当前进程处于这些状态中的一种，就要把它从运行队列中删除。

&emsp;&emsp;(3)	从运行队列中选择最值得运行的进程，也就是权值最大的进程。

&emsp;&emsp;(4)	如果已经选择的进程其权值为0，说明运行队列中所有进程的时间片都用完了（队列中肯定没有实时进程，因为其最小权值为1000），因此，重新计算所有进程的时间片，其中宏操作NICE_TO_TICKS就是把优先级nice转换为时钟节拍。

&emsp;&emsp;(5)	进程地址空间的切换。如果新进程有自己的用户空间，也就是说，如果next->mm与next->active_mm相同，那么，switch_mm( )函数就把该进程从内核空间切换到用户空间，也就是加载next的页目录。如果新进程无用户空间（next->mm为空），也就是说，如果它是一个内核线程，那它就要在内核空间运行，因此，需要借用前一个进程（prev）的地址空间，因为所有进程的内核空间都是共享的，因此，这种借用是有效的。

&emsp;&emsp;(6)	宏switch_to()进行真正的进程切换。

&emsp;&emsp;注意，从schedule(  )退出的return语句并不是由next进程立即执行，而是稍后一点在调度程序又选择prev执行时由prev进程执行。

&emsp;&emsp;switch_to（）的实现比较复杂，与具体的硬件体系结构有关，感兴趣的读者可以阅读相关的参考书。

### **3.4.6 Linux2.6调度程序的改进**

&emsp;&emsp;Linux2.4之前的版本，用较为简单的调度算法实现了进程调度。但是，随着Linux服务器上多处理器（SMP）的采用以及进程数量的增加，以前的调度算法存在以下问题：

&emsp;&emsp;（1）单就绪队列问题。不管进程的时间片是否耗完，都放在一个就绪队列中，这就使得时间片耗完的进程在不可能被调度的情况下，还毫无必要的参与调度，这是其一。其二，调度算法与系统进程数量密切相关，队列越长，选中一个进程的时间亦愈长，不适合用在硬实时系统。

&emsp;&emsp;（2）多处理器问题。多个处理器上的进程放在一个就绪队列中，使得这个就绪队列成为临界资源，各个处理器因为等待进入就绪队列而降低了系统效率。

&emsp;&emsp;（3）内核态不可抢占问题。只要一个进程进入了内核态，即使有另一个非常紧迫的任务到来，它也只能干等着，只有那个进程从内核态返回到用户态时，紧迫的任务才能占有处理机，这使得紧迫任务无法及时完成。
&emsp;&emsp;从以上分析可以看出，单就绪队列是影响调度性能的主要问题之一，因此改进就绪队列就成为改进调度算法的入口点。

**1. 就绪队列**

&emsp;&emsp;针对多处理器问题，每个CPU设置一个就绪队列。针对单就绪队列问题，设置两个队列组：活跃（active）队列组和时间片到期（expired）队列组。每个队列组中的元素以优先级再进行分类，相同优先级的进程为一个队列，最多可以有140个优先级，也就是对应140个队列，如图3.11
 
![](http://i.imgur.com/TJhVFpP.png)


&emsp;&emsp;如图3.11，没有耗完时间片的进程位于active队列组，耗完的进程存放在expired队列组，该组进程不再参与本轮调度，从而节省处理器时间。当一轮调度结束，则active队列组变为空，所有进程时间片耗完从而进入expired队列组。这时，active和expired两个指针互换，从而进入下一轮调度。

&emsp;&emsp;为了描述上述队列结构，同时考虑到SMP，Linux2.6为每个CPU定义一个struct runqueue数据结构：

	struct rt_rq {
	        struct rt_prio_array active;
	        unsigned int rt_nr_running;
	#if defined CONFIG_SMP || defined CONFIG_RT_GROUP_SCHED
	        struct {
	                int curr; /* highest queued rt task prio */
	#ifdef CONFIG_SMP
	                int next; /* next highest */
	#endif
	        } highest_prio;
	#endif
	#ifdef CONFIG_SMP
	        unsigned long rt_nr_migratory;
	        unsigned long rt_nr_total;
	        int overloaded;
	        struct plist_head pushable_tasks;
	#endif
	        int rt_throttled;
	        u64 rt_time;
	        u64 rt_runtime;
	        /* Nests inside the rq lock: */
	        raw_spinlock_t rt_runtime_lock;
	
	#ifdef CONFIG_RT_GROUP_SCHED
	        unsigned long rt_nr_boosted;
	
	        struct rq *rq;
	        struct list_head leaf_rt_rq_list;
	        struct task_group *tg;
	#endif
	};
 
其中，prio_array 定义为：

	struct rt_prio_array {
	        DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1); /* include 1 bit for delimiter */
	        struct list_head queue[MAX_RT_PRIO];
	};

&emsp;&emsp;runqueue中的两个指针active, expired分别指向array数组的array[0]和ayyay[1]，而这两个元素又分别指向队列数组queue[]，进一步，queue数组中的每个元素存放的是就绪进程的链表头，其中每个链表中的就绪进程具有相同的优先级。

**2. 就绪队列位图**

&emsp;&emsp;从图3.11可以看出，一个CPU上就绪队列最多可达280个。如何从中快速选中要运行的进程成为关系系统性能的一个关键因素。为此，Linux2.6为这两个进程组设置了以优先级为序的就绪队列位图，该位图的每一位对应一个就绪队列，只要队列中有一个就绪进程，则对应的位被置为1，否则置为0。这样，调度程序无需遍历所有的就绪队列，而只需遍历位图就可选中要运行的进程。例如，当前所有进程中最高优先级为50(换句话说，系统中没有任何进程的优先级小于50)。则调度程序先查找位图，如果找到优先级为38的队列有就绪进程，则直接读取active[37]，得到优先级为38的进程队列指针。该队列头上的第一个进程就是被选中的进程。这种算法的复杂度为O(1)，从而使得调度程序的开销与系统当前的负载（进程数）无关

**3. 优先级的动态调整**
  
&emsp;&emsp;为了提高交互式进程的响应时间，O(1)调度程序不仅动态地提高该类进程的优先级，还采用以下方法：

&emsp;&emsp;每次时钟节拍中断中，进程的时间片减1。当时间片为0时，调度程序判断当前进程的类型，如果是交互式进程或者实时进程，则重置其时间片并重新插入active数组。如果不是交互式进程则从active数组中移到expired数组。这样实时进程和交互式进程就总能优先获得CPU。然而这些进程不能始终留在active数组中，否则进入expire数组的进程就会产生饥饿现象。当进程已经占用CPU时间超过一个固定值后，即使它是实时进程或者交互式进程也会被移到expire数组中。

&emsp;&emsp;当active数组中的所有进程都被移到expire数组中后，调度程序交换active数组和expire数组。当进程被移入expire数组时，调度程序会重置其时间片，因此新的active数组又恢复了初始情况，而expire数组为空，从而开始新的一轮调度。

**4. 调度程序的再改进**

&emsp;&emsp;为了解决优先级动态调整等问题，大量难以维护和阅读的复杂代码被加入Linux2.6.0的调度模块，虽然很多性能问题因此得到了解决，可是另外一个严重问题始终困扰着许多内核开发者，那就是代码的复杂度问题。

&emsp;&emsp;在2004年，Con Kolivas提出了一个改进调度程序设计的补丁-楼梯调度程序（staircase scheduler，简称SD）。为调度程序设计提供了一种新的思路。

&emsp;&emsp;楼梯算法(SD)在思路上和O(1)算法的不同在于，它抛弃了动态优先级的概念，而采用了一种完全公平的思路。前任算法的主要复杂性来自动态优先级的计算，调度程序根据平均睡眠时间和一些很难理解的经验公式来修正进程的优先级以及区分交互式进程。这样的代码很难阅读和维护。

&emsp;&emsp;楼梯算法思路简单，但是实验证明它对应交互式进程的响应比其前任更好，而且极大地简化了代码。

&emsp;&emsp;楼梯算法和O(1)算法一样，也同样为每一个优先级维护一个进程队列，并将这些队列组织在active数组中。当选取下一个被调度进程时，SD算法也同样从active数组中直接读取进程。

&emsp;&emsp;与O(1)算法不同在于，当进程用完了自己的时间片后，并不是被移到expire数组中，而是被加入active数组的低一优先级队列中，即将其降低一个级别。不过请注意这里只是将该任务插入低一级优先级任务队列中，任务本身的优先级并没有改变。当时间片再次用完，任务被再次放入更低一级优先级任务队列中。就象一部楼梯，任务每次用完了自己的时间片之后就下一级楼梯。

&emsp;&emsp;任务下到最低一级楼梯时，如果时间片再次用完，它会回到初始优先级的下一级任务队列中。比如某进程的优先级为1，当它到达最后一级台阶140后，再次用完时间片时将回到优先级为2的任务队列中，即第二级台阶。不过此时分配给该任务的时间片将变成原来的2倍。比如原来该任务的时间片为10ms，则现在变成了20ms。基本的原则是，当任务下到楼梯底部时，再次用完时间片就回到上次下楼梯的起点的下一级台阶。并给予该任务相同于其最初分配的时间片。

&emsp;&emsp;以上描述的是普通进程的调度算法，实时进程还是采用原来的调度策略，即FIFO或者Round Robin。

&emsp;&emsp;楼梯算法能避免进程饥饿现象，高优先级的进程会最终和低优先级的进程竞争，使得低优先级进程最终获得执行机会。

&emsp;&emsp;对于交互式应用，当进入睡眠状态时，与它同等优先级的其他进程将一步一步地走下楼梯，进入低优先级进程队列。当该交互式进程再次唤醒后，它还留在高处的楼梯台阶上，从而能更快地被调度程序选中，加速了响应时间。

&emsp;&emsp;楼梯算法的优点在于，从实现角度看，SD基本上还是沿用了O(1)的整体框架，只是删除了O(1)调度程序中动态修改优先级的复杂代码；还淘汰了expire数组，从而简化了代码。

