## 5.2 中断描述符表的初始化

通过上面的介绍，我们知道了Intel微处理器对中断和异常所做的工作。下面，我们从操作系统的角度来对中断描述符表的初始化给予描述。

Linux内核在系统的初始化阶段要进行大量的初始化工作，其与中断相关的工作有：初始化可编程控制器8259A；将中断描述符表的起始地址装入IDTR寄存器，并初始化表中的每一项。这些操作的完成将在本节进行具体描述。

用户进程可以通过INT指令发出一个中断请求，其中断请求向量在0～255之间。为了防止用户使用INT指令模拟非法的中断和异常，必须对中断描述符表进行谨慎的初始化。其措施之一就是将中断门或陷阱门中的请求特权级DPL域置为0。如果用户进程确实发出了这样一个中断请求，CPU会检查出其**当前特权级**CPL（3）与所请求的特权级DPL（0）有冲突，因此产生一个“通用保护”异常。

但是，有时候必须让用户进程能够使用内核所提供的功能（比如系统调用），也就是说从用户态进入内核态，这可以通过把中断门或陷阱门的DPL域置为3  
来达到。

当计算机运行在实模式时，中断描述符表被初始化，并由BIOS使用。然而，一旦真正进入了Linux内核，中断描述符表就被移到内存的另一个区域，并为进入保护模式进行预初始化：用汇编指令LIDT对中断向量表寄存器IDTR进行初始化，即把IDTR置为0。把中断描述符表IDT的起始地址装入IDTR

用setup\_idt\(\)函数填充中断描述表中的256个表项。在对这个表进行填充时，使用了一个空的中断处理程序。因为现在处于初始化阶段，还没有任何中断处理程序，因此，用这个空的中断处理程序填充每个表项。

在对中断描述符表进行预初始化后,  
内核将在启用分页功能后对IDT进行第二遍初始化,也就是说，用实际的陷阱和中断处理程序替换这个空的处理程序。一旦这个过程完成，对于每个异常，IDT都由一个专门的陷阱门或系统门，而对每个外部中断，IDT都包含专门的中断门。

### 5.2.1 IDT表项的设置

IDT表项的设置是通过\_set\_gaet\(\)函数实现的，在此，我们给出如何调用该函数在IDT表中插入一个门：

#### 1. 插入一个中断门

```c
static inline void set_intr_gate(unsigned int n, void *addr)
{
    BUG_ON((unsigned)n > 0xFF);
    _set_gate(n, GATE_INTERRUPT, addr, 0, 0, __KERNEL_CS);
}
```

其中，n是中断号，addr是中断处理程序的入口地址，GATE\_INTERRUPT是中断门类型，这样我们能看出来这是一个中断门。

#### 2. 插入一个陷阱门

```c
static inline void set_trap_gate(unsigned int n, void *addr)
{
    BUG_ON((unsigned)n > 0xFF);
    _set_gate(n, GATE_TRAP, addr, 0, 0, __KERNEL_CS);
}
```

可以看到传入的参数是GATE\_TRAP，所以是一个陷阱门。其他参数如中断门所讲的一样。

#### 插入一个系统门

```c
static inline void set_system_trap_gate(unsigned int n, void *addr)
{
    BUG_ON((unsigned)n > 0xFF);
    _set_gate(n, GATE_TRAP, addr, 0x3, 0, __KERNEL_CS);
}
```

可以看到传入的参数是GATE\_TRAP，所以是系统门。其他的参数如同中断门讲述的一样。

### 5.2.2 对陷阱门、系统门和中断门的初始化

trap\_init\(\)函数就是设置中断描述符表开头的19个陷阱门，这些中断向量都是CPU保留用于异常处理的：

