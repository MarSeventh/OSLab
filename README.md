# Lab5实验报告

## 一、思考题汇总

### 1、如果通过 kseg0 读写设备，那么对于设备的写入会缓存到 Cache 中。这是 一种错误的行为，在实际编写代码的时候这么做会引发不可预知的问题。请思考：这么做 这会引发什么问题？对于不同种类的设备（如我们提到的串口设备和 IDE 磁盘）的操作会 有差异吗？可以从缓存的性质和缓存更新的策略来考虑。 

```
1、可能引发以下问题：
第一，数据不一致。设备数据与cache上的数据不一致，可能导致程序读取或者写入数据时结果与预期不符。
第二，性能问题。Cache缓存的是最近访问的数据，如果频繁读取设备，会导致Cache失效，需要从内存中重新加载数据，降低程序执行效率。
2、有差异：
对于串口设备：需要保证数据的及时性，不能将数据写入缓存中。
对于IDE磁盘：由于磁盘允许一定程度的延迟，因此可以写入缓存中，但是要注意缓存的更新机制，避免出现数据不一致的情况。
```

### 2、查找代码中的相关定义，试回答一个磁盘块中最多能存储多少个文件控制块？一个目录下最多能有多少个文件？我们的文件系统支持的单个文件最大为多大？ 

```
1、一个磁盘块最多存储BY2BLK / BY2FILE = 4096 / 256 = 16个文件控制块
2、一个目录下最多有BY2BLK / 4 * FILE2BLK = 4096 / 4 * 16 = 2^14个文件
3、文件系统支持的单个文件最大为BY2BLK / 4 * 4K = 4096KB = 4MB
```

### 3、请思考，在满足磁盘块缓存的设计的前提下，我们实验使用的内核支持的最 大磁盘大小是多少？

```
由于进程虚拟地址空间共4GB，而缓冲区为0x10000000-0x4fffffff，占总空间的1/4，所以支持最大磁盘空间为1GB。
```

### 4、在本实验中，fs/serv.h、user/include/fs.h 等文件中出现了许多宏定义， 试列举你认为较为重要的宏定义，同时进行解释，并描述其主要应用之处。

```c
1、fs/serv.h：
    #define PTE_DIRTY 0x0002 //定义虚拟地址第二位为脏位，该位为1说明缓冲区内容和磁盘不一致
    #define BY2SECT 512 //定义一个扇区的字节数
    #define SECT2BLK (BY2BLK / BY2SECT) //定义一个磁盘块的字节数
    #define DISKMAP 0x10000000 //定义磁盘缓冲区的起始地址
    #define DISKMAX 0x40000000 //定义磁盘缓冲区的最大容量
2、user/include/fs.h
    #define BY2BLK BY2PG //定义一个磁盘块的大小（4096B）
    #define MAXNAMELEN 128 //定义文件名的最大长度
    #define MAXPATHLEN 1024 //定义文件路径的最大长度
    #define NDIRECT 10 //定义了一个文件控制块直接指针的个数
    #define MAXFILESIZE (NINDIRECT * BY2BLK) //定义了一个文件的最大大小
    #define BY2FILE 256 //定义了一个文件控制块的大小
```

### 5、在 Lab4“系统调用与 fork”的实验中我们实现了极为重要的 fork 函数。那 么 fork 前后的父子进程是否会共享文件描述符和定位指针呢？请在完成上述练习的基础上 编写一个程序进行验证。

```c
为了验证fork前后的父子进程是否会共享文件描述符和定位指针，我们在test中增加以下代码：
160         int fd = open(file1, O_RDONLY);
161         char buf[1024];
162         read(fd, buf, 1);
163         int pd = fork();
164         if(pd > 0) {
165                 debugf("father fd = %d;\n",fd);
166         } else if(pd == 0) {
167                 debugf("son fd = %d;\n",fd);
168         }
169         close(fd);
```

在这段代码中，我首先打开file1这个文件，将其文件描述符存入fd中，之后执行fork函数创造子进程，之后通过pd的不同来输出父进程和子进程的fd序号，输出结果如下：

