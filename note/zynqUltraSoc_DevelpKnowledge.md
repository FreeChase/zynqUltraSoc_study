
- [前言](#前言)
- [一丶架构](#一丶架构)
      - [OCM(On Chip Memory)](#ocmon-chip-memory)
      - [cortex-A53 启动流程分析](#cortex-a53-启动流程分析)
- [二、windows+sdk](#二windowssdk)
  - [基础知识](#基础知识)
  - [启动文件详解](#启动文件详解)
    - [translation\_table.S](#translation_tables)
      - [MMU分为三级页表](#mmu分为三级页表)
  - [IDE\_DEBUG\_Register](#ide_debug_register)
- [三、petalinux](#三petalinux)
    - [基本概念](#基本概念)
      - [启动文件](#启动文件)
        - [1 BOOT.BIN](#1-bootbin)
        - [2 boot.scr](#2-bootscr)
        - [3 image.ub](#3-imageub)
      - [基本命令](#基本命令)
        - [1.petalinux-create](#1petalinux-create)
        - [2.petalinux-package](#2petalinux-package)
        - [3.petalinux-config](#3petalinux-config)
        - [4.petalinux-build](#4petalinux-build)
      - [配套工具安装](#配套工具安装)
        - [1.gdbserver](#1gdbserver)
        - [2.安装过程问题](#2安装过程问题)
      - [进程间通信](#进程间通信)
        - [1.有名管道](#1有名管道)
        - [2.共享内存](#2共享内存)
        - [3.消息队列](#3消息队列)
        - [4.信号量](#4信号量)
        - [5.socket\&文件](#5socket文件)
      - [中断机制](#中断机制)
        - [1.上半部](#1上半部)
        - [2.下半部](#2下半部)
      - [应用函数](#应用函数)
        - [1. mmap](#1-mmap)
        - [2.\_\_sync\_synchronize](#2__sync_synchronize)
    - [开发过中遇到的问题](#开发过中遇到的问题)
        - [1. nfs 挂载失败问题](#1-nfs-挂载失败问题)
        - [2.启动pl\_dma导致类似无法处理虚拟地址问题](#2启动pl_dma导致类似无法处理虚拟地址问题)
        - [3.考虑reserved-memory数据一致性问题](#3考虑reserved-memory数据一致性问题)
        - [4.编译器优化掉预留区域变量](#4编译器优化掉预留区域变量)
        - [5.串口设备ttyPS1乱码](#5串口设备ttyps1乱码)
        - [6.linux系统初始化完成后再执行脚本](#6linux系统初始化完成后再执行脚本)
        - [7.linux下串口设备号](#7linux下串口设备号)
        - [8.常量定义](#8常量定义)
        - [9.丢弃串口设备缓冲区数据](#9丢弃串口设备缓冲区数据)
        - [10.判断文件类型](#10判断文件类型)
        - [11.DSB、DMB、ISB、CACHE数据一致性](#11dsbdmbisbcache数据一致性)




# 前言


本文记录开发zynqUltra+MPSocs的知识点

---

`提示：以下是本篇文章正文内容，下面案例可供参考`

# 一丶架构

简介<br>
`Zynq UltraScale+ MPSoC` 是 AMD 的第二代 Zynq 平台，将强大的处理系统（PS）与用户可编程逻辑（PL）集成在同一器件中。
其处理系统采用 Arm® 主打的 Cortex®-A53 64 位四核或双核处理器，以及 Cortex-R5F 双核实时处理器。
其中Cortex-A53为ARM-v8A

#### OCM(On Chip Memory) 

这块 256 KB 的 RAM 阵列映射在高地址范围（0xFFFC_0000 到 0xFFFF_FFFF），
以四个独立的 64 KB 存储块为粒度进行划分。
每个存储块位于由 PMU（电源管理单元）控制的独立电源岛上。

| Address Range       | Size  | Memory Bank |
| ------------------- | ----- | ----------- |
| FFFC_0000-FFFC_FFFF | 64 KB | 0           |
| FFFD_0000-FFFD_FFFF | 64 KB | 1           |
| FFFE_0000-FFFE_FFFF | 64 KB | 2           |
| FFFF_0000-FFFF_FFFF | 64 KB | 3           |

#### cortex-A53 启动流程分析

```lua
+---------+       +-----------+       +-------------+       +-----------+
| boot.S  | ----> | B _startup| ----> | xil-crt0.S   | ----> | BL main   |
+---------+       +-----------+       +-------------+       +-----------+
                                                              |
                                                              v
                                                          +---------+
                                                          | main.c  |
                                                          +---------+


```


# 二、windows+sdk

## 基础知识

1. 通过xsa可以单独生成FSBL工程和APP工程
2. FSBL工程里面添加宏定义`FSBL_DEBUG_INFO`可以输出
    ```ini
    C/C++ Build -> Settings -> Symbols -> Defined symbols(-D) 增加工程宏定义
    C/C++ Build -> Settings -> Inferred Options -> Software platform 增加编译选项
    ```

## 启动文件详解

### translation_table.S

#### MMU分为三级页表

- 寻址地址由TCR_T0SZ确定,本工程为 2^(64-T0SZ(24)) = 2^40
    ```c
    /**********************************************
    * Set up TCR_EL1
    * Physical Address Size PS =  010 -> 44bits 16TB
    * Granual Size TG0 = 00 -> 4KB
    * size offset of the memory region T0SZ = 24 -> (region size 2^(64-24) = 2^40)
    ***************************************************/
    ldr     x1,=0x285800518
    msr     TCR_EL1, x1

    /*
    TCR[5:0]为T0SZ,配置为 0x18 =24
    */
    ```
- 这个translation_table.S文件，为**三级页表**设计，分别为MMUTableL0、MMUTableL1、MMUTableL2。
  - 每及Table最大支持`9bit`寻址
  -  MMUTableL0 **寻址unit**(`2^39bit=512GB`)
     -  /* 0x0000_0000 -  0x7F_FFFF_FFFF */
     -  /* 0x80_0000_0000 - 0xFF_FFFF_FFFF */
  -  MMUTableL1 **寻址unit**(`2^30bit=1GB`)

  -  MMUTableL2 **寻址unit**(`2^21bit=2MB`)

    故本设计页表基本单元大小为**2MB**

- 2

    ```arm
    .set Device,	0x409 | (1 << 53)| (1 << 54) |(0x0)	
    ```
    ```arm

    .rept	0x0200			/* 跟.endr搭配，重复代码0x200次*/
    .8byte	SECT + Device		/*声明一个8Byte空间，值为SECT + Device */
    .set	SECT, SECT+0x200000	/*SECT =  SECT+0x200000*/
    .endr

    # 0x200(512) x 0x200000(2MB) = 1 GB
    # 这个页表为Block,由Decive 确定, Device属性不带Cache，若要正常读写/执行程序，换成memory属性
    # Block大小为0x200000,由(.set SECT, SECT+0x200000) 确定
    # 
    ```

## IDE_DEBUG_Register
# 三、petalinux

### 基本概念

#### 启动文件

##### 1 BOOT.BIN
这是系统最先加载的引导文件
主要组成部分：

 - FSBL (First Stage Boot Loader)
 - PMU固件(如果有)
 - FPGA比特流文件(.bit)
 - U-Boot程序

生成方法：

使用Xilinx SDK/Vitis中的bootgen工具
通过.bif文件配置组成部分
 
 ##### 2 boot.scr

U-Boot的脚本文件,以文本形式编写后编译而成
包含U-Boot启动时要执行的命令序列
主要功能：

- 设置环境变量
- 配置启动参数
- 指定加载kernel的方式



生成方法：

```bash
# 先创建boot.cmd文本文件,然后转换为boot.scr
mkimage -A arm -O linux -T script -C none -a 0 -e 0 -n "Boot Script" -d boot.cmd boot.scr

```

 ##### 3 image.ub
 petalinux-build生成，由以下三个部分组成：
 

 - `kernel:` 一般就是linux生成的elf文件
 - `dtb:`设备树二进制文件，和驱动有关
 - `rootfs.img:`编译完成的文件系统

#### 基本命令

##### 1.petalinux-create
`petalinux-create` 命令主要用于创建新的 PetaLinux 组件。主要功能包括：

主要用途：

- 创建新项目
- 添加应用程序
- 创建 BSP 包


最常见的工作流程：

- 使用 `--type project` 创建新项目
- 使用 `--template` 指定目标架构
- 根据需要使用 `--type apps` 添加应用程序

```c
# 创建新的 PetaLinux 项目
petalinux-create --type project --name <项目名称> --template <模板类型>

# 可用的模板类型:
# - zynq          : 用于 Zynq-7000 SoC
# - zynqMP        : 用于 Zynq UltraScale+ MPSoC
# - microblaze    : 用于 MicroBlaze 处理器
# - versal        : 用于 Versal ACAP

# 示例: 创建一个 Zynq 项目
petalinux-create --type project --name my_zynq_project --template zynq

# 创建新的应用程序
petalinux-create --type apps --name <应用名称> --enable
# 示例:
petalinux-create --type apps --name my_application --enable

# 从现有项目创建 BSP
petalinux-create --type bsp --project <项目路径> --name <bsp名称> --hwsource <硬件源文件>

# 选项说明:
# --type          : 要创建的组件类型 (project/apps/bsp)
# --template      : 目标架构模板
# --name          : 组件名称
# --enable        : 在 rootfs 中启用应用程序(用于应用程序)
# --project       : 现有项目的路径(用于创建 BSP)
# --hwsource      : 硬件描述文件的路径
# --out          : 输出目录(可选)
# --force        : 强制创建(即使目录已存在)

# 注意: 创建项目前请确保:
# 1. PetaLinux 工具已安装
# 2. PetaLinux 环境已正确配置
# 3. 所需的硬件描述文件(.xsa/.hdf)可用


//从模板创建工程
petalinux-create -t project -n petalinux --template zynqMP
//工程打包到bsp
petalinux-package --bsp -p ./petalinux/ --output chapter1.bsp
//从bsp创建工程
petalinux-create -t project -n petalinux_bsp -s ./chapter1.bsp
```

##### 2.petalinux-package

`petalinux-package` 命令主要用于打包 PetaLinux 项目的各种组件。主要功能包括：

基本功能：

- 创建启动镜像(BOOT.BIN)
- 生成 BSP 文件
- 打包预编译镜像
- 创建 SDK
- 生成 WIC 镜像


最常见的使用场景：

- 生成可引导的 SD 卡镜像
- 创建完整的系统镜像
- 打包项目为 BSP 以便分享
- 准备开发环境的 SDK


使用流程：

先完成项目构建(petalinux-build)
根据需要选择打包选项
执行打包命令
验证生成的文件

```c
# 打包启动镜像(BOOT.BIN)
petalinux-package --boot --fsbl <fsbl镜像> --fpga <bit文件> --u-boot --force
# 示例:
petalinux-package --boot --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/system.bit --u-boot --force

# 创建 BSP 文件
petalinux-package --bsp --project <project路径> --output <输出文件名>.bsp
# 示例:
petalinux-package --bsp --project /path/to/project --output my_custom.bsp

# 打包预编译镜像
petalinux-package --prebuilt --fpga <bit文件> --u-boot --kernel --boot

# 创建 SDK
petalinux-package --sysroot

# 创建 WIC 镜像
petalinux-package --wic

# 主要选项说明:
# 启动镜像(--boot)相关选项:
# --fsbl          : 指定 FSBL (First Stage Boot Loader) 文件
# --fpga          : 指定比特流文件 (.bit)
# --u-boot        : 包含 U-Boot
# --pmufw         : 包含 PMU 固件 (仅 ZynqMP)
# --atf           : 包含 ARM Trusted Firmware (仅 ZynqMP)
# --force         : 强制覆盖已存在的文件

# BSP(--bsp)相关选项:
# --project       : 项目路径
# --output        : 输出的 BSP 文件名
# --hwsource      : 硬件平台规格文件路径

# 预编译(--prebuilt)相关选项:
# --fpga          : 包含比特流
# --u-boot        : 包含 U-Boot
# --kernel        : 包含内核
# --boot          : 生成启动镜像

# 注意事项:
# 1. 打包前确保已完成构建(petalinux-build)
# 2. BOOT.BIN 生成需要正确的 FSBL 和比特流文件
# 3. ZynqMP 设备可能需要额外的启动组件(ATF, PMUFW)
# 4. WIC 镜像需要正确配置存储分区

# 常见错误处理:
# - 如果出现文件缺失错误，检查构建是否完成
# - 权限错误时使用 sudo 或检查文件权限
# - 空间不足时清理项目目录或扩展存储空间
```

##### 3.petalinux-config


##### 4.petalinux-build

如果不是第一次编译设备树，即使修改了设备树执行`petalinux-build -c device-dree`也不会生成dtb文件，这时应先执行`petalinux-build -c device-tree -x cleansstate `清理编译状态后再编译设备树

1、将 system-user.dtsi复制到petalinux_project\components\plnx_workspace\device-tree\ 目录下(如果将system-user.dtsi放在原来目录，则修改无效，原因未知)

　　2、执行`petalinux-build -c device-tree -x cleansstate`清理设备树编译状态

　　3、执行`petalinux-build -c device-dree`编译设备树

一些常用命令

```c
petalinux-build -x clean   //清除缓冲

petalinux-build -x distclean //distclean 会删除更彻底的缓存，包括编译器和构建工具的缓存
```

#### 配套工具安装

##### 1.gdbserver
基本环境
| 参数1| 参数2|
|--|--|
| target board|  zynqUltra+Socs,4核a53|
|  gdb|  gdb8.2版本|
|  gcc|  gcc9.2版本|

编译过程

```c
//1.gdb 配置
tar -zxvf gdb-8.2.tar.gz
cd  gdb-8.2
mkdir _install
./configure \
  --build=x86_64-linux-gnu \
  --host=x86_64-linux-gnu \
  --target=aarch64-linux-gnu \
  --prefix=/home/alinx/work/tools/gdb-8.2/_install \
  CC=gcc \
  CXX=g++ \
  --with-python=no \
  --without-guile
make && make install

//2.gdbserver 配置
cd gdb-7.11.1 /gdb/gdbserver
./configure --host=aarch64-linux-gnu
make

3.将 gdb/gdbserver/下的gdbserver执行文件拷贝到根文件系统中的bin目录
同时记得将aarch64-linux-gnu-编译工具下的/lib&/usr/lib拷贝至根文件系统
(此处应为构建根文件系统就搭好)
```

##### 2.安装过程问题
问题1
```c
aarch64-xilinx-linux-ld.real: ../gdbsupport/libgdbsupport.a(agent.o): Relocations in generic ELF (EM: 62)
```
解决方案：
此版本下载的`gdb10.1`,petalinux版本`gcc9.2`，后将版本降为`gdb8.2`

问题2 configure指定参数问题

```c
./configure --target=aarch64-linux-gnu --prefix=/home/alinx/work/tools/gdb-8.2/_install
//提示配置异常
```
解决方案：
增加参数，指定build、host参数
```c
./configure \
  --build=x86_64-linux-gnu \
  --host=x86_64-linux-gnu \
  --target=aarch64-linux-gnu \
  --prefix=/home/alinx/work/tools/gdb-8.2/_install \
  CC=gcc \
  CXX=g++ \
  --with-python=no \
  --without-guile
```

#### 进程间通信

##### 1.有名管道

##### 2.共享内存
共享内存（`Shared Memory`）是进程间通信（IPC）的一种方式，允许多个进程访问同一块内存区域。它通过在内存中创建一个共享的区域，使得数据可以直接通过读写内存来交换，而不需要进行拷贝操作。这种方式效率很高，适合需要频繁和大量数据交换的场景。
**共享内存的关键特性**
- 高效性：

	共享内存是所有 IPC 机制中效率最高的，因为它直接使用内存，不需要内核参与**数据的搬运**。
- 持久性：

	共享内存段在显式销毁之前一直存在，即使没有进程使用它，直到调用 shmctl 删除为止。
- 同步问题：

	共享内存本身**不**提供**同步机制**。如果多个进程同时读写，需要额外使用信号量或其他同步机制避免冲突。
- 实现方式
	- System V 共享内存(shmget)
	
		主要特点：

		- 使用 key 标识共享内存段
		- 需要显式管理共享内存的创建和删除
		- 系统范围内可见
		- 有最大尺寸限制
		- 权限控制相对简单
	
  - POSIX 共享内存(shm_open)  **建议使用**
  	主要特点：
	- 使用文件系统路径命名
	- 可以像文件一样操作
	- 支持更细粒度的权限控制
	- 接口更现代，与其他 POSIX API 一致
	- 可以使用 mmap 的相关特性
	
```c
// 创建共享内存对象
int shm_open(const char *name, int oflag, mode_t mode);
// 设置大小
int ftruncate(int fd, off_t length);
// 映射到进程地址空间
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
// 解除映射
int munmap(void *addr, size_t length);
// 关闭和删除
int close(int fd);
int shm_unlink(const char *name);
```


  
##### 3.消息队列

##### 4.信号量

##### 5.socket&文件



#### 中断机制
##### 1.上半部
上半部(`Top Half`):

是中断处理的第一阶段,直接响应硬件中断
运行在中断上下文中,不可被中断
执行时间必须很短
(**ps**:感觉是为了快速处理+不漏中断，设计出上半部和下半部的概念)
主要完成以下工作:

- 保存中断现场
- 标记中断已处理
- 获取必要的硬件状态
- 唤醒下半部执行
- 恢复中断现场

**实现方式：**
```c
// 编程接口，硬中断
// 注册中断处理函数
request_irq(irq_num, handler, flags, name, dev);

// 中断处理函数原型
irqreturn_t interrupt_handler(int irq, void *dev_id)
```

##### 2.下半部
下半部(`Bottom Half`):

是中断处理的第二阶段,可以被其他中断打断
在进程上下文中运行
可以执行较长时间的任务
主要完成以下工作:

- 处理上半部保存的数据
- 执行耗时的数据处理
- 触发其他内核机制

举个例子:
当网卡收到数据包时:

- 上半部快速保存数据包到缓冲区
- 下半部再进行协议解析等耗时操作

这种分层设计的优点是:

- 提高系统响应性 - 上半部快速执行完毕
- 降低中断延迟 - 耗时操作放在可中断的下半部
- 更好的并发处理 - 下半部可以被调度

**实现方式：**
主要有三种机制：

- 软中断（`softirq`）：

	最底层的下半部机制
优先级固定
多个CPU可以同时执行

```c
// 定义软中断
struct softirq_action {
    void (*action)(struct softirq_action *);
};

// 注册软中断
open_softirq(NET_TX_SOFTIRQ, net_tx_action);
```



- `tasklet`：

	基于软中断实现
动态创建
同一类型的tasklet串行执行

```c
// 定义tasklet
DECLARE_TASKLET(name, func, data);

// 调度tasklet执行
tasklet_schedule(&my_tasklet);
```



- 工作队列（`workqueue`）：

	运行在进程上下文
可以睡眠
适合耗时较长的任务

```c
// 创建工作
DECLARE_WORK(name, func);

// 调度工作执行
schedule_work(&my_work);
```


#### 应用函数
##### 1. mmap

```c
// 设备文件，代表整个物理内存
int fd = open("/dev/mem", O_RDWR);

// 通过mmap将物理内存映射到用户空间，
void *virt_addr = mmap(NULL,                  // 建议的映射地址，NULL让系统自动选择
                      length,                  // 映射长度
                      PROT_READ | PROT_WRITE, // 读写权限
                      MAP_SHARED,             // 共享映射
                      fd,                     // /dev/mem的文件描述符
                      phys_addr);             // 物理地址
                      
物理地址(fd + phys_addr)映射到虚拟地址
```

##### 2.__sync_synchronize

`__sync_synchronize `是 GCC 提供的一个内置函数，用于实现内存屏障（memory barrier）。它是一个 全局的内存同步操作，确保当前线程对共享内存的所有读写操作在屏障前已经完成，并且这些操作的结果对其他线程是可见的。


- 防止编译器优化：

	编译器可能会对代码重新排序，以优化性能，但这可能打乱多线程环境下的预期行为。__sync_synchronize 可以禁止编译器重新排序操作。
- 防止 CPU 指令乱序：

	现代 CPU 为了提高性能，会对指令进行乱序执行（out-of-order execution）。__sync_synchronize 确保内存操作的顺序严格按照代码逻辑顺序执行。
- 多线程同步：

	在多线程编程中，确保不同线程之间对共享变量的访问是按照某种特定的顺序执行。

### 开发过中遇到的问题
##### 1. nfs 挂载失败问题
命令
`mount -t nfs 192.168.0.110:/home/alinx/work /mnt/nfsdir`
日志
```bash
//LOG
mount -t nfs 192.168.0.110:/home/alinx/work /mnt/nfsdir
mount: /mnt/nfsdir: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```
解决方案
`mount -t nfs 192.168.0.110:/home/alinx/work /mnt/nfsdir -o rw,nolock,addr=192.168.0.110`
在原命令基础上增加参数` -o rw,nolock,addr=192.168.0.110`


##### 2.启动pl_dma导致类似无法处理虚拟地址问题
日志
```c
`Unable to handle kernel paging request at virtual address ffffff8000f91000 [ 656.654134]`
```
解决方案
在设备树中写入了对plmemory地址的管理(0x8000 0000 ~0xc000 0000)
取消掉就正常了

##### 3.考虑reserved-memory数据一致性问题
情景

```c
设备树代码
memory {
        device_type = "memory";
        reg = <0x00 0x00 0x00 0x60000000>;
    };

    reserved-memory {
        #address-cells = <0x02>;
        #size-cells = <0x02>;
        ranges;

        reserved {
            reg = <0x00 0x60000000 0x00 0x1f000000>;
            no-map;
        };
    };
    
硬件架构
CPU <--> Cache <--> DDR
DMA <-----------> DDR
- DMA控制器直接通过系统总线访问DDR
- DMA没有cache层,不能访问CPU的cache
- 这就是为什么需要cache一致性维护的根本原因
```


解决方案
方案1：使用内存屏障

```c
// CPU写入后确保数据写回内存
wmb();  // 写内存屏障
dsb();  // ARM架构的数据同步屏障

// CPU读取前确保读到最新数据
rmb();  // 读内存屏障
```
方案2：手动管理cache

```c
CPU写->DMA读的场景
// CPU写入后刷新cache
//调用dma_sync_single_for_device: Cache中的脏数据写回DDR
//DMA读数据: 直接从DDR读取(此时DDR数据是最新的)
dma_sync_single_for_device(dev, addr, size, DMA_TO_DEVICE);
```
```c
DMA写数据: DMA -> DDR(写入新数据)
//调用dma_sync_single_for_cpu: 使Cache中对应的行失效
//CPU读数据: 因为Cache已失效,会从DDR重新加载最新数据到Cache
// DMA操作完成后使cache失效
dma_sync_single_for_cpu(dev, addr, size, DMA_FROM_DEVICE);
```
具体代码
```c
void dma_sync_single_for_device(struct device *dev, dma_addr_t addr,
                               size_t size, enum dma_data_direction dir)
{
    // 对于ARM架构，最终会调用到：
    __dma_map_area(phys_to_virt(addr), size, DMA_TO_DEVICE);
}

static inline void __dma_map_area(const void *addr, size_t size,
                                 enum dma_data_direction dir)
{
    if (dir == DMA_TO_DEVICE) {
        // 1. Clean - 将cache中的脏数据写回内存
        dmac_clean_range(addr, addr + size);
        // 或者使用
        __clean_dcache_area(addr, size);
        
        // 2. 确保清理完成
        dsb(ishst);
    }
}

void dma_sync_single_for_cpu(struct device *dev, dma_addr_t addr,
                            size_t size, enum dma_data_direction dir)
{
    // 对于ARM架构，最终会调用到：
    __dma_map_area(phys_to_virt(addr), size, DMA_FROM_DEVICE);
}

static inline void __dma_map_area(const void *addr, size_t size,
                                 enum dma_data_direction dir)
{
    if (dir == DMA_FROM_DEVICE) {
        // 1. Invalidate - 使cache line失效
        dmac_inv_range(addr, addr + size);
        // 或者使用
        __inval_dcache_area(addr, size);
        
        // 2. 确保失效操作完成
        dsb(ishst);
    }
}

// Clean操作的底层实现（ARM64为例）
void __clean_dcache_area(void *addr, size_t size)
{
    uint64_t line, end;
    
    // 计算需要clean的cache line范围
    line = (uint64_t)addr & ~(CACHE_LINE_SIZE - 1);
    end = (uint64_t)(addr + size - 1) & ~(CACHE_LINE_SIZE - 1);
    
    // 逐个clean cache line
    while (line <= end) {
        // 使用dc cvac指令clean cache line到point of coherency
        asm volatile("dc cvac, %0" : : "r" (line));
        line += CACHE_LINE_SIZE;
    }
}

// Invalidate操作的底层实现
void __inval_dcache_area(void *addr, size_t size)
{
    uint64_t line, end;
    
    // 计算需要invalidate的cache line范围
    line = (uint64_t)addr & ~(CACHE_LINE_SIZE - 1);
    end = (uint64_t)(addr + size - 1) & ~(CACHE_LINE_SIZE - 1);
    
    // 逐个invalidate cache line
    while (line <= end) {
        // 使用dc ivac指令invalidate cache line
        asm volatile("dc ivac, %0" : : "r" (line));
        line += CACHE_LINE_SIZE;
    }
}
```
内联汇编，无效指定缓存行(`cache line`)
```c

DC IVAC 是一条 数据缓存失效 指令，它通过指定一个内存地址，来使该地址所在的缓存行失效。
缓存行（cache line）通常是 64 字节，包含多个连续的内存地址。

#include <stdio.h>

int main() {
    int x = 42;  // 定义一个变量
    uintptr_t address = (uintptr_t)&x;  // 获取变量地址

    // ARM 汇编中使用 DC IVAC
    __asm__ volatile(
        "MOV x0, %0\n"    // 将 address 加载到寄存器 x0
        "DC IVAC, x0\n"   // 使缓存行失效
        :
        : "r"(address)    // 将变量地址传递给汇编
        : "x0"            // 声明汇编会使用 x0
    );

    return 0;
}

```

方案3: 将该区域设置为non-cacheable
可以通过修改设备树添加 "memory-region-no-cache" 属性，或通过MMU页表设置来实现


##### 4.编译器优化掉预留区域变量
情景
定义了全局变量，并设置在指定section，预留给DMA或者其他IP使用，被编译器优化掉
解决方案
在link.lds使用KEEP关键词

```c
SECTIONS {
    .my_section : {
        KEEP(*(.my_section))  /* 强制保留 .my_section 段 */
    } >RAM
}

int my_variable __attribute__((section(".my_section"))) = 42;

```

##### 5.串口设备ttyPS1乱码

情景：
在windows使用串口调试助手，接收串口数据，出现乱码
解决方案：

 - 检查波特率

```c
//查看设备信息命令
stty -F /dev/ttyPS1	
//日志输出如下
stty -F /dev/ttyPS1
speed 9600 baud; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>; eol2 = <undef>; swtch = <undef>;
start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R; werase = ^W; lnext = ^V; flush = ^O; min = 1; time = 0;
-brkint -imaxbel

```

 - 检查物理连线

##### 6.linux系统初始化完成后再执行脚本
情景：
想在系统完全初始化后，再执行脚本，包括应用函数、IP配置等

解决方案：
在 `/etc/ini.d/rc`中末尾增加命令，如 `source /home/root/user.h`
在`/etc/ini.d/rcS`末尾添加的命令的话，系统还未初始化完全


##### 7.linux下串口设备号


在 Linux 中，串口一般在操作系统的`/dev/`，并以 `tty* `开头。 常见名称有：
- `/dev/ttyACM0 `表示 USB 总线上的 ACM 调制解调器。Arduino UNO 一般是这个名字。
- `/dev/ttyPS0 `使用 Yocto 移植 Linux 版本的 Xilinx Zynq FPGA 一般使用此名称作为 Getty 连接的默认串行端口。
- `/dev/ttyS0`标准 COM 端口将使用此名称。如今，这些端口已不常见。
- `/dev/ttyUSB0`大多数 USB 转串行线将使用类似这样的文件显示。
- `/dev/pts/0`伪终端。


##### 8.常量定义
|  |  |
|--|--|
|无后缀	 |int（通常为 32 位） |
 |U	 |unsigned int（无符号 32 位） |
 |L	 |long（通常为 32 位或 64 位，视平台而定） |
 |UL	 |unsigned long（无符号长整型） |
 |LL	 |long long（通常为 64 位） |
 |ULL	 |unsigned long long（无符号 64 位） |


##### 9.丢弃串口设备缓冲区数据

情景：
使用`FILE* fopen`打开的文件，而不是`open`
而函数 `int tcflush(int fd, int queue_selector)` 使用的是int型的文件描述符
解决方案：
使用`int fd = fileno(fp)`从文件`FILE *fp`中获取fd
```c
#include <stdio.h>
#include <fcntl.h>
#include <termios.h>
#include <unistd.h>

int main() {
    const char *serial_port = "/dev/ttyS0"; // 替换为实际串口设备路径

    // 使用 fopen 打开串口设备
    FILE *fp = fopen(serial_port, "r+");
    if (!fp) {
        perror("Failed to open serial port");
        return 1;
    }

    // 从 FILE * 中获取文件描述符
    int fd = fileno(fp);
    if (fd < 0) {
        perror("Failed to get file descriptor");
        fclose(fp);
        return 1;
    }

    // 清空输入缓冲区
    if (tcflush(fd, TCIFLUSH) < 0) {
        perror("Failed to flush input buffer");
    } else {
        printf("Input buffer flushed successfully.\n");
    }

    // 清空输出缓冲区
    if (tcflush(fd, TCOFLUSH) < 0) {
        perror("Failed to flush output buffer");
    } else {
        printf("Output buffer flushed successfully.\n");
    }

    // 同时清空输入和输出缓冲区
    if (tcflush(fd, TCIOFLUSH) < 0) {
        perror("Failed to flush both input and output buffers");
    } else {
        printf("Both input and output buffers flushed successfully.\n");
    }

    fclose(fp);
    return 0;
}

```

##### 10.判断文件类型

```c
S_ISLNK(st_mode)：是否是一个连接.
S_ISREG(st_mode) ：是否是一个常规文件.
S_ISDIR(st_mode) ：是否是一个目录
S_ISCHR(st_mode)：是否是一个字符设备.
S_ISBLK(st_mode)：是否是一个块设备
S_ISFIFO(st_mode)：是否 是一个FIFO文件.
S_ISSOCK(st_mode)：是否是一个SOCKET文件 

void list_files(const char *path) {
    struct dirent *entry;
    struct stat file_stat;
    char filepath[1024];
    T_Sample_Param sampleCfg;
    DIR *dir = opendir(path);

    if (!dir) {
        perror("opendir");
        return;
    }

    while ((entry = readdir(dir)) != NULL) {
        // 忽略 "." 和 ".." 目录
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
            continue;

        // 构建文件完整路径
        snprintf(filepath, sizeof(filepath), "%s/%s", path, entry->d_name);

        // 获取文件状态信息
        if (stat(filepath, &file_stat) == -1) {
            perror("stat");
            continue;
        }

        // 判断是否为文件
        if (S_ISREG(file_stat.st_mode)) {
            printf("File: %s, Size: %ld bytes\n", entry->d_name, file_stat.st_size);
        }
    }

    closedir(dir);
}
```

##### 11.DSB、DMB、ISB、CACHE数据一致性

常规代码(默认数据为cacheable)
```c
// CPU0 写数据
void write_shared_data(int new_value) {
    shared_data = new_value;     // 写入数据，到cacheable区域
    DMB();                       // 确保数据写入完成
    data_ready = 1;             // 设置标志
    DSB();                      // 确保标志写入完成并对其他核心可见
}

// CPU1 读数据
int read_shared_data(void) {
    while (data_ready != 1) {
        DMB();                  // 确保每次都读取最新的标志值
    }
    ISB();                      // 刷新流水线
    DMB();                      // 确保读取最新数据
    return shared_data;
}
```
**代码补充说明：**
让我详细解释 CPU0 写入数据后的状态和 CPU1 观察数据的过程：
在 CPU0 执行完这段代码后：

**shared_data = new_value;** //假设`shared_data`为**cacheable内存区域**


- 数据首先写入 `CPU0` 的 `L1 cache`(写策略是**Write-Back**则写入cache，写策略是**Write-Through**则同步写入**RAM**和**cache**)
- 缓存一致性协议会将这个` cache line`标记为 `Modified `状态


**DMB():**


- 确保 `data` 的写入动作在`CPU0`中完成(根据**写策略**决定数据写入位置)

- 其他 CPU 此时可能还看不到更新


**flag = 1;**


- 同步标志也是先写入 CPU0 的 cache
- 缓存一致性协议会将这个 `cache line` 标记为 `Modified`


**DSB();**


- 确保所有缓存操作完成
- 数据依然可能只存在于 cache中，不一定写回 DDR

当 CPU1 要读取数据时：

- 通过缓存一致性协议(如 `MESI`)，`CPU1` 会直接从 `CPU0` 的 `cache `中获取数据
- 不需要等待数据写回 DDR(**写策略**为**Write-Back**时)
- 这个过程是由硬件自动完成的

**简单来说**：

数据更新后主要存在于` cache `中
写回 DDR 的时机由缓存策略决定（如 `write-back` 或 `write-through`）
CPU1 读取时是通过缓存一致性协议直接从其他` CPU `的` cache` 获取，而不是从 DDR 读取
`DMB/DSB `确保内存操作顺序，但不会强制写回 `DDR`

| 特性 |	DMB(Data Memory Barrier)  |  DSB(Data Synchronation Barrier) |  ISB(Instruction Synchronation Barrier) | 
|--|--|--|--|
| 类型 | 数据内存屏障 | 数据同步屏障  |  指令同步屏障 | 
| 主要用途 | 确保内存访问顺序 | 比DMB严格,确保DSB之前的所有指令（不仅是存储器访问，还包括cache维护、TLB维护等）都完成  | 确保指令流水线同步  | 
| 影响范围 | 数据内存访问 | 所有指令和内存访问  | 仅指令流水线  | 
| 典型场景 | 多核内存同步 |缓存失效、TLB 刷新 | 修改处理器状态寄存器  | 

**应用建议：**
- DMB：用于数据访问的内存同步，适合在共享内存或多线程环境下。
- DSB：用于确保所有系统级操作完成，常见于系统初始化和设备交互。
- ISB：用于确保指令序列重新获取，适合状态切换或上下文切换的场景。


**补充2 write-back或 write-through**

`Write-through`（写通策略）：

- **工作原理**：

	- 当CPU写数据时，同时写入cache和主存(DDR)
	- **每次写操作都会立即更新主存**
	- **cache和主存的数据始终保持一致**


- 优点：


	- 数据一致性好，主存总是包含最新数据
	- 系统掉电时数据不会丢失
	- 缓存一致性协议实现相对简单


- 缺点：
	- 写操作延迟较高（需要等待写入主存）
	- 占用更多内存带宽
	- 功耗较高（频繁访问主存）

`Write-back`（写回策略）：

- **工作原理**：
CPU写数据时只写入cache
将cache line标记为"脏"(`dirty`)状态

	只在**必要**时才写回主存：
	- cache line被替换时
	-  	其他CPU请求该数据时
	- 显式的cache flush操作
	- 同步指令（如DSB）要求时

- 优点：
	- 写操作延迟低（不用等待主存）
	- 减少内存带宽使用
	- 多次写入同一地址只需最后写回一次
	- 功耗较低

- 缺点：
	- 掉电可能丢失数据（dirty数据未写回）
	- cache一致性协议实现较复杂
	- 需要额外的dirty标志位

**补充3 DSB ISB区分**

DSB 和 ISB 关注的"数据"是不同的：

DSB 关注的"数据"：


内存中的数据（Memory data）
缓存中的数据（Cache data）
外设寄存器的数据（Peripheral register data）
这些数据的特点是：它们是程序操作的对象，是存储在内存系统中的内容


ISB 关注的"系统状态"：


系统寄存器的值（比如 SCTLR_EL1, TCR_EL1 等）
MMU 配置
特权级别
这些更像是处理器的"配置"或"状态"，决定了处理器如何执行指令

```c
场景1：修改内存数据

STR  X1, [X0]    // 写入内存数据
DSB  ISH         // 需要DSB确保数据写入完成

场景2：修改系统配置

MSR  SCTLR_EL1, X0  // 修改系统控制寄存器
ISB                 // 需要ISB确保新的系统配置生效
```

---


