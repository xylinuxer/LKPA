## **2.5 Linux中的汇编语言**

&emsp;&emsp;在Linux源代码中，大部分是用C语言编写的，但也涉及到汇编语言，有些汇编语言出现在以.S为扩展名的汇编文件中，在这种文件中，整个程序全部由汇编语言组成。有些汇编命令出现在以.c为扩展名的C文件中，在这种文件中，既有C语言，也有汇编语言，我们把出现在C代码中的汇编语言叫做“嵌入式”汇编。

&emsp;&emsp;尽管C语言已经成为编写操作系统的主要语言，但是，在操作系统与硬件打交道的过程中，在需要频繁调用的函数中以及某些特殊的场合中，C语言显得力不从心，这时，繁琐但又高效的汇编语言必须粉墨登场。因此，在了解80X86内存寻址的基础上，有必要对相关的汇编语言知识有所了解。

&emsp;&emsp;读者可能有过在DOS操作系统下编写汇编程序的经历,也具备一定的汇编知识。但是，在Linux的源代码中，你可能看到了与Intel的汇编语言格式不一样的形式，这就是AT&T的i386汇编语言。

### **2.5.1 AT&T与Intel汇编语言的比较**

&emsp;&emsp;我们知道，Linux是Unix家族的一员，尽管Linux的历史不长，但与其相关的很多事情都发源于Unix。就Linux所使用的i386汇编语言而言，它也是起源于Unix。Unix最初是为PDP-11开发的，曾先后被移植到VAX及68000系列的处理器上，这些处理器上的汇编语言都采用的是AT&T的指令格式。当Unix被移植到i386时，自然也就采用了AT&T的汇编语言格式，而不是Intel的格式。尽管这两种汇编语言在语法上有一定的差异，但所基于的硬件知识是相同的，因此，如果你非常熟悉Intel的语法格式，那么你也可以很容易地把它“移植”到AT&T来。下面我们通过对照Intel与AT&T的语法格式，以便于你把过去的知识能很快地“移植”过来。

 **1．前缀**

&emsp;&emsp;在Intel的语法中，寄存器和和立即数都没有前缀。但是在AT&T中，寄存器前冠以“％”，而立即数前冠以“$”。在Intel的语法中，十六进制和二进制立即数后缀分别冠以“h”和“b”，而在AT&T中，十六进制立即数前冠以“0x”，表2.2给出几个相应的例子。

&emsp;&emsp;表2.1 Intel与AT&T前缀的区别

|Intel语法|	AT&T语法|
|-----|-----|
| Mov	eax,8|	 movl	$8,%eax|
|Mov	ebx,0ffffh| movl	$0xffff,%ebx|
 |int	80h	 |  int 	$0x80|

**2. 操作数的方向**

&emsp;&emsp;Intel与AT&T操作数的方向正好相反。在Intel语法中，第一个操作数是目的操作数，第二个操作数源操作数。而在AT&T中，第一个数是源操作数，第二个数是目的操作数。由此可以看出，AT&T 的语法符合人们通常的阅读习惯。

&emsp;&emsp;例如：在Intel中，`mov	eax,[ecx]`

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;在AT&T中，`movl	(%ecx),%eax`

**3．内存单元操作数**

&emsp;&emsp;从上面的例子可以看出，内存操作数也有所不同。在Intel的语法中，基寄存器用“［］”括起来，而在AT&T中，用“（）”括起来。
    
&emsp;&emsp;例如： 在Intel中，`mov	eax,[ebx+5]`

&emsp;&emsp;&emsp;&emsp;&emsp;在AT&T，`movl	5(%ebx),%eax`

**4．操作码的后缀**

&emsp;&emsp;在上面的例子中你可能已注意到，在AT&T的操作码后面有一个后缀，其含义就是指出操作码的大小。“l”表示长整数（32位），“w”表示字（16位），“b”表示字节（8位）。而在Intel的语法中，则要在内存单元操作数的前面加上byte ptr、 word ptr和dword ptr，“dword”对应“long”。表2.4给出几个相应的例子。

> 表2.2 操作码的后缀举例

