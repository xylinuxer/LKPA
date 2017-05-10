## **2.4  Linux中的分页机制**
 
&emsp;&emsp;如前所述，Linux主要采用分页机制来实现虚拟存储器管理。这是因为：

&emsp;&emsp;（1）Linux的分段机制使得所有的进程都使用相同的段寄存器值，这就使得内存管理变得简单，也就是说，所有的进程都使用同样的线性地址空间（0～4G）。

&emsp;&emsp;（2）Linux设计目标之一就是能够把自己移植到绝大多数流行的处理器平台。但是，许多RISC处理器支持的段功能非常有限。

&emsp;&emsp;为了保持可移植性，Linux采用三级分页模式而不是两级，这是因为许多处理器（如康柏的Alpha，Sun的UltraSPARC，Intel的Itanium）都采用64位结构的处理器，在这种情况下，两级分页就不适合了，必须采用三级分页。图2.17为三级分页模式，为此，Linux定义了三种类型的页表：

- 	页总目录PGD（Page Global Directory）
- 	页中间目录PMD（Page Middle Derectory）
- 	页表PT（Page Table）
             
![](http://i.imgur.com/ua66uMo.png)

&emsp;&emsp;尽管Linux采用的是三级分页模式，但我们的讨论还是以80X86的两级分页模式为主，因此，Linux忽略中间目录层，以后，我们把页总目录就叫页目录。

&emsp;&emsp;我们将在第四章看到，每一个进程有它自己的页目录和自己的页表集。当进程切换发生时（参见第三章“进程切换”一节），Linux把cr3控制寄存器的内容保存在前一个执行进程的PCB中，然后把下一个要执行进程的PCB的值装入cr3寄存器中。因此，当新进程恢复在CPU上执行时，分页单元指向一组正确的页表。

&emsp;&emsp;当三级分页模式应用到只有两级页表的奔腾处理器上时会发生什么情况？Linux使用一系列的宏来掩盖各种平台的细节。例如，通过把PMD看作只有一项的表，并把它存放在pgd表项中（通常pgd表项中存放的应该是pmd表的首地址）。页表的中间目录(pmd)被巧妙地“折叠”到页表的总目录(pgd)中，从而适应了二级页表。


### **2.4.1 模拟页表初始化**

&emsp;&emsp;在了解页表基本原理之后，通过代码来模拟内核初始化页表的过程：

    #define NR_PGT 0x4
    #define PGD_BASE (unsigned int*)0x1000
    #define PAGE_OFFSET (unsigned int)0x2000

    #define PTE_PRE 0x01 /* Page present */
    #define PTE_RW  0x02 /* Page Readable/Writeable */
    #define PTE_USR 0x04 /* User Privilege Level*/ 

    void page_init()
    {
       int pages = NR_PGT;                     // 系统初始化创建4个页表

       unsigned int page_offset = PAGE_OFFSET;
       unsigned int* pgd = PGD_BASE;           // 页目录表位于物理内存的第二个页框内
       unsigned int phy_add = 0x0000;          // 在物理地址的最低端建立页机制所需的表格

       // 页表从物理内存的第三个页框处开始
       // 物理内存的头8KB没有通过页表映射
       unsigned int* pgt_entry = (unsigned int*)0x2000;
  
       while (pages--)// 填充页目录表,这里依次创建4个页目录表
       {   
           *pgd++ = page_offset |PTE_USR|PTE_RW|PTE_PRE;
           page_offset += 0x1000;
       }   

       pgd = PGD_BASE;

  
       while (phy_add < 0x1000000) {           // 在页表中填写页到物理地址的映射关系，映射了4M大小的物理内存
           *pgt_entry++ = phy_add |PTE_USR|PTE_RW|PTE_PRE;
           phy_add += 0x1000;
       }

  
       __asm__ __volatile__ ("movl %0, %%cr3;"
                             "movl %%cr0, %%eax;"
                             "orl  $0x80000000, %%eax;"
                             "movl %%eax, %%cr0;"::"r"(pgd):"memory","%eax");
    }

&emsp;&emsp;从代码中可以看出在物理内存的第二个页框设置了页目录，然后的while循环初始化了页目录中的四个页目录项，即四个页表。紧接着的第二个while循环初始化了四个页表中的第一个，其它三个没有用到，映射了4MB的物理内存，至此页表已初始化好，剩下的工作就是将页目录的地址pgd传递给cr3寄存器，这由gcc嵌入式代码部分完成，并且设置了cr0寄存器中的分页允许。最后一句以__asm开头的嵌入式汇编参见2.5.3节。

&emsp;&emsp;虽然上述代码比较简单，但却描述了页表初始化的过程，以此为模型我们可以更容易理解Linux内核中关于页表的代码。