```c
    /**
     *X86_TRAP_DE = 0,    /*  0, Divide-by-zero */
     *X86_TRAP_DB,        /*  1, Debug */
     *X86_TRAP_NMI,        /*  2, Non-maskable Interrupt */
     *X86_TRAP_BP,        /*  3, Breakpoint */
     *X86_TRAP_OF,        /*  4, Overflow */
     *X86_TRAP_BR,        /*  5, Bound Range Exceeded */
     *X86_TRAP_UD,        /*  6, Invalid Opcode */
     *X86_TRAP_NM,        /*  7, Device Not Available */
     *X86_TRAP_DF,        /*  8, Double Fault */
     *X86_TRAP_OLD_MF,    /*  9, Coprocessor Segment Overrun */
     *X86_TRAP_TS,        /* 10, Invalid TSS */
     *X86_TRAP_NP,        /* 11, Segment Not Present */
     *X86_TRAP_SS,        /* 12, Stack Segment Fault */
     *X86_TRAP_GP,        /* 13, General Protection Fault */
     *X86_TRAP_PF,        /* 14, Page Fault */
     *X86_TRAP_SPURIOUS,    /* 15, Spurious Interrupt */
     *X86_TRAP_MF,        /* 16, x87 Floating-Point Exception */
     *X86_TRAP_AC,        /* 17, Alignment Check */
     *X86_TRAP_MC,        /* 18, Machine Check */
     *X86_TRAP_XF,        /* 19, SIMD Floating-Point Exception */
     **/
    set_intr_gate(X86_TRAP_DE, &divide_error);
    set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
    /* int4 can be called from all */
    set_system_intr_gate(X86_TRAP_OF, &overflow);
    set_intr_gate(X86_TRAP_BR, &bounds);
    set_intr_gate(X86_TRAP_UD, &invalid_op);
    set_intr_gate(X86_TRAP_NM, &device_not_available);
#ifdef CONFIG_X86_32
    set_task_gate(X86_TRAP_DF, GDT_ENTRY_DOUBLEFAULT_TSS);
#else
    set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
#endif
    set_intr_gate(X86_TRAP_OLD_MF, &coprocessor_segment_overrun);
    set_intr_gate(X86_TRAP_TS, &invalid_TSS);
    set_intr_gate(X86_TRAP_NP, &segment_not_present);
    set_intr_gate(X86_TRAP_SS, stack_segment);
    set_intr_gate(X86_TRAP_GP, &general_protection);
    set_intr_gate(X86_TRAP_SPURIOUS, &spurious_interrupt_bug);
    set_intr_gate(X86_TRAP_MF, &coprocessor_error);
    set_intr_gate(X86_TRAP_AC, &alignment_check);
#ifdef CONFIG_X86_MCE
    set_intr_gate_ist(X86_TRAP_MC, &machine_check, MCE_STACK);
#endif
    set_intr_gate(X86_TRAP_XF, &simd_coprocessor_error);
    for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
        set_bit(i, used_vectors);

#ifdef CONFIG_IA32_EMULATION
    set_system_intr_gate(IA32_SYSCALL_VECTOR, ia32_syscall);
    set_bit(IA32_SYSCALL_VECTOR, used_vectors);
#endif

#ifdef CONFIG_X86_32
    set_system_trap_gate(SYSCALL_VECTOR, &system_call);
    set_bit(SYSCALL_VECTOR, used_vectors);
#endif

}
```

其中，“&”之后的名字就是每个异常处理程序的名字。最后一个是对系统调用的设置。

set\_system\_intr\_gate\(IA32\_SYSCALL\_VECTOR, ia32\_syscall\);这个是对系统中断门进行初始化。

set\_system\_trap\_gate\(SYSCALL\_VECTOR, &system\_call\);这个是对系统门进行初始化。

### 5.2.3 初始化中断

内核是在异常和陷阱初始化完成的情况下才会进行中断的初始化，中断的初始化也是处于start\_kernel\(\)函数中，分为两个部分，分别是early\_irq\_init\(\)和init\_IRQ\(\)。early\_irq\_init\(\)是第一步的初始化，其工作主要是跟硬件无关的一些初始化，比如一些变量的初始化，分配必要的内存等。init\_IRQ\(\)是第二步，其主要就是关于硬件部分的初始化了。

