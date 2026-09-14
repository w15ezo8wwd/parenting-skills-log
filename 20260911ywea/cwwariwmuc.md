# 玄铁处理器的Linux移植

> 更新时间：2026-09-11 (UTC+8)

操作系统是控制硬件资源和应用程序的软件，它与硬件部分交互，为包含在硬件平台上的所有可编程部件提供服务；为运算在操作系统上的应用程序提供执行环境。其与CPU存在密切的联系和互动，它们共同协作来管理计算机系统的各种任务，例如进程管理、资源分配、中断处理、多任务管理等等。操作系统赖于CPU来执行其功能，而CPU依赖于操作系统来协调和控制多个任务的执行，以及提供对硬件的抽象和管理。这种协作使计算机系统能够高效地运行各种应用程序。因此离开了操作系统，CPU只是一个孤立的硬件，无法实现高效的利用。

​ 本专栏前面有一系列的文章介绍玄铁C910处理器，这是一款使用RISC-V指令集而设计的一款高性能处理器。为了打造出能支撑它的生态系统，平头哥开源了处理器核的源代码，让众多芯片工程师能够基于它打造起平台生态，而生态系统最重要的一环则是与处理器相配合的操作系统。开源的Linux系统支持多种指令集，我们可以对其做一些适配工作来支持RISC-V指令集，使其能在以C910为处理器核的硬件系统中运行。因此本项目基于开源的玄铁C910处理器核搭建SoC，并实现Linux的移植，进一步丰富开源玄铁系列的生态。
Linux移植过程和所需最小SoC结构
嵌入式linux系统移植的步骤：

搭建交叉开发环境

​ 需要搭建交叉开发环境在pc机上编译出符合相应嵌入式系统体系结构和指令集的机器码，同时确定目标机和主机的连接方式，搭建数据传输通道。

​ 一般而言，目标机、主机的连接方式有：

UART异步串行通信接口

USB串行通信接口

TCP/IP网络通信接口

Debug Jtag调试接口

连接方式的选择将会相应决定需要给SoC增加什么外设。

2. 移植u-boot

u-boot是一个通用的引导程序，也就是bootloader的一种，它的功能是引导操作系统，即将内核加载到DDR中，而DDR必须要软件进行初始化才能运行，因此在搬运内核之前U-boot还需要初始化内存。U-boot的启动过程分为stage1和stage2两大部分，处理器驱动一般在stage1用汇编语言来实现，这一部分代码位于start.s文件中，实现定义执行入口，设置异常向量，设置中断控制器等功能。而stage2则一般用c语言实现，包括调用一系列初始化函数，初始化flash等系统设备。

3. Kernel的配置，编译和移植

​ Kernel本质上也是一个程序，从开始运行一直进入到最后的while(1),内核一旦成功运行，U-boot即完成工作，内核则会一直在内存中运行，直到系统复位重启。内核会根据U-boot传递的参数去指定地址寻找根文件系统rootfs,一旦找到rootfs之后，控制权则会传递给文件系统。

4. 根文件系统的制作

​ Linux需要在一个分区上存放系统启动的必要文件，内核启动运行后的第一个程序(init一号进程)、用于挂接文件系统的脚本、shell程序等，这些必要的文件的集合称为根文件系统，根文件系统的制作和移植则是Linux系统移植的最后一部。

​ 通过上述四个步骤我们可以确定一个能运行Linux系统的SoC最少需要如下外设：

UART异步串行通信接口：实现目标机和主机交互

Debug Jtag调试接口：这里是根据选择的数据传输通道确定，用于完成程序的加载

BootRom:一小块内存单元，包含处理器在上电或者复位时执行的第一个代码。用来存储引导程序U-boot或者其他外设测试程序，一旦系统上电处理器就会运行BootRom中存储的程序。

SPI接口：与开发板上的flash通讯，搬运存储在flash中的Linux内核和根文件系统。

DDR控制器：访问DDR4，用于存储处理器运行过程中产生的数据。

​ 因此玄铁C910处理器的Linux移植一共分为两个大部分，第一步是完成上述外设的添加，第二步则是进行Linux移植。本文章将会介绍如何搭建起一套可以正常运行的简易SoC,后续会有其他文章介绍软件的移植。

