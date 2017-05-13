## 8.5编写一个文件系统

&emsp;&emsp;文件系统比较庞杂，很难编写一个恰当的例子来演示文件系统的实现。如果写一个纯虚的文件系统(没有存储设备)，因为虚文件系统不涉及I/O操作，缺少实现文件系统中至关重要的部分，因此必要性不大；如果写一个实际文件系统，但是涉及的东西太多，不容易让读者简明扼要的理解文件系统的实现。幸好，内核中提供的romfs文件系统是个非常理想的实例，它既有实际应用结构，也清晰明了，因此我们以romfs为实例分析文件系统的实现。

### 8.5.1 Linux文件系统的实现要素

&emsp;&emsp;编写新文件系统涉及一些基本对象，具体地说，需要建立“**一个结构四个操作表**”：

-  文件系统类型结构（file\_system\_type）

-  超级块操作表（super\_operations）

-  索引节点操作表（inode\_operations）

-  页缓冲区表（address\_space\_operations）

-  文件操作表（file\_operations）

&emsp;&emsp;对上述几种结构的操作贯穿了文件系统实现的主要过程，理清晰这几种结构之间的关系是编写文件系统的基础，如图8.10所示，下来我们具体分析这几个结构和文件系统实现的要点。

<div align=center>
<img src="图8_10.png" />  
</div>

<div align=center>
图8.10 一个结构及四个操作表之间的关系
</div>

&emsp;&emsp;首先，必须建立一个文件系统类型(file\_system\_type)来描述文件系统，它含有文件系统的名称、类型标志以及get\_sb()等操作。当安装文件系统时，系统会对该文件系统进行注册，即填充file\_system\_type结构，然后调用get\_sb()函数来建立该文件系统的超级块。注意对于基于块的文件系统，如Ext4、romfs等，需要从文件系统的宿主设备读入超级块，然后在内存中建立对应的超级块，如果是虚文件系统（如proc文件系统），则不读取宿主设备的信息（因为它没有宿主设备），而是在现场创建一个超级块，这项任务也由get\_sb()完成。

&emsp;&emsp;可以说，超级块是一切文件操作的鼻祖，因为超级块是我们寻找索引节点的唯一源头。我们操作文件必然需要获得其对应的索引节点（或从宿主设备读取或现场建立），而获取索引节点是通过超级块操作表提供的read\_inode()函数完成的。同样操作索引节点的底层次任务，如创建一个索引节点、释放一个索引节点，也都是通过超级块操作表提供的有关函数完成的。所以超级块操作表(super\_operations)是第二个需要创建的数据结构。

&emsp;&emsp;除了派生或释放索引节点等操作是由超级块操作表中的函数完成外，索引节点还需要许多自操作函数，比如lookup()搜索索引节点，建立符号链接等，这些函数都包含在索引节点操作表中，因此索引节点操作表(inode\_operations)是第三个需要创建的数据结构。

&emsp;&emsp;为了提高文件系统的读写效率，Linux内核设计了I/O缓存机制。所有的数据无论出入都会经过系统管理的缓冲区，不过，对基于非块的文件系统则可跳过该机制。页缓冲区同样提供了页缓冲区操作表（address\_space\_operations），其中包含有readpage()、writepage()等函数负责对页缓冲区中的页进行读写等操作。

&emsp;&emsp;文件系统最终要和用户交互，这是通过文件操作表（file\_operations）完成的，该表中包含有关用户读写文件、打开、关闭、映射文件等用户接口。

&emsp;&emsp;一般来说，基于块的文件系统的实现都离不开以上5种数据结构。但根据文件系统的特点（如有的文件系统只可读、有的没有目录），并非要实现操作表中的全部函数，因为有的函数系统已经实现，而有的函数不必实现。

### 8.5.2 什么是Romfs文件系统

&emsp;&emsp;Romfs是一种相对简单、占用空间较少的文件系统。

&emsp;&emsp;空间的节约来自于两个方面：首先内核支持Romfs文件系统比支持
Ext4文件系统需要更少的代码；其次Romfs文件系统相对简单，在建立文件系统超级块（Superblock）需要更少的存储空间。

&emsp;&emsp;Romfs是只读的文件系统，禁止写操作，因此系统同时需要虚拟盘（RAMDISK）支持临时文件和数据文件的存储。

