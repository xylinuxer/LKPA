##5.2中断描述符表的初始化

通过上面的介绍，我们知道了Intel微处理器对中断和异常所做的工作。下面，我们从操作系统的角度来对中断描述符表的初始化给予描述。

Linux内核在系统的初始化阶段要进行大量的初始化工作，其与中断相关的工作有：初始化可编程控制器8259A；将中断描述符表的起始地址装入IDTR寄存器，并初始化表中的每一项。这些操作的完成将在本节进行具体描述。

用户进程可以通过INT指令发出一个中断请求，其中断请求向量在0～255之间。为了防止用户使用INT指令模拟非法的中断和异常，必须对中断描述符表进行谨慎的初始化。其措施之一就是将中断门或陷阱门中的请求特权级DPL域置为0。如果用户进程确实发出了这样一个中断请求，CPU会检查出其**当前特权级**CPL（3）与所请求的特权级DPL（0）有冲突，因此产生一个“通用保护”异常。

但是，有时候必须让用户进程能够使用内核所提供的功能（比如系统调用），也就是说从用户态进入内核态，这可以通过把中断门或陷阱门的DPL域置为3
来达到。

当计算机运行在实模式时，中断描述符表被初始化，并由BIOS使用。然而，一旦真正进入了Linux内核，中断描述符表就被移到内存的另一个区域，并为进入保护模式进行预初始化：用汇编指令LIDT对中断向量表寄存器IDTR进行初始化，即把IDTR置为0。把中断描述符表IDT的起始地址装入IDTR

用setup_idt()函数填充中断描述表中的256个表项。在对这个表进行填充时，使用了一个空的中断处理程序。因为现在处于初始化阶段，还没有任何中断处理程序，因此，用这个空的中断处理程序填充每个表项。

在对中断描述符表进行预初始化后,
内核将在启用分页功能后对IDT进行第二遍初始化,也就是说，用实际的陷阱和中断处理程序替换这个空的处理程序。一旦这个过程完成，对于每个异常，IDT都由一个专门的陷阱门或系统门，而对每个外部中断，IDT都包含专门的中断门。

### 5.2.1 IDT表项的设置

IDT表项的设置是通过_set_gaet()函数实现的，在此，我们给出如何调用该函数在IDT表中插入一个门：

####1. 插入一个中断门

```c
void set_intr_gate(unsigned int n, void *addr)

{

		_set_gate(idt_table+n,14,0,addr);

}
```

其中，idt_table是中断描述符表IDT在程序中的符号表示，n表示在第n个表项中插入一个中断门。这个门的段选择符设置成代码段的选择符，DPL域设置成0，14表示D标志位为1（表示32位）而类型码为110，所以set_intr_gate（）设置的是中断门，偏移域设置成中断处理程序的地址addr。

####2. 插入一个陷阱门

```c
static void __init set_trap_gate(unsigned int n, void *addr)

{

		_set_gate(idt_table+n,15,0,addr);

}
```
在第n个表项中插入一个陷阱门。这个门的段选择符设置成代码段的选择符，DPL域设置成0，15表示D标志位为1而类型码为111，所以set_trap_gate（）设置的是陷阱门，偏移域设置成异常处理程序的地址addr。

#### 插入一个系统门

```c
static void __init set_system_gate(unsigned int n, void *addr)

{

		_set_gate(idt_table+n,15,3,addr);

}
```
在第n个表项中插入一个系统门。这个门的段选择符设置成代码段的选择符，DPL域设置成3，15表示D标志位为1而类型码为111，所以set_system_gate（）设置的也是陷阱门，但因为DPL为3，因此，系统调用在用户态可以通过“INT0x80”顺利穿过系统门，从而进入内核态。

### 5.2.2对陷阱门和系统门的初始化

trap_init()函数就是设置中断描述符表开头的19个陷阱门和系统门，这些中断向量都是CPU保留用于异常处理的：

```c
set_trap_gate(0,&divide_error);

set_trap_gate(1,&debug);

……

set_trap_gate(19,&simd_coprocessor_error);

set_system_gate(SYSCALL_VECTOR,&system_call);
```