​ 平头哥开源了C910的小型测试平台smart_run,将会以此为基础来进行SoC搭建。对于DDR或者SRAM等需要高带宽的外设来说，一般是采用AXI或者AHB总线，由于AXI具有完全独立的读写通道，可以实现数据传输的全双工模式，所以当SRAM为双端口时，AXI读写性能相较于AHB会有较大提升，因此SRAM和DDR控制器都采用AXI接口。而SPI接口，UART接口等外设则实现APB接口与核进行通讯，整个SoC结构如下图：
SoC结构
​ AXI总线一共有一个master和三个slave，master为开源的C910核，其主设备接口支持AMBA 4.0AXI协议，同时还有jtag接口可以用于下载测试程序以及debug。三个slave分别为SRAM，DDR控制器以及AXI到AHB转换器。SRAM可用于存放测试代码，便于后续的硬件功能测试。通过APB Bridge一共在APB上接了四个外设，而APB则通过AHB2APB完成协议转换，再通过AXI2AHB实现和核的通讯。
SoC工程实现SoC设计环境
​ 为了方便连线，将在vivado中搭建整个SoC,并且使用Block Design来简化设计。为此需要提前将开源的C910源码中的cpu_sub_system_axi.v封装成IP核，这个模块将C910核接上了一个AXI的master接口和JTAG接口用于调试，封装之后的IP如图：
C910 IP
​ AXI总线则是用vivado自带IP AXI Interconnect,设置为四个Master Interfaces和一个Slave Interfaces,例化后的IP如下图。S00_AXI用于与C910通讯，其他四个M_AXI则是与其他外设的AXI接口连接，剩下的端口则是相应的时钟和复位信号，需要与所设计时钟相连,在这里AXI Interconnect解决了不同外设之间跨时钟域的问题。
AXI InterconnectSoC仿真和上板验证工具
​ 前文介绍了在哪里搭建SoC,并且例化了处理器核以及AXI总线，接下来则需要将所需要的外设接入总线，但是将外设接上总线之后如何进行仿真和上板验证呢，这一节首先会介绍接在AXI总线上的一块512KB的RAM，这块RAM可以在仿真的时候存储测试代码来进行SoC的功能验证，同时这也是之后移植Linux时存放U-boot的地方。

​ 这块RAM由16个数据宽度为DATAWIDTH，深度为MEMDEPTH的寄存器数组组成，并且会将已经用riscv工具链编译好的可执行文件通过readmemh读入RAM中，可以用于上板进行功能验证。部分代码如下：

reg [(DATAWIDTH-1):0] mem0 [(MEMDEPTH-1):0];reg [(DATAWIDTH-1):0] mem1 [(MEMDEPTH-1):0];reg [(DATAWIDTH-1):0] mem2 [(MEMDEPTH-1):0];...reg [(DATAWIDTH-1):0] mem14 [(MEMDEPTH-1):0];reg [(DATAWIDTH-1):0] mem15 [(MEMDEPTH-1):0];initialbegin $readmemh("D:/Files/Projects/vivado_projects/sram/case0.pat", mem0); $readmemh("D:/Files/Projects/vivado_projects/sram/case1.pat", mem1); $readmemh("D:/Files/Projects/vivado_projects/sram/case2.pat", mem2); ... $readmemh("D:/Files/Projects/vivado_projects/sram/case14.pat", mem14); $readmemh("D:/Files/Projects/vivado_projects/sram/case15.pat", mem15);End
​ 同样需要给这块RAM设计AXI slave接口，再进行封装成IP，最后在Block Design例化，连接上AXI Interconnect之后在Address Editor中设置RAM的总线地址，这里可以先将地址设置为0x0。在完成这一步之后，这个小型的SoC已经可以运行一些简单的程序，只需要将程序用riscv工具链编译好之后，放入RAM中的指定的地址，即readmemh中的绝对地址。除了这种方法也可以通过tb将测试程序写入RAM中。