|Intel语法|	AT&T语法|
|------|------|
 |Mov	al,bl|	 movb	%bl,%al|
 |Mov	ax,bx|	 movw	%bx,%ax|
| Mov	eax,ebx	 |movl	%ebx,%eax|
| Mov	eax, dword ptr [ebx]|	 movl	(%ebx),%eax|

### **2.5.2 AT&T汇编语言的相关知识**

&emsp;&emsp;在Linux源代码中，以.S为扩展名的文件是“纯”汇编语言的文件。这里，我们结合具体的例子再介绍一些AT&T汇编语言的相关知识。

**1．GNU汇编程序GAS（GNU ASsembly和连接程序**

&emsp;&emsp;当你编写了一个程序后，就需要对其进行汇编（assembly）和连接。在Linux下有两种方式，一种是使用汇编程序GAS和连接程序ld，一种是使用gcc。我们先来看一下GAS和ld:

&emsp;&emsp;GAS把汇编语言源文件（.o）转换为目标文件（.o），其基本语法如下：

    as filename.s -o filename.o

&emsp;&emsp;一旦创建了一个目标文件，就需要把它连接并执行，连接一个目标文件的基本语法为：

    ld filename.o -o filename

&emsp;&emsp;这里 filename.o是目标文件名，而filename 是输出(可执行) 文件。
 
&emsp;&emsp;GAS使用的是AT&T的语法而不是Intel的语法，这就再次说明了AT&T语法是Unix世界的标准，你必须熟悉它。

&emsp;&emsp;如果要使用GNC的C编译器gcc，就可以一步完成汇编和连接，例如：

    gcc -o example example.S2 .  **AT&T中的节（Section）**

&emsp;&emsp;在AT&T的语法中，一个节由.section关键词来标识，当你编写汇编语言程序时，至少需要有以下三种节：

&emsp;&emsp;.section .data： 这种节包含程序已初始化的数据，也就是说，包含具有初值的那些变量，例如：

    hello : .string "Hello world!\n"
    hello_len : .long 13

&emsp;&emsp;.section .bss：这个节包含程序还未初始化的数据，也就是说，包含没有初值的那些变量。当操作系统装入这个程序时将把这些变量都置为0，例如：

    name : .fill 30   # 用来请求用户输入名字
    name_len : .long  0   # 名字的长度 (尚未定义)
&emsp;&emsp;当这个程序被装入时，name 和 name_len都被置为0。如果你在.bss节不小心给一个变量赋了初值，这个值也会丢失，并且变量的值仍为0。

&emsp;&emsp;使用.bss比使用.data的优势在于，.bss节不占用磁盘的空间。在磁盘上，一个长整数就足以存放.bss节。当程序被装入到内存时，操作系统也只分配给这个节4个字节的内存大小。
  
&emsp;&emsp;注意：编译程序把.data和.bss在4字节上对齐（align），例如，.data总共有34字节，那么编译程序把它对其在36字节上，也就是说，实际给它36字节的空间。

&emsp;&emsp;.section .text ：这个节包含程序的代码，它是只读节，而.data 和.bss是读／写节。

&emsp;&emsp;当然，关于AT&T的汇编内容还很多，感兴趣的读者可以参看相关文档。

### **2.5.3 Gcc嵌入式汇编**

&emsp;&emsp;在Linux的源代码中，有很多C语言的函数中嵌入一段汇编语言程序段，这就是gcc提供的“asm”功能，例如内核代码中，读控制寄存器CR0的一个功能函数read_cr0()以及其中的调用函数native_read_cr0()：

    static inline unsigned long read_cr0(void)
    {
        return native_read_cr0();
    }

    static inline unsigned long native_read_cr0(void)
    {
        unsigned long val;
        asm volatile("mov %%cr0,%0\n\t" : "=r" (val), "=m" (__force_order));
        return val;
    }
 
&emsp;&emsp;这种形式看起来比较陌生，因为这不是标准C所定义的形式，而是gcc 对C语言的扩充。其中__dummy为C函数所定义的变量；关键词__asm__表示汇编代码的开始。括弧中第一个引号中为汇编指令movl，紧接着有一个冒号，这种形式阅读起来比较复杂。