&emsp;&emsp;Romfs是在嵌入式设备上常用的一种文件系统，具备体积小，可靠性好，读取速度快等优点。同时支持目录，符号链接，硬链接，设备文件。但也有其局限性。Romfs是一种只读文件系统，同时由于Romfs本身设计上的原因，使得Romfs支持的最大文件不超过256M。Linux,
uclinux都支持Romfs文件系统。除Romfs外，其它常用的嵌入式设备的文件系统还有CRAMFS，JFFS2等，它们各有特色。

&emsp;&emsp;下面我们来分析它的实现方法，为读者勾勒出编写新文件系统的思路和步骤。

### 8.5.3 Romfs文件系统布局与文件结构

&emsp;&emsp;设计一个文件系统首先要确定它的数据存储方式。不同的数据存储方式对文件系统占用空间，读写效率，查找速度等主要性能有极大影响。Romfs是一种只读的文件系统，它使用顺序存储方式，所有数据都是顺序存放的。因此Romfs中的数据一旦确定就无法修改，这是Romfs只能是一种只读文件系统的原因，它的数据存储方式决定了无法对Romfs进行写操作。由于采用了顺序存放策略，Romfs中每个文件的数据都能连续存放，读取过程中只需要一次寻址操作，进而就可以读入整块数据，因此Romfs中读取数据效率很高。

&emsp;&emsp;在Linux内核源代码的Document/fs/romfs中介绍了romfs文件系统的布局和文件结构,如图8.11所示。


<div align=center>
<img src="图8_11.jpg" />  
</div>

<div align=center>
图 8.11 romfs文件系统布局
</div>

&emsp;&emsp;从图可以看出，文件系统中每部分信息都是16字节对齐的，也就是说存储的偏移量必须最后4位为0，这样作是为了提高访问速度。如果信息不足时，需要填充0以保证所有信息的开始位置都以16来对齐。

&emsp;&emsp;文件系统的开始8个字节存储文件系统的ASCII形式的名称，在这里是”-rom1fs-”；接着4个字节记录文件大小；然后的4个字节存储的是文件系统开始处512字节的检验和；接下来是卷名；最后是第一个文件的文件头，从这里开始依次存储的就是文件本身的信息了。

&emsp;&emsp;Romfs的文件结构如图8.12所示。


<div align=center>
<img src="图8_12.jpg" />  
</div>

<div align=center>
图8.12 Romfs的文件结构
</div>

### 8.5.4具体实现的对象

&emsp;&emsp;针对文件系统布局和文件结构，
Romfs文件系统定义了一个磁盘超级块结构和文件的inode结构，磁盘超级块的结构定义如下：

```c
    struct romfs_super_block {
	__be32 word0;
	__be32 word1;
	__be32 size;
	__be32 checksum;
	char name[0];		  
    };
```

&emsp;&emsp;该结构用于识别整个Romfs文件系统，大小为512字节。word0初始值为'-','r','o','m'，word1初始值为
'-','1','f','s'，通过这两个字操作系统确定这是一个Romfs文件系统。size字段用于记录整个文件系统的大小，理论上Romfs大小最多可以达到4G。checksum是前512字节的校验和，用于确认整个文件系统结构数据的正确性。前面4个字段占用了16字节，剩下的都可以用作文件系统的卷名，如果整个首部不足512字节便用0填充，以保证首部符合16字节对齐的规则。
```c
    struct romfs_inode {
	__be32 next;		
	__be32 spec;
	__be32 size;
	__be32 checksum;
	char name[0];
};
```

&emsp;&emsp;next
字段是下一个文件的偏移地址，该地址的后4位是保留的，用于记录文件模式信息，其中前两位为文件类型，后两位则标识该文件是否为可执行文件。因此Romfs用于文件寻址的字段实际上只有28bit，所以Romfs中文件大小不能超过256M。spec字段用于标识该文件类型。目前Romfs支持的文件类型包括普通文件，目录文件，符号链接，块设备和字符设备文件。size是文件大小，checksum是校验和，校验内容包括文件名，填充字段。name是文件名首地址，文件长度必须保证16字节对齐，不足的部分可以用0填充。

&emsp;&emsp;上述两种结构分别描述了文件系统结构与文件结构，它们将在内核装配超级块对象和索引节点对象时被使用。

&emsp;&emsp;Romfs文件系统首先要定义的对象是文件系统类型romfs\_fs\_type：
```c
    static DECLARE_FSTYPE_DEV(romfs_fs_type, "romfs", romfs_read_super);
```

&emsp;&emsp;DECLARE\_FSTYPE\_DEV是内核定义的一个宏，用来建立file\_system\_type结构。从上面的申明可以看出，file\_system\_type结构的类型变量为romfs\_fs\_type，文件系统名为”romfs”，读超级块的函数为romfs_read_super()。