​ 整个工程上板测试需要用到平头哥提供CPU调试器CK LINK以及调试软件CDK。在上文介绍的SoC结构中，CPU核有一个JTAG接口，CDK可以通过CK LINK将程序通过JTAG接口下载到用户在链接脚本中设置的地址，并且在主机上进行debug,CDK具体使用细节可以参考相应的使用教程。
UART
`define PS1_BASE_START 40'h40015000`define PS1_BASE_END 40'h40015fff`define PS2_BASE_START 40'h40011000`define PS2_BASE_END 40'h40011fff`define PS3_BASE_START 40'h40016000`define PS3_BASE_END 40'h40016fff`define PS4_BASE_START 40'h40010000`define PS4_BASE_END 40'h40010fff
​ 需要注意的是这些地址不能超过在AXI总线中给AXI2AHB分配的地址范围，如果超过了处理器就无法访问这些外设。本工程给UART分配的地址为40‘h40015000,这个地址即为UART的基址，而在UART中还会有许多的配置寄存器用来初始化UART，这些寄存器都有自己的相对于基址的偏移量。当配置UART时，只需要往这个地址写值即可。下面为一段UART的驱动程序:

#define UART0_BASE_ADDR 0x40015000/* UART registers addr definition */#define CK_UART_RBR 0x00 /* Receive Buffer Register (32 bits, R) */#define CK_UART_THR 0x00 /* Transmit Holding Register (32 bits, W) */#define CK_UART_DLL 0x00 /* Divisor Latch(Low) (32 bits, R/W) */#define CK_UART_IER 0x01 /* Interrupt Enable Register (32 bits, R/W) */#define CK_UART_DLH 0x01 /* Divisor Latch(High) (32 bits, R/W) */#define CK_UART_IIR 0x02 /* Interrupt Identity Register (32 bits, R) */#define CK_UART_FCR 0x02 /* fifo Countrol Register (32 bits, W) */#define CK_UART_LCR 0x03 /* Line Control Register (32 bits, R/W) */#define CK_UART_MCR 0x04 /* Modem Control Register (32 bits, W) */#define CK_UART_LSR 0x05 /* Line Status Register (32 bits, R) */#define CK_UART_MSR 0x06 /* Modem Status Register (32 bits, R/W) */#define CK_UART_USR 0x1f /* UART Status Register (32 bits, R/W) */
​ 这段代码定义了UART基址和所有配置寄存器的偏移，这些值与硬件设计相关。例如我们需要通过UART发送数据时，只需要将发送的数据写入到CK_UART_THR对应的地址并且读取状态寄存器也就是CK_UART_LSR的值即可判断数据是否发送成功即可，完整的发送函数如下：

