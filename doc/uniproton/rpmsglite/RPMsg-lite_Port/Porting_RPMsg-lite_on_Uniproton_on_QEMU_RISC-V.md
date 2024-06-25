# RPMsg-lite移植到Uniproton on QEMU RISC-V

- 1.[概述](#概述)
- 2.[RPMsg-lite environment层移植](#environment-layer移植)
  - 2.1[env_sleep_msec接口适配](#env_sleep_msec)
  - 2.2[RPMsg-lite queue接口适配](#rpmsg-lite-queue相关接口)
- 3.[RPMsg-lite platform层移植](#environment-layer移植)
  - 3.1[RPMsg-lite中断发送实现](#rpmsg-lite中断发送接口)
  - 3.2[RPMsg-lite中断回调实现](#rpmsg-lite中断回调接口)
  - 3.3[Uniproton RISC-V简单IPI module实现](#uniproton-risc-v-ipi-module)
    - 3.3.1[中断向量实现](#中断向量)
    - 3.3.2[中断发送实现](#中断发送)
    - 3.3.3[中断mailbox实现](#中断数据传递简单mailbox)
## 概述
NXP在RPMsg-lite（见RPMsg-lite的github仓库）中介绍了移植RPMsg-lite到特定环境的思路。大致分为两个方面：**environment layer和platform layer**的移植。（**[具体的移植代码在Uniproton的gitee仓库源码根目录下src/component/rpmsglite](https://gitee.com/openeuler/UniProton/tree/master)**）
- environment layer基本可以理解为OS layer（裸机（baremental）则可能直接的板级SDK的支持），因为有OS层的抽象，所以在不同的硬件环境上相对比较稳定。主要需要基于OS/baremental提供的接口实现RPMsg-lite中以下接口：
  - 内存管理/字符串操作/打印接口
  - 互斥锁接口
  - （非必须）FIFO队列接口（如果想启用RPMsg-lite queue特性则必须实现）
  - 任务休眠接口等
- platform layer则是直接对接硬件环境。因为和硬件环境耦合，所以platform相比environment变动更多，任何两个不同的硬件环境上platform的配置可能都不一样。platform layer主要关注外部中断寄存器的地址（因为要通过IPI进行消息的通知），并且根据硬件特性实现以下接口：
  - 硬件中断失活/使能接口
  - 发送RPMsg-lite IPI的接口
  - RPMsg-lite IPI的中断回调接口
  - 当前核v2p/p2v的地址转换函数（主要在启用了MMU的环境要注意）
  - （非必须）自定义shmem配置的获取接口（如果启用了自定义shmem配置则必须实现）
## environment layer移植
RPMsg-lite在rpmsg_env.h中声明了env layer实现的核心函数，可以参考RPMsg-lite仓库中其它OS的实现。
- Uniproton移植了libc，因此内存管理/字符串操作/打印接口基本是对libc的二次封装
- 互斥锁接口直接对接Uniproton的semaphore相关接口

以上接口大部分OS都提供了直接支持，移植上没有太多难度
### env_sleep_msec
这个接口主要在RPMsg-lite申请用于发送消息的buffer时使用。**RPMsg-lite希望该接口可以实现毫秒级的任务休眠**，而**Uniproton提供的PRT_TaskDelay是按照系统tick数进行休眠**，因此在移植时面临两个问题：
  1. 毫秒到tick数的换算
  2. 系统时钟间隔至少不大于1ms才能满足该接口需求

- 对于第一个问题，Uniproton在prt_config.h中定义了
  - 芯片主频（OS_SYS_CLOCK）
  - **一个系统tick的长度（OS_TICK_PER_SECOND）（单位：频率）**.[1]

  所以，系统每秒的tick数 = OS_SYS_CLOCK/OS_TICK_PER_SECOND，除以ms:s的比例（1000:1）就可以计算出每ms的tick数

- 对于第二个问题，
  - QEMU RISC-V设备的芯片主频是$1.0 \times 10^{8}$，Uniproton设置的系统tick长度为$1.0 \times 10^{6}$，每秒的tick数=$\dfrac{1.0 \times 10^{8}}{1.0 \times 10^{6}}$=$1.0 \times 10^{2}$，**tick间隔超过了1ms，无法满足env_sleep_msec的需求。**
  - 为了能够正常使用该接口，现在将QEMU RISC-V上Uniproton的系统tick长度缩短到$1.0 \times 10^{5}$，即$1 \text{ tick/ms}$，刚好可以满足该接口需求
### RPMsg-lite queue相关接口
RPMsg-lite未定义queue，仅定义了标准的env_queue相关接口用于对接OS实现的queue。RPMsg-lite queue的移植过程中有以下问题
  1. RPMsg-lite queue定义返回0为error，非0为success。但是Uniproton实现的PRT_queue定义的逻辑相反。
  2. PRT_queue支持紧急消息，即允许后插入的高优先级的消息先被处理，因此需要为每个消息指定优先级。而RPMsg-lite queue是无优先级的FIFO队列。
- 对于第一个问题，目前有两个解决方案
  - 对PRT_queue的返回值进行处理，非0则返回1，反之返回0。但是PRT_queue有多种错误返回值，这样做可能会导致无法直接根据返回值判断error类型（**目前使用方案**）
  - 修改RPMsg-lite对返回值的处理逻辑
- 对于第二个问题，目前有三种解决方案（前两种不涉及RPMsg-lite源码修改）
  - 所有消息都采用同一优先级，使用RPMsg-lite queue时的PRT_queue退化为标准FIFO队列（**目前使用方案**）
  - 要求用户通过RPMsg-lite queue发送的报文进行二次封装，在报文中指定消息的优先级，并在插入PRT_queue中时解析报文获取优先级
  - 修改RPMsg-lite的源码，增加参数用于传递消息优先级
## platform layer移植
Platform layer与硬件环境强耦合，最关键的是RPMsg-lite中断发送和中断回调接口的实现。
### RPMsg-lite中断发送接口
发送接口的名字是platform_notify，除了发送IPI，这个接口还需要向目标核传递发送请求的信道（一个整型）。Uniproton目前主要是面向单核的RTOS（目前加入了SMP支持，但是RISC-V上还未适配），因此对于中断的支持还不完善，为了实现跨核通信，目前在Uniproton RISC-V上添加了基础的IPI module。
### RPMsg-lite中断回调接口
RPMsg-lite对于中断回调没有太多要求，仅需要在中断回调中调用env_isr并传入vector_id以进入RPMsg-lite的接收流程。目前定义了一个简易的中断回调函数（默认仅有2个core）。目前在第一次RPMsg-lite设备初始化时注册到IPI module中。
### Uniproton RISC-V IPI module
#### 中断向量
Uniproton目前的中断处理不是中断向量表的模式（即硬件根据中断号直接跳转到指定的处理函数），而是一个单一的中断入口，通过软件逻辑判断当前中断/异常类型。目前加入了对于IPI的判断（通过判断mcause寄存器中当前异常的原因），如果当前异常原因是IPI中断，则跳转到应用注册的用户中断回调函数.[2]。
#### 中断发送
发送中断通过设置CLIINT寄存器的值完成
#### 中断数据传递——简单mailbox
RPMsg-lite在发送IPI的时候需要额外传递vector_id，目前在物理内存中开辟了4KB的区域作为mailbox，在发送IPI时将vector_id存入mailbox中对应核的区域，在处理IPI时再从mailbox中读出。
# QA
- [1] : Uniproton在这个宏的命名上有问题，这个宏不是表示每秒tick数的，而是表示每个tick长度

- [2] : 目前Uniproton RISC-V IPI module中的用户中断回调仅考虑了RPMsg-lite（暂时没有其它IPI的需求），因此只考虑了一个用户中断回调，只提供了一个函数指针。