### 计算机如何启动(bios)
BIOS先加电自检(键鼠、cpu等)，再初始化设备(IO端口地址、分配中断请求线等)，接下来加载引导程序并移交控制权。

## 中断与内核
### 硬中断软中断

中断处理程序的上部分和下半部可以理解为：
上半部直接处理硬件请求，也就是硬中断，主要是负责耗时短的工作，特点是快速执行；
下半部是由内核触发，也就说软中断，主要是负责上半部未完成的工作，通常都是耗时比较长的事情,特点是延迟执行；
硬中断是会打断 CPU 正在执行的任务，然后立即执行中断处理程序。
软中断是以内核线程的方式执行，并且每一个 CPU 都对应一个软中断内核线程。

比如网卡收到一个数据包：
上部分要做的事情很少，会先禁止网卡中断，避免频繁硬中断，而降低内核的工作效率。
接着，内核会触发一个软中断，把一些处理比较耗时且复杂的事情，交给「软中断处理程序」去做，
主要是需要从内存中找到网络数据，再按照网络协议栈对网络数据进行逐层解析和处理，最后把数据送给应用程序。

### 多核CPU关中断可以保证原子性吗
不能。
单核系统中，中断是唯一能打断当前执行指令的机制。关中断后，CPU不响应外部设备的中断请求，
也就能完整做完当前操作，保证原子性。
多核系统中，仅仅是阻止了该核心响应中断，但其他核心仍然可以正常运行，并且可能会访问和修改共享资源。

### 内存管理
操作系统为每个进程分配独立的一套「虚拟地址」，人人都有，大家自己玩自己的地址就。
那具体怎么映射呢？
- 内存分段。把程序分成若干个逻辑分段(比如代码一段、数据一段、栈一段)，每段分开放。

那就得知道每段对应到哪里，我们用段表来表示这个映射。段表由段基地址+段界限组成，一个段拿着自己的段号来表里找对应的基地址，
加上偏移量就能找到物理地址。比如代码段里需要访问偏移量为500的虚拟地址，在段表里找到代码段的基地址为7000，加上偏移量500=7500，7500就是物理地址了。

分段的思想很清晰，但容易有内存碎片，会产生多个不连续的小物理内存，导致新的程序无法被装载。而这些小物理内存的空间如果安排
得当，又是能连在一起放更大的进程的。这样内存空间无法连续利用的碎片叫外部内存碎片。解决手段是内存交换(Linux里的swap)，先把某程序占用的内存写到硬盘上，再读回来，不过
读取回来时不装到原来位置，而是紧贴着某块被占用的内存后面。这样就能保证空间尽量连续。

看起来很美好，但硬盘读写太慢了，所以分段从机制上就不行，接下来我们考虑内存分页。

- 内存分页。分页裁剪得更细致，不是按逻辑分段，而是把虚拟空间、内存空间都切成一段段固定的“页”，linux下页的大小是4KB。
这样映射时，虚拟内存的一个页对应到物理内存的一个页，页之间都是贴着的，就不会有外部碎片。但可能数据用不满4KB，所以就出现了
内部碎片。要关注的另一个问题是分页时怎么地址转换的？和分段有段表一样，分页也有页表。页表存着虚拟页对应物理页的基地址，基地址加上
偏移量就是真实地址。
看起来很美好，但有大问题：页表会很大很大。考虑用多级页表解决：原本32位linux下每个进程拥有4G的虚拟地址，每个虚拟地址都需要映射，页大小位4KB，所以4G一共需要$4*1024*1024/4=1048576$这么多条。页表里每个项占用4Byte，所以一个页表要占$1048576*4(Byte)=4MB$。
考虑把这100多万项再分页，分成$1024(第一级)*1024(第二级)$，可以理解为原本是一维数组，现在压缩成了二维数组，但空间不变。
理论上二级表占用的空间是4KB(一级页表)+4MB(二级页表)好像还多了一级页表的开销，但事实上有“局部性原理”，其实我们用不了那么多空间。
所以如果一级页表的某个项没被用到，就不用创建对应二级页表，这部分就节省了很多空间。

热知识：64位的系统页表是4级的。

- 段页式管理。上述两种方法各有千秋，但不是对立的，实践中经常组合起来使用。先将程序划分为多个有逻辑意义的段，也就是前面提到的分段机制；接着再把每个段划分为多个页，也就是对分段划分出来的连续空间，再划分固定大小的页。这样，地址结构就由段号、段内页号和页内位移三部分组成。当然也要建立对应的段表、页表了。

### 虚拟内存有什么用
- 隔离进程
- 内存共享，方便通信
- 按需分配，不用将所有程序都一次性载入(单片机就是一次性烧入)
- 突破物理内存限制，通过将暂时不用的页面交换到磁盘上，2G的程序能在只有1G物理内存的系统上运行。
- 页表有读写权限、标记该页是否存在，提供更好的安全性
- 方便开发者吧
  