uint32_t ck_uart_putc(p_ck_uart_device uart_device, uint8_t c){ if (uart_device-txmode == DISABLE) return 1; // wait until uart transmit buffer is empty while (!((*(reg8_t*)(uart_device-register_map+CK_UART_LSR)) CK_LSR_TRANS_EMPTY));//extract the CK_LSR_TRANS_EMPTY bit and wait for its validation *(reg8_t*)(uart_device-register_map+CK_UART_THR) = c; while(!((*(reg8_t*)(uart_device-register_map+CK_UART_LSR)) CK_LSR_TRANS_EMPTY)); return 0;}
​ 其他驱动函数也是这种原理，往相应寄存器对应的地址赋值即可，这里就不展开讨论。
DDR控制器MIG核的例化
​ DDR是DDR SDRAM的简称，全称为Double Data Rate Synchronous Dynamic Random Access Memory，中文名为：双倍速率同步动态随机存储器，同步是指需要时钟。根据前面的介绍，我们知道需要将Linux内核搬运至DDR，对于DDR的访问我们不可能直接对DDR的接口信号进行读写操作，这样太过于复杂，而是使用Xilinx提供的IP核MIG，它可以为DDR3、DDR4等多种存储器提供接口。由于所使用的开发板不同，其上带的DDR颗粒型号也会不同，需要根据开发板的官方手册去确定对应的型号来例化MIG。

​ 本项目所采用的开发板为VCU118,开发板能够提供的时钟频率为250MHz,DDR颗粒为MT40A256M16GE-083E,其时钟速率为1200M，时钟周期为833ps。MIG核最关键的配置就是时钟的配置。这里的4:1表示DDR4内存的驱动频率，和用户程序代码运行的时钟频率之比为4:1，DDR4运行于1200MHz，那么用户程序就运行于300MHz。根据上述参数来例化MIG 。Board界面中的CLK需要设置成开发板提供的时钟频率，DDR4选择的是Custom。在Basic中需要设置时钟速率与开发板上的DDR保持一致，同时指定好对应的DDR型号,数据位宽，其他数值保持默认即可。
MIG Board界面MIG Basic界面
​ 这里需要注意的是开发板所能提供的时钟频率为250MHz,而处理器核、SPI接口等外设工作频率达不到这么高，因此需要进行分频，而MIG核则提供了这一项功能。如下图所示，在Advanced Options界面的Additional Clock Outputs提供了四个用户时钟，可以用于其他逻辑电路的时钟驱动，本工程其他外设和处理器核工作时钟频率都为100MHz,因此这里只需要初始化一个用户时钟。

​ 例化完成之后，得到如下的MIG核，这里有两个时钟信号需要注意。首先是c0_ddr4_ui_clk，这个时钟信号是DDR为了能够同步数据输入端的时序，而自己产生出来给用户使用的，也就是说需要将这个时钟信号连接到AXI_Interconnect的M00_ACLK，这样使得CPU可以通过总线控制MIG，而另外一个时钟信号为addn_ui_clkout1，这个则是例化IP时选的用户时钟，可以用来给其他外设提供100MHZ的时钟信号。除此之外MIG还提供给用户端的同步复位信号c0_ddr4_ui_clk_sync_rst,这个信号可以作为之后需要使用的IP核 Processor System Reset的外部复位信号，来实现MIG与系统的同步复位。之后将该IP连接上AXI interconnect并且设置好地址即完成连接。
DDR访问测试
​ 为了测试DDR,需要将MIG核相应的输出引脚分配到VCU188的DDR上。接着通过CDK将测试程序下载到RAM中，测试程序会完成往DDR里写值操作，并且会将写入的值通过UART发送到主机来判断DDR是否能正确读写。除了测试读写功能，当然最重要的是CPU能否读取DDR中存储的程序并且运行，为此需要设置CDK的链接脚本，将程序下载到DDR中，再设置CPU复位之后运行的第一条指令地址为DDR的地址，之后上板验证功能即可。当这一步完成之后，我们成功给SoC添加上了DDR。以下是部分测试代码：

#include "stdio.h"#include "uart.h"int main (void){ t_ck_uart_cfig uart_cfig; t_ck_uart_device uart0; char* str = "0123456789"; int i = 0;char* addr = 0x8000000000;char* p; // setup uart //-------------------------------------------------------- uart_cfig.baudrate = BAUD; // any integer value is allowed uart_cfig.parity = PARITY_NONE; // PARITY_NONE / PARITY_ODD / PARITY_EVEN uart_cfig.stopbit = STOPBIT_1; // STOPBIT_1 / STOPBIT_2 uart_cfig.wordsize = WORDSIZE_8; // from WORDSIZE_5 to WORDSIZE_8 uart_cfig.txmode = ENABLE; // ENABLE or DISABLE uart_cfig.rxmode = ENABLE; // ENABLE or DISABLE // open UART device with id = 0 (UART0) ck_uart_open(uart0, 0); // initialize uart using uart_cfig structure ck_uart_init(uart0, uart_cfig); //--------------------------------------------------------for (i = 0; i = 10; i++){*(char*)(0x8000000000+i) = str[i];}p = 0x8000000003;ck_uart_putc(uart0, *p); return 0;}
SPI接口

​ SPI通讯协议也是一种串行通讯方式，其有单独的数据线和单独的时钟信号来保证发送端和接收端的同步。整体的传输分为以下几个步骤：

 主机先将NSS信号拉低，表示将要开始收发数据；

 当接收端检测到时钟的边沿信号时，它将立即读取数据线上的信号，这样就得到了一位数据（1bit）;

 主机发送到从机时：主机产生相应的时钟信号，然后数据一位一位地将从MOSI信号线上进行发送到从机；

 主机接收从机数据：如果从机需要将数据发送回主机，则主机将继续生成预定数量的时钟信号，并且从机会将数据通过MISO信号线发送；

SPI总线包括四条逻辑线：

MISO：Master input slave output 主机输入，从机输出（数据来自从机）；

MOSI：Master output slave input 主机输出，从机输入（数据来自主机）；

SCLK ：Serial Clock 串行时钟信号，由主机产生发送给从机；

NSS：Slave Select 片选信号，由主机发送，以控制与哪个从机通信，通常是低电有效信号。

​ SPI总线上的主机必须在通信开始时候配置并生成相应的时钟信号。在每个SPI时钟周期内，都会发生全双工数据传输。主机在MOSI线上发送一位数据，从机读取它，而从机在MISO线上发送一位数据，主机读取它。就算只进行单向的数据传输，也要保持这样的顺序。这就意味着无论接收任何数据，必须实际发送一些东西。在这种情况下，我们称其为DUMMY；除了配置时钟频率外，还需要配置时钟极性CPOL和时钟相位CPHA。CPOL是用来决定SCK时钟信号空闲时的电平，CPOL＝0，空闲电平为低电平，CPOL＝1时，空闲电平为高电平。CPHA是用来决定采样时刻的，CPHA=0，在每个周期的第一个时钟沿采样，CPHA＝14，在每个周期的第二个时钟沿采样。

 SPI协议其他细节可以查阅网上相关资料，同时xilinx也提供了相应的IP AXI Quad SPI来实现SPI协议通讯，而本工程则移植开源SoC无剑100的USI接口，该接口配合软件使用可以实现串行通讯。其包含了能实现UART、SPI和I2C通讯的硬件功能，只需要配置控制寄存器即可以实现相应的通讯功能。 

 当将SPI移植到SoC上之后则需要将其连接上FPGA的FLASH颗粒，这里无法直接约束分配引脚，因为外部存储器的时钟管脚一般与fpga的CCLK_0连接，当使用远程更新时，首先fpga内部有控制flash的驱动（即逻辑控制flash时序），当然flash时钟也需要控制了，但这时时钟管脚已经连接到CCLK_0，这个时候就需要STARTUP3原语来进行管脚的分配，当使用STARTUP3之后工程顶层就不会再有SPI接口的四个信号，因为这个原语已经将其接到了FLASH的引脚。例化代码如下：

STARTUPE3 #( .PROG_USR("FALSE"), // Activate program event security feature. Requires encrypted bitstreams. .SIM_CCLK_FREQ(0.0) // Set the Configuration Clock Frequency (ns) for simulation. ) STARTUPE3_inst ( .CFGCLK(), // 1-bit output: Configuration main clock output. .CFGMCLK(), // 1-bit output: Configuration internal oscillator clock output. .DI({1'b0,1'b0,sd1_pad_0,1'b0}), // 4-bit output: Allow receiving on the D input pin. .EOS(), // 1-bit output: Active-High output signal indicating the End Of Startup. .PREQ(), // 1-bit output: PROGRAM request to fabric output. .DO({1'b1,1'b1,1'b0,sd0_pad_1}), // 4-bit input: Allows control of the D pin output. .DTS(4'b0010), // 4-bit input: Allows tristate of the D pin. .FCSBO(nss_pad_0), // 1-bit input: Controls the FCS_B pin for flash access. .FCSBTS(0), // 1-bit input: Tristate the FCS_B pin. .GSR(0), // 1-bit input: Global Set/Reset input (GSR cannot be used for the port). .GTS(0), // 1-bit input: Global 3-state input (GTS cannot be used for the port name). .KEYCLEARB(0), // 1-bit input: Clear AES Decrypter Key input from Battery-Backed RAM (BBRAM). .PACK(0), // 1-bit input: PROGRAM acknowledge input. .USRCCLKO(sclk_pad_0), // 1-bit input: User CCLK input. .USRCCLKTS(0), // 1-bit input: User CCLK 3-state enable input. .USRDONEO(), // 1-bit input: User DONE pin output control. .USRDONETS(0) // 1-bit input: User DONE 3-state enable output. );
​ 这个原语的电路结构图如下：
STARTUP3电路结构
​ 可以看到这个原语将FLASH颗粒的inout类型的芯片引脚通过三态门缓冲器变成与SPI连接的输入输出以及控制信号，由于时钟信号以及片选信号是由SPI接口输出给flash颗粒，因此对应的控制信号USRCCLKTS和FCSBTS需要置零。这里比较复杂的则是数据信号的连接。上图中FLASH颗粒的数据通道为DQ[3:0]，这是双向通道。具有四位的原因是flash最高可以支持QUAD SPI协议，而本工程SPI协议只支持双通道通信，这个时候DQ的高两位数据则会变成FLASH的复位和保护信号，具体细节可以去查看flash颗粒对应的datasheet,因此我们需要将高两位的输入置一。在双通道通讯协议下，DQ[0]即上面介绍的MOSI信号，DQ[1]为MISO信号，因此只需要将SPI接口相应信号连接到STARUP3原语的DO信号对应的位即可。这里需要注意的是，如果是MISO信号则要将DTS对应的位置一，否则由电路图可以得知，SPI收到的信号io_i将是io_o信号而不是DQ[1],因此STARTUP3D DTS信号我们赋值4‘b0010.

​ 到这一步也就完成了SPI接口的连接，接下来需要写驱动程序。驱动程序与UART的类似，也是需要设置好SPI的基址以及对应配置寄存器和读写FIFO的偏移地址。VCU118上的flash颗粒型号是MT25QU01GBBB，每个地址能写入一个字节的数据，我们设置SPI FIFO的长度为一个字节。当向flash写入数据时，需要将数据逐字节的写入FIFO中，当FIFO不为空时，SPI会一直发送数据，直到全部发送完成。从FLASH读取数据也同理。接下来是一段简单的写flash函数：

void WriteByte(uint32_t str,uint8_t data){ int reg,tx_empty,rx_empty,spi_busy,a; uint8_t *array; Variable32ToArray(array, str);/******发送写使能指令***********/ REG32(USI0_BADDR+TX_FIFO) = WREN;/*****等待指令发送完成********/ do{ reg = REG32(USI0_BADDR+FIFO_STA); tx_empty = reg 0x1; reg = REG32(USI0_BADDR+SPI_STA); spi_busy = reg 0x1; } while(tx_empty==0 || spi_busy==1);/*****发送三字节地址访问指令********/ REG32(USI0_BADDR+TX_FIFO) = THRADR; ... REG32(USI0_BADDR+TX_FIFO) = WREN; ... /*****发送写FLASH指令以及发送写入的地址和数据********/ REG32(USI0_BADDR+TX_FIFO) = PROGRAM; REG32(USI0_BADDR+TX_FIFO) =*(array+1); REG32(USI0_BADDR+TX_FIFO) =*(array+2); REG32(USI0_BADDR+TX_FIFO) =*(array+3); REG32(USI0_BADDR+TX_FIFO) =data; ... /*********清空接收FIFO*************/ do{ reg = REG32(USI0_BADDR+FIFO_STA); rx_empty = reg 0x4; if(rx_empty == 0){ reg = REG32(USI0_BADDR+RX_FIFO); } } while(rx_empty==0); /**********读取FLASH状态寄存器来判断是否将数据写入FLASH中********/ do { REG32(USI0_BADDR+TX_FIFO) = 0x05; REG32(USI0_BADDR+TX_FIFO) = 0xFF; ... do{ reg = REG32(USI0_BADDR+FIFO_STA); rx_empty = reg 0x4; if(rx_empty == 0){ a = REG32(USI0_BADDR+RX_FIFO);//ck_uart_putc(uart, a); } } while(rx_empty==0); }while(a!=0); }
​ FLASH的指令编码以及读写时序需要去阅读相应的datasheet确定。到这里就完成了整个SoC搭建
bootloader跳转和搬运的简单测试
​ 我们知道存储在bootrom里的u-boot的一个重要功能是把flash中的linux内核搬运至ddr并跳转，为了测试硬件电路能够实现这个功能，我们可以编写一个简单的bootloader程序来将flash里的一段程序搬运至DDR中并跳转执行。

​ 首先将一段通过uart输出字符的程序通过riscv工具链编译成bin文件之后，可以通过vivado烧录至flash中。再编写bootloader函数如下:

void (*FUN)(void) = 0x8000;int main(){ // TODO REG32(USI0_BADDR+MODE_SEL) = 0x2; //F(sclk) : 100Khz //CLK_DIV0 = 20M/100K = 200 REG32(USI0_BADDR+CLK_DIV0) = 0xC8; //master mode REG32(USI0_BADDR+SPI_MODE) = 0x1; //data_size=8 //Transmit Receive //CPHA=0 //CPOL=0 REG32(USI0_BADDR+SPI_CTRL) = 0x07; //enable SPI //enable TX FIFO //enable RX FIFO REG32(USI0_BADDR+USI_CTRL) = 0xf; ReadPage(0, (char*)0x8000, 0); FUN(); return 0;}
​ 首先完成SPI的驱动，再通过ReadPage函数将flash地址从0开始的150KB数据读到DDR中的0x8000处，之后运行FUN函数，也就是0x8000处存储的程序，当成功运行之后我们可以在串口助手读到如下结果：
执行结果
​ 这就说明bootloder成功搬运程序和跳转执行，至此linux移植所需的硬件就全部搭建并测试完成.
总结
​ 本文介绍了能够运行Linux操作系统的最小SoC结构，并且展示了如何一步一步搭建SoC以及测试方法，在完成了硬件系统之后，就需要开始u-boot的移植，这将在后续文章中介绍u-boot的修改以及移植。

## 相关阅读

- ["疯狂转盘抽大奖  体验京津冀最高端CT"微信有奖活动邀您参加！](https://github.com/exfk8bm0mc/child-care-diary/blob/main/20260911acmc/koqozvqspq.md)
- [郑州市妇幼保健院试管婴儿费用表？会有哪些问题！](https://github.com/b1xp80vbpv/pregnancy-care-hub/blob/main/20260910vwhn/ioswfiipho.md)
- [武汉哪些医院提供试管婴儿？附上医院和费用介绍](https://github.com/h538vradpp/child-care-essays/blob/main/20260910edsg/esxmzlezhy.md)
- [宁波地区试管婴儿的平均费用是多少?](https://github.com/y9qvvxks1i/pregnancy-care-hub/blob/main/20260910alcw/bwfhtknrfm.md)
- [试管婴儿失败有哪些原因](https://github.com/y9qvvxks1i/pregnancy-care-hub/blob/main/20260910alcw/lzaoqjpiyt.md)
- [聚焦团队｜泌尿外科：扶危渡厄  无愧先锋](https://github.com/l9lvqnbe4d/child-care-diary/blob/main/20260911sbmp/vbfyyudcvu.md)
- [宝宝8个月，大人吃了一次毓婷喂奶了会怎样？](https://github.com/l5q2j5iic2/pregnancy-care-hub/blob/main/20260910nxdo/dnwhixrwue.md)
- [婴儿不睡觉的生理性原因,婴儿睡觉应该如何护理](https://github.com/iebkyzpjrn/baby-care-journal/blob/main/20260910eulh/ejjoourxcq.md)
- [今日刚刚！邯郸做三代助孕试管的私立试管机构名单](https://github.com/olvqsk2upx/parenting-daily-tips/blob/main/20260910qtge/qvoiwjycws.md)
- [说一说试管婴儿龙凤胎多少钱(试管婴儿龙凤胎花费)](https://github.com/opigz3v852/parenting-daily-tips/blob/main/20260910xhnl/ynkckpotqn.md)
- [胆囊隆起样病变不容忽视，可能为腺瘤样息肉](https://github.com/rnf9cvz5iw/mother-baby-diary/blob/main/20260911ndmf/neewskyzgx.md)
- [凯越团购 凯越官方旗舰店](https://github.com/ualf0k98cv/parenting-daily-tips/blob/main/20260910fplj/tdalhwuorj.md)

## 推荐站点

- [['https://www.huaiyunq.cn/110194111123.html', '高龄备孕警惕卵巢功能衰退的早期症状与饮食调理']](https://www.huaiyunq.cn/110194111123.html)
- [['https://www.cddyunw.com/421771660401.html', '宝宝第11天迎来翻倍喜讯，我的辅助生殖旅程']](https://www.cddyunw.com/421771660401.html)
- [['https://www.apkbwvg.cn/danshenqiuzi/104.html', '试管婴儿前卵泡调理的科学方法与营养建议']](https://www.apkbwvg.cn/danshenqiuzi/104.html)
- [['https://www.szgwzx.cn/172.html', '排卵期淡粉色出血的原因与代孕流程解析']](https://www.szgwzx.cn/172.html)
- [['https://www.super-hn.cn/202534056470.html', '肇庆三代试管助孕费用全解析：流程、成功率与费用构成深度分析']](https://www.super-hn.cn/202534056470.html)
- [['https://www.wqxmm.cn/322724566279.html', '代孕试管中心哪家好,取卵要慎重，专业的才是最好的！']](https://www.wqxmm.cn/322724566279.html)
- [['https://www.sandwnot.com/111062712060.html', '2026年辽宁比较好的三代试管医院有几家']](https://www.sandwnot.com/111062712060.html)
- [['https://www.weywjei.cn/20250826-178.html', '借卵试管代怀：二甲双胍与空卵泡的调理及子宫穿孔风险']](https://www.weywjei.cn/20250826-178.html)
- [['https://www.sdxxy.cn/20250517-457.html', '承德哪家医院做代生及费用成功率比较高']](https://www.sdxxy.cn/20250517-457.html)
- [['https://www.gzgudadl.cn/1712677025537.html', '南华附一哪做代生好成功率：南华附一生殖中心怎么样']](https://www.gzgudadl.cn/1712677025537.html)
- [['https://www.lianhuahushengqun.cn/215203314361.html', '可靠供卵资询：孕早期褐色分泌物持续多长时间']](https://www.lianhuahushengqun.cn/215203314361.html)
- [['https://www.eduency.com/320621201252.html', None]](https://www.eduency.com/320621201252.html)
- [['https://www.luruihang.com/2110.html', '宫腹腔镜后怀孕几率']](https://www.luruihang.com/2110.html)
- [['https://www.cd-hssf.com/212190931534.html', '试管代生官网：2026青岛排名最好试管婴儿医院推荐，青大附院位列前三']](https://www.cd-hssf.com/212190931534.html)
- [['https://www.jmxmintuhg.cn/20250511-170.html', '备孕五年的大龄夫妇，在传承生殖做两次试管婴儿，生下健康男宝宝']](https://www.jmxmintuhg.cn/20250511-170.html)
- [['https://www.dgshengxigongchengsl.cn/3726215476857.html', '昆明包成功代生地址成功率高的医院真实成功率']](https://www.dgshengxigongchengsl.cn/3726215476857.html)
- [['https://www.ewdboe.cn/305692240047.html', None]](https://www.ewdboe.cn/305692240047.html)
- [['https://www.xczxcy.com/29.html', '为什么拉拉都很好看？拉拉伴侣如何通过A卵B怀拥有宝宝']](https://www.xczxcy.com/29.html)
- [['https://www.toothree006.cn/323603034302.html', '供卵机构-高龄女性试管有哪些风险和危害？除了染色体异常这三大风险要知道']](https://www.toothree006.cn/323603034302.html)
- [['https://www.njxxwcr.cn/daishengfeiyongmingxi/142.html', '助孕双胞胎费用及移植后艾草泡脚的利弊分析，代生经验分享']](https://www.njxxwcr.cn/daishengfeiyongmingxi/142.html)
- [['https://www.vhpowpj.cn/20250821-171.html', '优化卵子质量与数量：助孕备孕的中药调理之道']](https://www.vhpowpj.cn/20250821-171.html)
- [['https://www.hg00fj88.com/2276.html', '试管代生网-卵巢早衰没有窦卵泡还有救么？别担心，有四种解救办法总有一种适合你']](https://www.hg00fj88.com/2276.html)
- [['https://www.dymgp.com/8020.html', '宁波子宫脱垂的治疗：宁波子宫脱垂的治疗医生']](https://www.dymgp.com/8020.html)

*本文整理自母婴健康资讯，仅供科普参考。*