```
init.c: mips_init() is called
Memory size: 65536 KiB, number of pages: 16384
to memory 80430000 for struct Pages.
pmap.c:  mips vm init success
FS is running
superblock is good
read_bitmap is good
father fd = 0;
son fd = 0;
[00000800] destroying 00000800
[00000800] free env 00000800
i am killed ... 
[00001802] destroying 00001802
[00001802] free env 00001802
i am killed ... 
panic at sched.c:43 (schedule): schedule: no runnable envs
```

从中可以看到，father和son的文件描述符编号是相同的，说明fork前后的父子进程会共享文件描述符和定位指针。

### 6、请解释 File, Fd, Filefd 结构体及其各个域的作用。比如各个结构体会在哪 些过程中被使用，是否对应磁盘上的物理实体还是单纯的内存数据等。说明形式自定，要 求简洁明了，可大致勾勒出文件系统数据结构与物理实体的对应关系与设计框架。 

```c
1 struct File {
2 char f_name[MAXNAMELEN]; // 文件名
3 uint32_t f_size; // 文件大小（字节）
4 uint32_t f_type; // 文件类型，有FTYPE_REG和FTYPE_DIR两种
5 uint32_t f_direct[NDIRECT]; //文件的十个直接直接指针
6 uint32_t f_indirect; //文件的间接指针
7
8 struct File *f_dir; // 该文件目录的文件指针地址（仅在内存中生效）
9 char f_pad[BY2FILE - MAXNAMELEN - (3 + NDIRECT) * 4 - sizeof(void *)]; //占位，使一个文件结构体占256B
10 } __attribute__((aligned(4), packed));
```

File结构体中，各个属性的定义已在注释中标出。File结构体中主要存储文件或者目录的基本信息，在对特定文件进行读写操作时都会被使用，对应磁盘上的物理实体。

```c
1 struct Fd {
2 u_int fd_dev_id; //文件描述符对应文件所在的设备编号
3 u_int fd_offset; //文件描述符对应文件内部的偏移量（已操作部分）
4 u_int fd_omode; // 文件的打开方式
5 };
6
7 struct Filefd {
8 struct Fd f_fd; //文件描述符
9 u_int f_fileid; //打开文件的编号
10 struct File f_file; //打开文件的文件结构体
11 };
```

Fd结构体和Filefd结构体中各个属性的定义已在注释中标出。这两个结构体主要用来记录进程已打开文件的信息，便于对文件进行操作，在fopen时分配，在fclose时释放。

### 7、图5.7中有多种不同形式的箭头，请解释这些不同箭头的差别，并思考我们 的操作系统是如何实现对应类型的进程间通信的。