### 虚拟内存大于可用的物理内存会发生什么
- 页面置换：物理内存不足时，os会使用页面置换算法（如 FIFO、LRU 等）选择暂时不用的页面，将其从物理内存中换出到磁盘
- 页面交换：当进程需要访问被置换到磁盘的页面时，会发生页面错误（Page Fault）。os就得暂停当前进程，
将所需页面从磁盘交换回来，再恢复进程的执行。
整个过程涉及磁盘 I/O 操作，速度慢，会影响性能。

### 禁止换出到磁盘会发生什么
待定：内存不足、程序崩溃？

### Linux的内存空间布局
先说intel的设计方式，毕竟软件都是建立在硬件上的。Intel X86 CPU一律对程序先段映射，再页映射。Linux为了绕开段映射，每个段都是从 0 地址开始的整个 4GB 虚拟空间，可以理解为只有一个段。这样就屏蔽了处理器中的逻辑地址概念，段只被用于访问控制和内存保护。
通过这种方式，在intel x86上实现了页式内存管理。

### 进程的内存空间布局
- 用户空间和内核空间。虚拟地址空间的内部又被分为内核空间和用户空间两部分，32位下内核空间1G，用户空间3G。

- 两者区别是进程在用户态时，只能访问用户空间内存；只有进入内核态后，才可以访问内核空间的内存。虽然每个进程都各自有独立的虚拟内存，但是每个虚拟内存中的内核地址，其实关联的都是相同的物理内存(好比每个房间都有一个地道，通向最核心的内核空间，在里面完成系统调用)。这样，进程切换到内核态后，可以方便快捷地访问内核空间内存。

- 用户空间的内存分布：从高到低依次是内核空间、栈、文件映射、堆、BSS段、数据段、代码段、保留区。

冷知识：保留区是因为大多数的系统认为较小数值的地址不是一个合法地址。
所以我们通常在 C 的代码里将无效指针赋为 NULL。

### 内核的地址是什么
32 位系统的内核空间占用 1G，位于最高处，从0XC0000000-0XFFFFFFFF。
64 位系统的内核空间和用户空间都是 128T，占据整个内存空间的最高,从0XFFFF800000000000-0XFFFFFFFFFFFFFFFF。

### 用户态可以访问内核吗，为什么
不能。
- 不安全。能的话岂不是可以直接改内核的各种数据，比如进程调度表，恶意涂写数据。
但可以通过系统调用(open/read/write)、硬件中断(键盘输入、除零错误)切换到内核态。

## 进程
### 进程，线程
进程是正在运行的一段程序，线程是进程当中的一条执行流程。
线程最主要的特点是并发运行、共享相同的资源(代码段、数据、文件)但是各自有独立的寄存器和栈。
比如有份代码

```cpp
main(){
  while(1){
     read();
     unZip();
     Play();
  }
}
```
单进程播放的话，只能顺序执行，这显然是不对的，音画都不连贯了，事实上可以同时做。如果简单地改成多进程，那进程间该如何通信呢？(此处不表，见下一道)

所以这就引入了线程。开3个线程read()、unZip()、Play()并放到线程池里，
就能并发执行了。
比较进程和线程：
- 线程是调度的基本单位，进程是资源拥有的基本单位
- 线程更快，因为能共享资源，不需要太多额外的信息，所以创建快、终止快、切换快(共享页表)

### 线程池实现高并发任务队列


### 进程间通信方法
这个问题很大，尽量简单讲。本来进程的用户地址空间是独立的，不能互相访问，但所有进程
都共享内核空间，所以进程之间通信必须通过内核。
- 管道(内核里面的一串缓存)。linux命令里有个"|"，意思是把前者的输出作为后者的输入，这就算单向通信。
通信的话需要有来有往，管道的设计是让一个进程同时连管道的读端和写端，这怎么通信？
A：再fork一个子进程(fork能复制上一级进程的文件描述符)，这样两个进程都接到了管道里，就可以读写同一个管道了。你也可以简单地理解为它们约好了一个神秘地点，要写东西和拿东西都来这里，所以也特别慢。

- 消息队列(保存在内核中的消息链表)。A 进程要给 B 进程发送消息，A 把数据放在对应的消息队列后就可以返回，B 进程需要的时候再去读就行。和管道不同，匿名管道随着进程创建完就销毁，消息队列会随着内核一直在。消息就像邮件，不及时、有附件限制、数据拷贝开销(因为是保存在内核中的，所以用户态内核态之间读写时，肯定有拷贝开销)。

- 共享内存。本来每个进程都有独立的虚拟空间(当然了，进程的虚拟内存会映射到不同的物理内存，才不会互相影响)，共享就是拿出一块虚拟地址映射到相同的物理内存。这样一个进程一写入，另一个马上能看，不用拷贝。