那让我们来看一下最主要的init\_IRQ\(\)实现了什么。

```c
void __init init_IRQ(void)
{
    int i;
    x86_add_irq_domains();
    for (i = 0; i < legacy_pic->nr_legacy_irqs; i++)
        per_cpu(vector_irq, 0)[IRQ0_VECTOR + i] = i;

    x86_init.irqs.intr_init();
}
```

x86\_init.irqs.intr\_init\(\)是一个函数指针，其指向native\_init\_IRQ\(\);我们可以直接看看native\_init\_IRQ\(\);

```c
void __init native_init_IRQ(void)
{
    int i;

    x86_init.irqs.pre_vector_init();

    apic_intr_init();

    i = FIRST_EXTERNAL_VECTOR;
    for_each_clear_bit_from(i, used_vectors, NR_VECTORS) {
        set_intr_gate(i, interrupt[i - FIRST_EXTERNAL_VECTOR]);
    }

    if (!acpi_ioapic && !of_ioapic)
        setup_irq(2, &irq2);

#ifdef CONFIG_X86_32
    irq_ctx_init(smp_processor_id());
#endif
}
```

interrupt\[\]就是中断程序的入口地址，外部中断的门描述的中断处理函数都为interrupt\[i\]。

### 5.2.4 中断处理程序的形成

由前一节知道，interrupt\[\]为中断处理程序的入口地址，这只是一个笼统的说法。实际上不同的中断处理程序，不仅名字不同，其内容也不同，但是，这些函数又有很多相同之处，因此应当以统一的方式形成其函数名和函数体，于是，内核对该数组的定义如下：

```c
static void (*interrupt[NR_IRQS])(void) = {
    NULL, NULL, IRQ2_interrupt, IRQ3_interrupt,
    IRQ4_interrupt, IRQ5_interrupt, IRQ6_interrupt, IRQ7_interrupt,
    IRQ8_interrupt, IRQ9_interrupt, IRQ10_interrupt, IRQ11_interrupt,
    IRQ12_interrupt, IRQ13_interrupt, NULL, NULL,    
    IRQ16_interrupt, IRQ17_interrupt, IRQ18_interrupt, IRQ19_interrupt,    
    IRQ20_interrupt, IRQ21_interrupt, IRQ22_interrupt, IRQ23_interrupt,    
    IRQ24_interrupt, IRQ25_interrupt, NULL, NULL, NULL, NULL, NULL,
    IRQ31_interrupt
};
```

nterrupt\[i\]的每个元素都相同，执行相同的汇编代码，这段汇编代码实际上很简单，它主要工作就是将**中断向量号**和**被中断上下文**

\(进程上下文或者中断上下文\)保存到栈中，最后调用do\_IRQ函数。 从IRQ2\_interrupt一直到IRQ31\_interupt。那么这些函数名又是如何形成的？我们看如下宏定义：

```
#define IRQ_NAME2(nr) nr##_interrupt(void)
#define IRQ_NAME(nr) IRQ_NAME2(IRQ##nr)
```

从这两个宏的定义可以推知，IRQ\_NAME\(n\)就是IRQn\_interrupt\(void\)函数形式，其中随n具体数字不同，则形成不同的IRQn\_interrupt\(\)函数名。接下来，又如何以统一的方式让这些函数拥有内容，也就是说，这些函数的代码是如何形成的？内核定义了BUILD\_IRQ宏。

BUILD\_IRQ宏是一段嵌入式汇编代码，为了有助于理解，我们把它展开成下面的汇编语言片段：

```c
IRQn_interrupt:
                pushl $n-256
                jmp common_interrupt
```

把中断号减256的结果保存在栈中，这是进入中断处理程序后第一个压入堆栈的值，是一个负数，正数留给系统调用使用。对于每个中断处理程序，唯一不同的就是压入栈中的这个数。然后，所有的中断处理程序都跳到一段相同的代码common\_interrupt。关于这段代码，请参看5.3.3一节中断处理程序IRQn\_interrupt。