![](https://cdn.jsdelivr.net/gh/MarSeventh/imgbed/posts/image-20230517160247833.png)

图中，实心箭头表示调用关系，空心箭头表示被调用者将结果返回给调用者。

我们的操作系统是通过系统调用的方式实现进程间通信的，首先如果一个进程要调用另一个进程的服务，他需要通过系统调用将有关参数放入内核中，然后内核再传递给被调用的进程，被调用进程按照调用者要求实现有关功能（如文件操作），将返回值存入调用者指定的虚拟地址，再通过IPC传递给调用者。

## 二、实验难点总结

本次实验主要分为三部分，分别是IDE磁盘交互、文件系统的启动、文件系统服务（如open、read、write、close等）的实现。这三个部分环环相扣，逐层递进，前两个部分是文件系统服务的基础，最后一部分则是向用户提供操作文件的接口。下面分别对这三部分的重难点进行分析。

### 2.1IDE磁盘交互

在这一部分，我们首先要掌握内存映射I/O(MMIO)的相关概念，了解每一种外设都是通过读写外设寄存器来和CPU进行通信的，并且外设寄存器都被映射到了指定的物理地址空间，从而我们就可以通过访问指定的物理地址来访问外设。知道了这一点之后，我们还需要了解磁盘的基本组成结构(如下图)。

![](https://cdn.jsdelivr.net/gh/MarSeventh/imgbed/posts/image-20230517162801974.png)

在了解了这些基础知识之后，我们便可以编写IDE磁盘的驱动程序，通过访问指定物理地址来实现磁盘的读写操作。

总体来说，这一部分难度不大，只要掌握好**抽象化**这个关键思想，将磁盘抽象为一个文件进行访问即可。

### 2.2文件系统的启动

这一部分的主要任务就是基于已经实现的IDE驱动程序，建立相应的管理文件的数据结构，建立内存和磁盘之间的联系，然后把我们的文件系统进程启动起来。

具体来说，我们首先需要将磁盘初始化为我们想要的样子，也就是将它从一个物理概念中剥离出来，抽象为一块虚拟的空间。比如，将多个扇区合并为一个逻辑上的磁盘块，并且按照下图初始化前几个磁盘块。

![](https://cdn.jsdelivr.net/gh/MarSeventh/imgbed/posts/image-20230517164032224.png)

完成了这一部分之后，我们的磁盘已经初具雏形，但是内部的文件还是一盘散沙，没有有效地组织起来，所以我们还需要对文件建立有关的数据结构，方便我们的文件系统对磁盘中的文件进行具体操作。于是我们建立了File这个结构体，让磁盘中的每一个文件都有一个这样的结构体来对它进行标识和管理。

但是这样还不够，我们的CPU还是不能方便地从磁盘中读取我们需要的文件。我们知道，CPU的操作是和内存打交道的，我们访问的都是内存中的地址，所以如果想要访问磁盘中的文件，还要在内存中开辟一块空间来映射磁盘中的内容，这就是我们的**块缓存**，在建立好磁盘和内存的映射关系之后，我们如果想要访问磁盘中的某个文件，就只需要访问这个磁盘块对应的虚拟地址即可。

这一部分较难理解的点其实就是如何将磁盘中的文件和我们的CPU联系起来，便于CPU的访问。要掌握这一点，关键还是要理解**抽象**这个思想。

完成这三个步骤之后，文件系统就能够正常启动了，下一部分只要实现用户操作文件的接口，就大功告成了。

### 2.3文件系统服务的实现

由于MOS系统采用微内核的设计，所以文件系统和我们的用户进程实际上是两个同类型的进程，所以用户要对磁盘中的文件进行操作，就需要经过**用户进程->文件系统->内核->磁盘->文件系统->用户进程**的过程，其中无论是文件系统和用户进程的交互还是文件系统和内核的交互，都需要通过IPC机制实现。当然，又因为用户进程访问一个文件之前除了这个文件的名字和我需要的打开方式之外一无所知，所以文件系统首先要从磁盘中取出文件的有关信息，将它读入内存中，再把这块内存通过IPC传递给用户进程，之后用户进程才能根据内存中的文件信息进行下一步操作。也就是说，上述的交互流程需要多次进行，那么这就存在一个问题，我们需要建立一个新的数据结构来在内存中存储待访问文件的有关信息，这就引入了Fd和Filefd两个结构体，也就是我们所说的**文件描述符**。简单地说，Fd就是Filefd在文件系统还没有从磁盘中读出文件信息时的一个简略版，读入之后再将File结构体等信息存入Fd中，从而形成了完整的Filefd以供用户进程进行访问和操作。

上述过程用时序图表示就是下图：

![](https://cdn.jsdelivr.net/gh/MarSeventh/imgbed/posts/image-20230517170603863.png)

这一部分的难点也就是理解这个时序图以及文件描述符的实际作用，清楚这些之后实现文件系统用户接口也就比较轻松了。

## 三、实验心得体会

这次实验的难点主要在理解层面，需要时刻记住**抽象**这个思想，无论是将IDE磁盘抽象为一块逻辑空间，还是建立文件控制块，实现虚拟地址和磁盘的映射，亦或是建立文件描述符描述已打开的文件，其实都是对磁盘中存储内容的一次次抽象，将其逻辑化为方便我们操作和使用的逻辑数据。

只要我们掌握好这个思想，一步步去将磁盘数据抽象出来，我们就比较容易实现对磁盘的操作了。当然，对于其他外设的操作也是这样，我们必须要针对不同外设的特点来逐层建立各种数据结构，才能使得对一种外设的操作最简化。在这个实现过程中，需要我们在思想角度上从全局出发，不拘泥于微小的细节，应当首先建立起整个I/O操作的架构，然后分析其中每一层应该如何抽象和管理，有了整体架构之后才能更好地实现对外设的操作。