&emsp;&emsp;一般而言，嵌入式汇编语言片段比单纯的汇编语言代码要复杂得多，因为这里存在怎样分配和使用寄存器，以及把C代码中的变量应该存放在哪个寄存器中的问题。为了达到这个目的，就必须对一般的C语言进行扩充，增加对编译器的指导作用，因此，嵌入式汇编看起来晦涩而难以读懂。

**1. 嵌入式汇编的一般形式：**
 
    __asm__ __volatile__ ("<asm routine>" : output : input : modify);
 
&emsp;&emsp;其中，__asm__表示汇编代码的开始，其后可以跟__volatile__（这是可选项），其含义是避免“asm”指令被删除、移动或组合；然后就是小括弧，括弧中的内容是我们介绍的重点：
&emsp;&emsp;·"<asm routine>"为汇编指令部分，例如，"movl %%cr0,%0\n\t"。数字前加前缀“％“，如％1，％2等表示使用寄存器的样板操作数。可以使用的操作数总数取决于具体CPU中通用寄存器的数量，如Intel可以有8个。指令中有几个操作数，就说明有几个变量需要与寄存器结合，由gcc在编译时根据后面输出部分和输入部分的约束条件进行相应的处理。由于这些样板操作数的前缀使用了”％“，因此，在用到具体的寄存器时就在前面加两个“％”，如%%cr0。

&emsp;&emsp;·输出部分（output），用以规定对输出变量（目标操作数）如何与寄存器结合的约束（constraint）,输出部分可以有多个约束，互相以逗号分开。每个约束以“＝”开头，接着用一个字母来表示操作数的类型，然后是关于变量结合的约束。例如，上例中：

&emsp;&emsp;:"=r" (__dummy)

&emsp;&emsp;“＝r”表示相应的目标操作数（指令部分的%0）可以使用任何一个通用寄存器，并且变量__dummy 存放在这个寄存器中，但如果是：

&emsp;&emsp;：“＝m”（__dummy）

&emsp;&emsp;“＝m”就表示相应的目标操作数是存放在内存单元__dummy中。

&emsp;&emsp;表示约束条件的字母很多，表 2.3 给出几个主要的约束字母及其含义：

> 表2.3  主要的约束字母及其含义

   |字母|	含义|
   |-----|-----|
  | m, v,o	|表示内存单元
   |R|	表示任何通用寄存器
   |Q	|表示寄存器eax, ebx, ecx,edx之一
   |I, h	|表示直接操作数
   |E, F	|表示浮点数
   |G	|表示“任意”
   |a, b.c d	|表示要求使用寄存器eax/ax/al, ebx/bx/bl, ecx/cx/cl或edx/dx/dl
   |S, D	|表示要求使用寄存器esi或edi
  | I	|表示常数（0～31）|



-     输入部分（Input）：输入部分与输出部分相似，但没有“＝”。如果输入部分一个操作数所要求使用的寄存器，与前面输出部分某个约束所要求的是同一个寄存器，那就把对应操作数的编号（如“1”，“2”等）放在约束条件中，在后面的例子中，我们会看到这种情况。
-     修改部分（modify）:这部分常常以“memory”为约束条件，以表示操作完成后内存中的内容已有改变，如果原来某个寄存器的内容来自内存，那么现在内存中这个单元的内容已经改变。
 
&emsp;&emsp;注意，指令部分为必选项，而输入部分、输出部分及修改部分为可选项，当输入部分存在，而输出部分不存在时，分号“：“要保留，当“memory”存在时，三个分号都要保留，例如函数static inline void native_irq_disable()：

    static inline void native_irq_disable(void)
    {
        asm volatile("cli": : :"memory");
    }

**2.  Linux源代码中嵌入式汇编举例**

&emsp;&emsp;Linux源代码中，在arch目录下的.h和.c文件中，很多文件都涉及嵌入式汇编，下面以/arch/x86/include/asm/irqflags.h中的C函数为例，说明嵌入式汇编的应用。