- 信号量。既然共享，就会冲突。用信号量来实现任意时刻资源只能被一个进程访问的保护机制。信号量是一个int，有PV操作，在分类上也分为同步信号量和互斥信号量。P操作：把信号量-1，如果此时信号量<0，表示被占用；如果>=0，说明还有资源可用。V操作：把信号量+1，如果<=0，表明还有进程阻塞着，就把它叫起来；如果信号量>0，说明当前没进程阻塞了。使用时P操作在进入共享资源前，V操作在离开共享资源后，必须成对出现。接下来就互斥信号量和共享信号量举个例子：
**互斥信号量**(初始值为1)：①进程A执行P操作，执行后信号量为0，资源可用，A使用。②如果此时B想访问，也执行了P，信号量变成-1，
资源不能用，B被堵塞。③A用完后，V操作让信号量恢复为0，发现有进程阻塞，所以把B叫起来。④等B用完后，V操作又让信号量恢复到1。
**同步信号量**(初始值为0)：①B要用资源，执行完P发现信号量是-1，表示A还没产生数据，B就阻塞等待。②A产生完数据后，V操作让信号量变成0，唤起B。
所以同步信号量能保证进程A在进程B之前执行。

- socket通信。是的，不仅能在网络间通信，进程间也行。不过bind的不是IP地址和端口，而是绑定本地文件。
  
### 多线程同时调用方法查询，要求阻塞其他线程，只执行一次查询
call_once()和once_flag()


## Linux相关(命令、设计、概念)

### linux看各种参数
- 看服务器负载：htop(有交互界面)、uptime
- 看网络配置：ifconfig
- 看路由表、socket：netstat
- 看连通性和延时：ping
- 看PPS和网络吞吐率：sar
- 看内存：htop/top/ps
  
### linux用户组的概念
用户组是具有相同特征的用户集合，系统管理员可以将多个用户划分到一个组中，对组进行统一的权限设置和管理。
命令：
```bash
groupadd/del/mod xxx
```

## 网络系统
### I/O多路复用
- 概念：指在单线程或单进程环境下同时监控多个 I/O 事件的技术。
- 原理：使用一个系统调用(select、poll、epoll等)同时监听多个IO源的状态变化。
程序将需要监控的IO源(通常是文件描述符)注册到这个系统调用中，然后阻塞等待。
当其中任何一个或多个IO源发生变化(如可读、可写、出错等)时，系统调用会返回，并告知程序哪些IO源已经准备好进行相应的操作。
程序根据返回的信息，对准备好的IO源进行读写操作。

也就是说通过系统调用，起到一个监控的作用，同时监控很多IO。

### select/epoll了解过吗？
- select:
```cpp
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
参数依次是
nfds：需要监控的最大文件描述符值加 1；readfds：指向一个fd集合，监控读事件；writefds：指向一个fd集合，监控写事件。；exceptfds：指向一个fd集合，用于监控这些文件描述符的异常事件。；timeout：指定 select 函数的超时时间。

工作流程是：使用时初始fd_set集合，将需要监控的文件描述符加入，调用select函数。
原理是：监控时遍历所有监控的文件描述符，很慢；能监控的fd也有限，一般是1024个；每次调用都要传入这么多参数，有的fd
可能被反复传了很多次。

- epoll:

工作流程是：调用 epoll_create 创建一个 epoll 实例；使用 epoll_ctl 函数向 epoll 实例中增删改需要监控的文件描述符和事件;调用 epoll_wait 函数进行阻塞等待。当有事件发生或超时后，epoll_wait 函数返回，检查 events 数组来确定哪些文件描述符有事件发生。

原理是：红黑树+链表。红黑树实现快速增删改，链表存储有变动的事件调用 epoll_wait 函数时，内核会将链表复制到用户空间，通知用户哪些fd准备好了。
红黑树的增删改是O(logN)的，返回的链表也只包含了需要处理的事件，没有冗余信息。


### 边缘触发模式和水平触发模式
epoll有两种工作模式，边缘触发(Edge Triggered，ET)和水平触发(Level Triggered，LT)。
- ET：只有fd状态变化时(如从无数据变为有数据，从不可写变为可写)，才会触发事件通知。
用户必须一次性处理完所有的数据，否则剩余的数据将不会再次触发事件。
ET 模式是一种高速模式，适合处理大量的并发连接，但对编程要求较高。需要搭配非阻塞套接字使用。
如果用阻塞套接字，读到中途没数据读时，read函数会阻塞当前线程，直到有新数据到达。
但新数据到达前不会再次触发事件，就可能导致部分数据没法读，就这么丢了。

- LT：只要文件描述符的状态满足事件条件(如有数据可读，可写)，就会一直触发事件通知。即使一次没有处理完所有的数据，下次 epoll_wait 仍然会再次通知。