其中，“&”之后的名字就是每个异常处理程序的名字。最后一个是对系统调用的设置。

### 5.2.3 中断门的设置

中断门的设置是由init_IRQ()函数中的一段代码完成的：

```c
for (i = 0; i < (NR_VECTORS - FIRST_EXTERNAL_VECTOR); i++) {  
		int vector = FIRST_EXTERNAL_VECTOR + i;  
		if (i >= NR_IRQS)  
				break;  
		if (vector != SYSCALL_VECTOR)  
		set_intr_gate(vector, interrupt[i]);  
   }
```

从FIRST_EXTERNAL_VECTOR开始，设置NR_IRQS（NR_VECTORS - FIRST_EXTERNAL_VECTOR）个IDT表项。常数FIRST_EXTERNAL_VECTOR定义为0x20，而NR_IRQS则为224[^1]，即中断门的个数。注意，必须跳过用于系统调用的向量0x80，因为这在前面已经设置好了。
这里，中断处理程序的入口地址是一个数组interrupt[]，数组中的每个元素是指向中断处理函数的指针。

### 5.2.4 中断处理程序的形成

由前一节知道，interrupt[]为中断处理程序的入口地址，这只是一个笼统的说法。实际上不同的中断处理程序，不仅名字不同，其内容也不同，但是，这些函数又有很多相同之处，因此应当以统一的方式形成其函数名和函数体，于是，内核对该数组的定义如下：

```c
static void (*interrupt[NR_VECTORS - FIRST_EXTERNAL_VECTOR])(void) = {

IRQLIST_16(0x2), IRQLIST_16(0x3),

IRQLIST_16(0x4), IRQLIST_16(0x5), IRQLIST_16(0x6), IRQLIST_16(0x7),

IRQLIST_16(0x8), IRQLIST_16(0x9), IRQLIST_16(0xa), IRQLIST_16(0xb),

IRQLIST_16(0xc), IRQLIST_16(0xd), IRQLIST_16(0xe), IRQLIST_16(0xf)

 };
```

这里定义的数组interrupt[]，从IRQLIST_16(0x2)到IRQLIST_16(0xf)一共有14个数组元素，其中IRQLIST_16()宏的定义如下：

  #define IRQLIST_16(x) 

  IRQ(x,0), IRQ(x,1), IRQ(x,2), IRQ(x,3), 

  IRQ(x,4), IRQ(x,5), IRQ(x,6), IRQ(x,7), 

  IRQ(x,8), IRQ(x,9), IRQ(x,a), IRQ(x,b), 

  IRQ(x,c), IRQ(x,d), IRQ(x,e), IRQ(x,f)

该宏中定义了16个IRQ(x,y)，这样就有224（14*16）个函数指针。不妨再接着展开IRQ(x,y)宏：

#define IRQ(x,y) 

IRQ##x##y##_interrupt

## 表示将字符串连接起来，比如IRQ(0x2,0)就是IRQ0x20_interrupt。

综上可知，以这样的方式就定义出224个函数，从
IRQ0x20_interrupt一直到IRQ0xff_interupt。那么这些函数名又是如何形成的？我们看如下宏定义：

#define IRQ_NAME2(nr) nr##_interrupt(void)

#define IRQ_NAME(nr) IRQ_NAME2(IRQ##nr)

从这两个宏的定义可以推知，IRQ_NAME(n)就是IRQn_interrupt(void)函数形式，其中随n具体数字不同，则形成不同的IRQn_interrupt()函数名。接下来，又如何以统一的方式让这些函数拥有内容，也就是说，这些函数的代码是如何形成的？内核定义了BUILD_IRQ宏。

BUILD_IRQ宏是一段嵌入式汇编代码，为了有助于理解，我们把它展开成下面的汇编语言片段：

IRQn_interrupt:

            pushl $n-256

jmp common_interrupt

把中断号减256的结果保存在栈中，这是进入中断处理程序后第一个压入堆栈的值，是一个负数，正数留给系统调用使用。对于每个中断处理程序，唯一不同的就是压入栈中的这个数。然后，所有的中断处理程序都跳到一段相同的代码common_interrupt。关于这段代码，请参看5.3.3一节中断处理程序IRQn_interrupt。