&emsp;&emsp;（1）简单应用

     static inline unsigned long native_save_fl(void)
     {
        unsigned long flags;

        /*
         * "=rm" is safe here, because "pop" adjusts the stack before
         * it evaluates its effective address -- this is part of the
         */
        asm volatile("# __raw_save_flags\n\t"
                     "pushf ; pop %0"
                     : "=rm" (flags)
                     : /* no input */
                     : "memory");

        return flags;
    }
    
    static inline void native_restore_fl(unsigned long flags)
    {
        asm volatile("push %0 ; popf"
                     : /* no output */
                     :"g" (flags)
                     :"memory", "cc");
    }

&emsp;&emsp;第一个宏是保存标志寄存器的值，第二个宏是恢复标志寄存器的值。第一个宏中的pushfl指令是把标志寄存器的值压栈。而popl是把栈顶的值（刚压入栈的flags）弹出到x变量中，这个变量可以存放在一个寄存器或内存中。这样，你可以很容易地读懂第二个宏。

&emsp;&emsp;(2) 较复杂应用

    static inline unsigned long get_limit(unsigned long segment)
    {
        unsigned long __limit;
        asm("lsll %1,%0" : "=r" (__limit) : "r" (segment));
        return __limit + 1;
    }
    
&emsp;&emsp;这是一个设置段界限的函数，汇编代码段中的输出参数为__limit（即%0），输入参数为segment（即%1）。lsll是加载段界限的指令，即把segment段描述符中的段界限字段装入某个寄存器（这个寄存器与__limit结合），函数返回__limit加1，即段长。

&emsp;&emsp;（3）复杂应用

&emsp;&emsp;在Linux内核代码中，有关字符串操作的函数都是通过嵌入式汇编完成的，因为内核及用户程序对字符串函数的调用非常频繁，因此，用汇编代码实现主要是为了提高效率（当然是以牺牲可读性和可维护性为代价的）。在此，我们仅列举一个字符串比较函数strcmp，其代码在/arch/x86/lib/string_32.c中。

    int strcmp(const char *cs, const char *ct)
    {
        int d0, d1;
        int res;
        asm volatile("1:\tlodsb\n\t"
                "scasb\n\t"
                "jne 2f\n\t"
                "testb %%al,%%al\n\t"
                "jne 1b\n\t"
                "xorl %%eax,%%eax\n\t"
                "jmp 3f\n"
                "2:\tsbbl %%eax,%%eax\n\t"
                "orb $1,%%al\n"
                "3:"
                : "=a" (res), "=&S" (d0), "=&D" (d1)
                : "1" (cs), "2" (ct)
                : "memory");
        return res;
    }

&emsp;&emsp;其中的“\n”是换行符，“\t”是tab符，在每条命令的结束加这两个符号，是为了让gcc把嵌入式汇编代码翻译成一般的汇编代码时能够保证换行和留有一定的空格。例如，上面的嵌入式汇编会被翻译成：

&emsp;&emsp;**1：**

    **lodsb**           //装入串操作数,即从[esi]传送到al寄存器，然后esi指向串中下一个元素

    **scasb**           //扫描串操作数，即从al中减去es:[edi]，不保留结果，只改变标志

    **jne2f**           //如果两个字符不相等，则转到标号2 
  
    **testb %al  %al**  

    **jne 1b**

    **xorl %eax %eax**

    **jmp 3f**

&emsp;&emsp;**2: **

    **sbbl %eax %eax**

    **3**

    **orb $1 %al**

&emsp;&emsp;这段代码看起来非常熟悉，读起来也不困难。其中1f 表示往前（forword）找到第一个标号为1的那一行，相应地，1b表示往后找。其中嵌入式汇编代码中输出和输入部分的结合情况为：



-     返回值__res，放在al寄存器中，与%0相结合；
-     局部变量d0，与％1相结合，也与输入部分的cs参数相对应，也存放在寄存器ESI中，即ESI中存放源字符串的起始地址。
-     局部变量d1， 与％2相结合，也与输入部分的ct参数相对应，也存放在寄存器EDI中，即EDI中存放目的字符串的起始地址。

&emsp;&emsp;通过对这段代码的分析我们应当体会到，万变不利其本，嵌入式汇编与一般汇编的区别仅仅是形式，本质依然不变。