&emsp;&emsp;Romfs\_read\_super()从磁盘读取磁盘超级块，主要步骤如下:

&emsp;&emsp;1.装配超级块，具体步骤为：

&emsp;&emsp;(1).初始化超级块对象某些域。

&emsp;&emsp;(2).从设备中读取磁盘第0块到内存，即调用函数bread(dev,0,ROMBSIZE)，其中dev是文件系统安装时指定的设备，0为设备的首块，也就是磁盘超级块，ROMBSIZE是读取的大小。

&emsp;&emsp;(3).检验磁盘超级块中的校验和。

&emsp;&emsp;(4).继续初始化超级块对象某些域。

&emsp;&emsp;2.给超级块对象的操作表赋值（s-\>s\_op = &romfs\_ops）

&emsp;&emsp;3.给根目录分配目录项。

&emsp;&emsp;其中，romfs文件系统在超级块操作表中实现了两个函数：

```c
    static struct super_operations romfs_ops = {
            read_inode:     romfs_read_inode,
            statfs:         romfs_statfs,
    };
```

&emsp;&emsp;第一个函数romfs\_read\_inode(inode)是从磁盘读取数据填充参数指定的索引节点,主要步骤如下：

&emsp;&emsp;1.根据inode参数寻找对应的索引节点。

&emsp;&emsp;2.初始化索引节点某些域

&emsp;&emsp;3.根据文件类型设置索引节点的相应操作表

&emsp;&emsp;(1).如果是目录文件，则将索引节点表设为i->i_op = &romfs_dir_inode_operations;文件操作表设置为i->i_fop = &romfs_dir_operations; 这两个表分别定义如下：
```c
static struct inode_operations romfs_dir_inode_operations = {
        lookup:         romfs_lookup,
};
static struct file_operations romfs_dir_operations = {
        read:           generic_read_dir,
        readdir:        romfs_readdir,
};
```
&emsp;&emsp;(2).如果是常规文件，则将文件操作表设置为i->i_fop = &generic_ro_fops; 
对常规文件的操作也只需要使用内核提供的通用函数表，它包含三种基本的常规文件操作：
```c
struct generic_ro_fops{
        llseek:         generic_file_llseek,
        read:           generic_file_read,
        mmap:           generic_file_mmap,
}
```

&emsp;&emsp;(3).将页缓冲区表设置为i->i_data.a_ops = &romfs_aops;  
```c
static struct address_space_operations romfs_aops = {
        readpage: romfs_readpage
};
```

&emsp;&emsp;回忆前面我们描述过的页缓冲区，显然常规文件访问需要经过它，因此有必要实现页缓冲区操作。因为只需要读文件，所以只用实现romfs_readpage函数，这里readpage函数完成将数据从设备读入页缓冲区，该函数根据文件格式从设备读取需要的数据。设备读取操作需要使用bread块I/O例程，它的作用是从设备读取指定的块。

&emsp;&emsp;(4).由于romfs是只读文件系统，它在对常规文件操作时不需要索引节点操作，如mknod，link等，因此使用索引节点的操作表。

&emsp;&emsp;(5).如果是连接文件，则将索引节点操作表设置为：i->i_op=&page_symlink_inode_operations;

&emsp;&emsp;(6).如果是套接字或管道，则进行特殊文件初始化操作init_special_inode(i, ino, nextfh);

&emsp;&emsp;到此，我们已经遍例了romfs文件系统使用的几种对象结构：romfs_super_block、romfs_inode、romfs_fs_type、super_operations 、address_space_operations、file_operations、romfs_dir_operations、inode_operations、romfs_dir_inode_operations 。实现上述对象中的函数是设计一个新文件系统的最低要求，具体函数的实现请读者阅读源代码。

&emsp;&emsp;最后要说明的是,为了使得romfs文件系统作为模块安装，需要实现以下两个函数:

```c
    static int __init init_romfs_fs(void)
    {
            return register_filesystem(&romfs_fs_type);
    }
    static void __exit exit_romfs_fs(void)
    {
            unregister_filesystem(&romfs_fs_type);
    }
```

&emsp;&emsp;安装和卸载romfs文件系统的例程为：

```c
    module_init(init_romfs_fs)
    module_exit(exit_romfs_fs)
```

&emsp;&emsp;到此，介绍了romfs文件系统的主要结构，至于细节，需要读者仔细推敲。Romfs是最简单的基于块的只读文件系统，而且没有访问控制等功能。所以很多访问权限以及写操作相关的方法都不必去实现。
