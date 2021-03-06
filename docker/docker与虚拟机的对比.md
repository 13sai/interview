Form: [Docker与虚拟机性能比较](http://www.linuxidc.com/Linux/2016-05/130991.htm)


### 概要

Docker是近年来新兴的虚拟化工具，它可以和虚拟机一样实现资源和系统环境的隔离。本文将主要根据IBM发表的研究报告，论述Docker与传统虚拟化方式的不同之处，并比较物理机、Docker容器、虚拟机三者的性能差异及差异产生的原理。

### Docker与虚拟机实现原理比较

如下图分别是虚拟机与Docker的实现框架。
![虚拟机实现框架](http://www.linuxidc.com/upload/2016_05/160504202511241.png) ![docker实现框架](http://www.linuxidc.com/upload/2016_05/160504202511242.png)
比较两图的差异，左图虚拟机的Guest OS层和Hypervisor层在Docker中被Docker Engine层所替代。虚拟机的Guest OS即为虚拟机安装的操作系统，它是一个完整操作系统内核；虚拟机的Hypervisor层可以简单理解为一个硬件虚拟化平台，它在Host OS是以内核态的驱动存在的。
虚拟机实现资源隔离的方法是利用独立的OS，并利用Hypervisor虚拟化CPU、内存、IO设备等实现的。例如，为了虚拟CPU，Hypervisor会为每个虚拟的CPU创建一个数据结构，模拟CPU的全部寄存器的值，在适当的时候跟踪并修改这些值。需要指出的是在大多数情况下，虚拟机软件代码是直接跑在硬件上的，而不需要Hypervisor介入。只有在一些权限高的请求下，Guest OS需要运行内核态修改CPU的寄存器数据，Hypervisor会介入，修改并维护虚拟的CPU状态。
Hypervisor虚拟化内存的方法是创建一个shadow page table。正常的情况下，一个page table可以用来实现从虚拟内存到物理内存的翻译。在虚拟化的情况下，由于所谓的物理内存仍然是虚拟的，因此shadow page table就要做到：虚拟内存->虚拟的物理内存->真正的物理内存。
对于IO设备虚拟化，当Hypervisor接到page fault，并发现实际上虚拟的物理内存地址对应的是一个I/O设备，Hypervisor就用软件模拟这个设备的工作情况，并返回。比如当CPU想要写磁盘时，Hypervisor就把相应的数据写到一个host OS的文件上，这个文件实际上就模拟了虚拟的磁盘。
对比虚拟机实现资源和环境隔离的方案，Docker就显得简练很多。Docker Engine可以简单看成对Linux的NameSpace、Cgroup、镜像管理文件系统操作的封装。Docker并没有和虚拟机一样利用一个完全独立的Guest OS实现环境隔离，它利用的是目前Linux内核本身支持的容器方式实现资源和环境隔离。简单的说，Docker利用namespace实现系统环境的隔离；利用Cgroup实现资源限制；利用镜像实现根目录环境的隔离。

通过Docker和虚拟机实现原理的比较，我们大致可以得出一些结论：

1. Docker有着比虚拟机更少的抽象层。由于Docker不需要Hypervisor实现硬件资源虚拟化，运行在Docker容器上的程序直接使用的都是实际物理机的硬件资源。因此在CPU、内存利用率上Docker将会在效率上有优势，具体的效率对比在下几个小节里给出。在IO设备虚拟化上，Docker的镜像管理有多种方案，比如利用Aufs文件系统或者Device Mapper实现Docker的文件管理，各种实现方案的效率略有不同。
2. Docker利用的是宿主机的内核，而不需要Guest OS。因此，当新建一个容器时，Docker不需要和虚拟机一样重新加载一个操作系统内核。我们知道，引导、加载操作系统内核是一个比较费时费资源的过程，当新建一个虚拟机时，虚拟机软件需要加载Guest OS，这个新建过程是分钟级别的。而Docker由于直接利用宿主机的操作系统，则省略了这个过程，因此新建一个Docker容器只需要几秒钟。另外，现代操作系统是复杂的系统，在一台物理机上新增加一个操作系统的资源开销是比较大的，因此，Docker对比虚拟机在资源消耗上也占有比较大的优势。事实上，在一台物理机上我们可以很容易建立成百上千的容器，而只能建立几个虚拟机。

### Docker与虚拟机计算效率比较

在上一节我们从原理的角度推测Docker应当在CPU和内存的利用效率上比虚拟机高。在这一节我们将根据IBM发表的论文给出的数据进行分析。以下的数据均是在IBM x3650 M4服务器测得，其主要的硬件参数是：

- 2颗英特尔xeon E5-2655 处理器，主频2.4-3.0 GHz。每颗处理器有8个核，因此总共有16个核

- 256 GB RAM

  

在测试中是通过运算Linpack程序来获得计算能力数据的。结果如下图所示：
![此处输入图片的描述](http://www.linuxidc.com/upload/2016_05/160504202511243.png)
图中从左往右分别是物理机、Docker和虚拟机的计算能力数据。可见Docker相对于物理机其计算能力几乎没有损耗，而虚拟机对比物理机则有着非常明显的损耗。虚拟机的计算能力损耗在50%左右。

为什么会有这么大的性能损耗呢？一方面是因为虚拟机增加了一层虚拟硬件层，运行在虚拟机上的应用程序在进行数值计算时是运行在Hypervisor虚拟的CPU上的；另外一方面是由于计算程序本身的特性导致的差异。虚拟机虚拟的cpu架构不同于实际cpu架构，数值计算程序一般针对特定的cpu架构有一定的优化措施，虚拟化使这些措施作废，甚至起到反效果。比如对于本次实验的平台，实际的CPU架构是2块物理CPU，每块CPU拥有16个核，共32个核，采用的是NUMA架构；而虚拟机则将CPU虚拟化成一块拥有32个核的CPU。这就导致了计算程序在进行计算时无法根据实际的CPU架构进行优化，大大减低了计算效率。

### Docker与虚拟机内存访问效率比较

内存访问效率的比较相对比较复杂一点，主要是内存访问有多种场景：

- 大批量的，连续地址块的内存数据读写。这种测试环境下得到的性能数据是内存带宽，性能瓶颈主要在内存芯片的性能上；

- 随机内存访问性能。这种测试环境下的性能数据主要与内存带宽、cache的命中率和虚拟地址与物理地址转换的效率等因素有关。

以下将主要针对这两种内存访问场景进行分析。在分析之前我们先概要说明一下Docker和虚拟机的内存访问模型差异。下图是Docker与虚拟机内存访问模型：
![此处输入图片的描述](http://www.linuxidc.com/upload/2016_05/160504202511247.png)
可见在应用程序内存访问上，虚拟机的应用程序要进行2次的虚拟内存到物理内存的映射，读写内存的代价比Docker的应用程序高。
下图是场景（1）的测试数据，即内存带宽数据。左图是程序运行在一块CPU（即8核）上的数据，右图是程序运行在2块CPU（即16核）上的数据。单位均为GB/s。
![此处输入图片的描述](http://www.linuxidc.com/upload/2016_05/160504202511244.png) ![此处输入图片的描述](http://www.linuxidc.com/upload/2016_05/160504202511246.png)
从图中数据可以看出，在内存带宽性能上Docker与虚拟机的性能差异并不大。这是因为在内存带宽测试中，读写的内存地址是连续的，大批量的，内核对这种操作会进行优化（数据预存取）。因此虚拟内存到物理内存的映射次数比较少，性能瓶颈主要在物理内存的读写速度上，因此这种情况Docker和虚拟机的测试性能差别不大;

内存带宽测试中Docker与虚拟机内存访问性能差异不大的原因是由于内存带宽测试中需要进行虚拟地址到物理地址的映射次数比较少。根据这个假设，我们推测，当进行随机内存访问测试时这两者的性能差距将会变大，因为随机内存访问测试中需要进行虚拟内存地址到物理内存地址的映射次数将会变多。结果如下图所示。
  ![此处输入图片的描述](http://www.linuxidc.com/upload/2016_05/160504202511245.png)![此处输入图片的描述](http://www.linuxidc.com/upload/2016_05/160504202511248.png)

左图是程序运行在一个CPU上的数据，右图是程序运行在2块CPU上的数据。从左图可以看出，确实如我们所预测的，在随机内存访问性能上容器与虚拟机的性能差距变得比较明显，容器的内存访问性能明显比虚拟机优秀；但出乎我们意料的是在2块CPU上运行测试程序时容器与虚拟机的随机内存访问性能的差距却又变的不明显。

针对这个现象，IBM的论文给出了一个合理解释。这是因为当有2块CPU同时对内存进行访问时，内存读写的控制将会变得比较复杂，因为两块CPU可能同时读写同一个地址的数据，需要对内存数据进行一些同步操作，从而导致内存读写性能的损耗。这种损耗即使对于物理机也是存在的，可以看出右图的内存访问性能数据是低于左图的。2块CPU对内存读写性能的损耗影响是非常大的，这个损耗占据的比例远大于虚拟机和Docker由于内存访问模型的不同产生的差异，因此在右图中Docker与虚拟机的随机内存访问性能上我们看不出明显差异。

### Docker与虚拟机启动时间及资源耗费比较

上面两个小节主要从运行在Docker里的程序和运行在虚拟机里的程序进行性能比较。事实上，Docker之所以如此受到开发者关注的另外一个重要原因是启动Docker的系统代价比启动一台虚拟机的代价要低得多：无论从启动时间还是从启动资源耗费角度来说。Docker直接利用宿主机的系统内核，避免了虚拟机启动时所需的系统引导时间和操作系统运行的资源消耗。利用Docker能在几秒钟之内启动大量的容器，这是虚拟机无法办到的。快速启动、低系统资源消耗的优点使Docker在弹性云平台和自动运维系统方面有着很好的应用前景。

### Docker的劣势

前面的内容主要论述Docker相对于虚拟机的优势，但Docker也不是完美的系统。相对于虚拟机，Docker还存在着以下几个缺点：

1. 资源隔离方面不如虚拟机，Docker是利用cgroup实现资源限制的，只能限制资源消耗的最大值，而不能隔绝其他程序占用自己的资源。
2. 安全性问题。Docker目前并不能分辨具体执行指令的用户，只要一个用户拥有执行Docker的权限，那么他就可以对Docker的容器进行所有操作，不管该容器是否是由该用户创建。比如A和B都拥有执行Docker的权限，由于Docker的server端并不会具体判断Docker cline是由哪个用户发起的，A可以删除B创建的容器，存在一定的安全风险。
3. Docker目前还在版本的快速更新中，细节功能调整比较大。一些核心模块依赖于高版本内核，存在版本兼容问题