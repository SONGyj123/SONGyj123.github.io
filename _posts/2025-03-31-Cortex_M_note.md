# Cortex M系列笔记

## 1. Architecture

### 1. 寄存器
<style>
  .highlight-row {
    background-color: yellow; /* 设置行的背景色 */
  }
  .highlight-cell {
    background-color: yellow; /* 设置单元格的背景色 */
  }
</style>
<table>
  <thead>
    <tr>
      <th>寄存器</th>
      <th>功能</th>
      <th>备注</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>r0~r12</td>
      <td>通用</td>
      <td></td>
    </tr>
    <tr>
      <td style="color: red;" class="highlight-cell">MSP PSP（r13）</td>
      <td class="highlight-cell">栈指针寄存器</td>
      <td class="highlight-cell">默认MSP 支持OS使用</td>
    </tr>
    <tr>
      <td style="color: red;" class="highlight-cell">LR（r14）</td>
      <td class="highlight-cell">返回地址寄存器</td>
      <td class="highlight-cell">函数返回 异常返回都是更新LR</td>
    </tr>
    <tr>
      <td style="color: red;" class="highlight-cell">PC（r15）</td>
      <td class="highlight-cell">PC</td>
      <td class="highlight-cell">读取返回当前指令地址+4（pipleline）</td>
    </tr>
  </tbody>
</table>



<table>
  <thead>
    <tr>
      <th>寄存器</th>
      <th>功能</th>
      <th>备注</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>APSR</td>
      <td>Application PSR</td>
      <td></td>
    </tr>
    <tr>
      <td>EPSR</td>
      <td>Execution PSR</td>
      <td></td>
    </tr>
    <tr>
      <td>IPSR</td>
      <td>Intertupt PSR</td>
      <td>IPSR[8:0] 9位表示全部512个异常</td>
    </tr>
    <tr>
      <td>CONTROL</td>
      <td>控制寄存器</td>
      <td></td>
    </tr>
  </tbody>
</table>



**cm4**就是**cm3**加上一个**DSP指令**再加**FPU**

**cm3**有两种模式**Thread模式**和**Handler模式**

### 2.函数调用规则AAPCS

![](https://i.imgur.com/JTLXmqo.png)

## 2.异常和中断

### 1.中断向量表

<style>
  .highlight-row {
    background-color: yellow; /* 设置行的背景色 */
  }
  .highlight-cell {
    background-color: lightblue; /* 设置单元格的背景色 */
  }
</style>

<table>
  <tr>
    <th class="highlight-cell">VTOR</th>
    <th>Exception Number</th>
    <th>Armv6-M &lt;Priority&gt;</th>
    <th>Armv7-M</th>
    <th>Armv8-M</th>
  </tr>
  <tr>
    <td>0x0000</td>
    <td>0</td>
    <td style="color: green;">MSP</td>
    <td style="color: green;">MSP</td>
    <td style="color: green;">MSP</td>
  </tr>
  <tr>
    <td>0x0004</td>
    <td>1</td>
    <td>Reset&lt;-3&gt;</td>
    <td>Reset&lt;-3&gt;</td>
    <td>Reset&lt;-4&gt;</td>
  </tr>
  <tr>
    <td>0x0008</td>
    <td>2</td>
    <td>NMl&lt;-2;&gt;</td>
    <td>NMI&lt;-2&gt;</td>
    <td>NMl&lt;-2&gt;</td>
  </tr>
  <tr>
    <td>0x0012</td>
    <td>3</td>
    <td>HardFault&lt;-1&gt;</td>
    <td>HardFault&lt;-1&gt;</td>
    <td style="color: red;">HardFault&lt;-1&gt;<br>Secure HF(NS=0)&lt;-1&gt;<br>Secure HF(NS=1)&lt;-3&gt;</td>
  </tr>
  <tr>
    <td>0x0016</td>
    <td>4</td>
    <td></td>
    <td style="color: red;">MemManage</td>
    <td>MemManage</td>
  </tr>
  <tr>
    <td>0x0020</td>
    <td>5</td>
    <td></td>
    <td style="color: red;">BusFault</td>
    <td>BusFault</td>
  </tr>
  <tr>
    <td>0x0024</td>
    <td>6</td>
    <td></td>
    <td style="color: red;">UsageFault</td>
    <td>UsageFault</td>
  </tr>
  <tr>
    <td>0x0028</td>
    <td>7</td>
    <td></td>
    <td></td>
    <td style="color: red;">SecureFault</td>
  </tr>
  <tr>
    <td>0x0032</td>
    <td>8</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>0x0036</td>
    <td>9</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>0x0040</td>
    <td>10</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>0x0044</td>
    <td>11</td>
    <td>SVCall &lt;configurable&gt;</td>
    <td>SVCall</td>
    <td>SVCall</td>
  </tr>
  <tr>
    <td>0x0048</td>
    <td>12</td>
    <td></td>
    <td style="color: red;">DebugMonitor</td>
    <td>DebugMonitor</td>
  </tr>
  <tr>
    <td>0x0052</td>
    <td>13</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <td>0x0056</td>
    <td>14</td>
    <td>PendSV</td>
    <td>PendSV</td>
    <td>PendSV</td>
  </tr>
  <tr>
    <td>0x0060</td>
    <td>15</td>
    <td>SysTick, optional</td>
    <td style="color: red;">SysTick</td>
    <td>SysTick</td>
  </tr>
  <tr>
    <td>0x0064</td>
    <td>16</td>
    <td>External Interrupt(0)</td>
    <td>External Interrupt(0)</td>
    <td>External Interrupt(0)</td>
  </tr>
  <tr>
    <td>0x0068</td>
    <td>17</td>
    <td>External Interrupt(1)</td>
    <td>External Interrupt(1)</td>
    <td>External Interrupt(1)</td>
  </tr>
  <tr>
    <td>0x0072</td>
    <td>18</td>
    <td>External Interrupt(2)</td>
    <td>External Interrupt(2)</td>
    <td>External Interrupt(2)</td>
  </tr>
  <tr>
    <td>...</td>
    <td>...</td>
    <td>...</td>
    <td>...</td>
    <td>...</td>
  </tr>
  <tr>
    <td>0x2040</td>
    <td>510</td>
    <td>External Interrupt(494)</td>
    <td>External Interrupt(494)</td>
    <td>External Interrupt(494)</td>
  </tr>
  <tr>
    <td>0x2044</td>
    <td>511</td>
    <td>External Interrupt(495)</td>
    <td>External Interrupt(495)</td>
    <td>External Interrupt(495)</td>
  </tr>
</table>
前16个是异常，后面(512-16)个是中断，各芯片厂自己的外设就是设置在后面的(512-16)，比如串口什么的。







### 2.异常处理流程

cm3在异常处理中的一大特色就是可以直接使用c语言来编写exception_handler(),在cortex-A或者是riscv中，异常的返回都需要使用特殊的指令**eret sret mret**

不同与其他架构例如cortex-A，m3会使用硬件将栈帧(stack frame)压栈而不需要手动压栈，这也是为什么我们在cm相关的库文件中没有看见过压栈操作的原因。cm3硬件压栈只保存八个寄存器，剩余的callee保存的，所以编译器在编译c代码时会根据这个procedure协议生成保存剩余寄存器的汇编代码。

* ***cm3栈帧结构：***

cm3的栈帧可以设置8byte对齐也就是他说的**double word align**，可以注意到cm3的栈帧结构就是8byte：

![](https://i.imgur.com/eN2VLrW.png)

注意下图的**padding**，栈从高地址向低地址生长，这个padding是在栈帧之前的，因为在入栈时栈可能并没有8byte对齐。

![](https://i.imgur.com/c8Fl4X1.png)



* ***cm3硬件压栈顺序：***

```c=
//和栈帧的位置并不相同
PC -> PSR -> R0 -> R1 -> R2 -> R3 -> R12 -> LR
```

压栈顺序和栈帧的顺序是不一样的，首先会压入PC和PSR，这样做可以更快开始**vector fetch**，**cm3是哈佛结构**，数据总线和指令总线是分开的这样一来压栈和取值的操作可以并行进行

MSP和PSP的切换是通过设置**CONTROL**寄存器的**bit1**来完成的(0使用MSP 1使用PSP)，值得一提的是**中断嵌套时的压栈必须使用MSP**


## 3. cortex-m3 启动

### 

| BOOT0 | BOOT1 |  启动介质  |  Column 3   |
| :---: | :---: | :--------: | :---------: |
|   x   |   0   |    SRAM    | 0x2000 0000 |
|   0   |   1   |   Flash    | 0x0800 0000 |
|   1   |   1   | System ROM | 0x1FFF B000 |

### Flash

mcu内置的Flash一般是nor flash可以任意读写，所以stm32就使用的这类Flash。
nand flash是块读写，不能任意读写。这类芯片需要将所有拷贝到RAM。

[存储区别](https://www.51zxw.net/TechArticleDetails.aspx?zid=471&id=2196)

### 1 从flash启动流程

*前提：*
硬件可以将**FLASH**以及**System ROM**映射到**0x00000000**开始的一段空间上，所以在设置了**boot pin**脚电平之后，**FLASH**或者**SYS ROM**当中的前一小段数据就可以从**0x00000000**访问了。

1. 上电之后硬件复位会自动进行如下操作：
从**0x00000000**读取数据写入**MSP**(main stack pointer)
在**0x00000000**存放**stack**地址
从**0x00000004**读取数据写入**PC**(program counter)
在**0x00000004**存放**Reset_Handler()** 的入口地址

2. **Reset_Handler()** 做的事情是：
    1. copy data from flash to sram
    2. initialize .data .bss in RAM
    3. bring up main()

**为什么要将数据拷贝到RAM中去呢？**

虽然单片机上的flash也可以执行和读写程序，但是SRAM比起flash更快读写更flexible，所以更倾向将需要大量改动的数据存储在SRAM中，如果SRAM足够大，我们甚至想把代码也copy到SRAM中去执行就像A架构的芯片使用DDR一样。

**为什么拷贝的数据只包含.data？**

.bss不需要copy只需要空出来赋值0，因为这里面都是没有赋值的变量，.bss段甚至不再bin文件中，这也是.bss段存在的意义，减小bin文件的大小，减少需要搬移的数据。
.rodata和.text不会改动所以不用拷贝

**链接的时候我们不就已经确定了程序的加载位置吗？如果改变了数据段的位置，程序又怎么能正确的找到重定位之后的数据段呢？**

linkerscript中每一个输出段都有两个地址：**LMA**(load memory addr)和**VMA**(virtual memory addr)，下图中**RAM**是**VMA**，flash是**LMA**。

```ld=
/* 
程序在执行的时候知道data段是存放在RAM当中的所以链接的地址都是RAM的
但是ld加载器把data段初始值是放在FLASH当中的，所以startup需要copy data from FLASH to RAM
*/
    .data  : AT ( _sidata )
    {
	    . = ALIGN(4);
        /* This is used by the startup in order to initialize the .data secion */
        _sdata = . ;
        
        *(.data)
        *(.data.*)

	    . = ALIGN(4);
	    /* This is used by the startup in order to initialize the .data secion */
   	 _edata = . ;
    } >RAM
```

<details>
  <summary>完整的代码加载到片上flash数据加载到sram的ld脚本</summary>

```ld=
//需要将数据段copy到sram
ENTRY(Reset_Handler)

/* Highest address of the user mode stack */
_estack = 0x20005000;    /* end of 20K RAM */

/* Generate a link error if heap and stack don't fit into RAM */
_Min_Heap_Size = 0;      /* required amount of heap  */
_Min_Stack_Size = 0x100; /* required amount of stack */

/* Specify the memory areas */
MEMORY
{
  FLASH (rx)      : ORIGIN = 0x08000000, LENGTH = 128K
  RAM (xrw)       : ORIGIN = 0x20000000, LENGTH = 20K
  MEMORY_B1 (rx)  : ORIGIN = 0x60000000, LENGTH = 0K
}

SECTIONS
{
  /* The startup code goes first into FLASH */
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector)) /* Startup code */
    . = ALIGN(4);
  } >FLASH

  /* The program code and other data goes into FLASH */
  .text :
  {
    . = ALIGN(4);
    *(.text)           /* .text sections (code) */
    *(.text*)          /* .text* sections (code) */
    *(.rodata)         /* .rodata sections (constants, strings, etc.) */
    *(.rodata*)        /* .rodata* sections (constants, strings, etc.) */
    *(.glue_7)         /* glue arm to thumb code */
    *(.glue_7t)        /* glue thumb to arm code */

    KEEP (*(.init))
    KEEP (*(.fini))

    . = ALIGN(4);
    _etext = .;        /* define a global symbols at end of code */
  } >FLASH

  /* used by the startup to initialize data */
  _sidata = .;

  /* Initialized data sections goes into RAM, load LMA copy after code */
  .data : AT ( _sidata )
  {
    . = ALIGN(4);
    _sdata = .;        /* create a global symbol at data start */
    *(.data)           /* .data sections */
    *(.data*)          /* .data* sections */

    . = ALIGN(4);
    _edata = .;        /* define a global symbol at data end */
  } >RAM

  /* Uninitialized data section */
  . = ALIGN(4);
  .bss :
  {
    /* This is used by the startup in order to initialize the .bss secion */
    _sbss = .;         /* define a global symbol at bss start */
    __bss_start__ = _sbss;
    *(.bss)
    *(.bss*)
    *(COMMON)

    . = ALIGN(4);
    _ebss = .;         /* define a global symbol at bss end */
    __bss_end__ = _ebss;
  } >RAM

  PROVIDE ( end = _ebss );
  PROVIDE ( _end = _ebss );

  /* User_heap_stack section, used to check that there is enough RAM left */
  ._user_heap_stack :
  {
    . = ALIGN(4);
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(4);
  } >RAM
}
```
</details>



### 2. 从外部flash启动流程

即使需要从外部flash启动最开始也是在内部flash开始的：

1. 在内部flash初始化中断向量表，初始化外部flash（例如spi flash需要初始化spi读写）
2. 重定位中断向量表到sram
3. 从外部flash拷贝代码到sram

### 3. 中断向量表重定位

单片机程序做的第一件事情就是设置中断向量表，因为有NMI（non Maskable interrupt）可能会在上电之后马上发生。cm3和cm4的中断向量表中存放的是各个中断处理函数的起始地址

cm3和cm4提供vtor（vector table offset reg）reset value是0写vtor之前需要先将向量表搬移到指定位置
```c=
#define HW32_REG(ADDRESS) (*((volatile unsigned long *)(ADDRESS)))
#define VTOR_NEW_ADDR 0x20000000

for (i=0;i<48;i++)
{    
    // Assume maximum number of exception is 48
    // Copy each vector table entry from flash to SRAM
    HW32_REG((VTOR_NEW_ADDR + (i<<2))) = HW32_REG((i<<2));
}

__DMB(); // Data Memory Barrier to ensure write to memory is completed

SCB->VTOR = VTOR_NEW_ADDR; // Set VTOR to the new vector table location

__DSB(); // Data Synchronization Barrier to ensure all subsequence instructions use the new configuation
```

注意即使是单核心的cpu也要考虑内存屏障，因为cm3cm4都有自己的icache dcache，而且是乱序执行（out of order execution）的

__DMB();确保没有乱序
__DSB();确保缓存刷新了

#### 为什么需要中断向量表重定位？我感觉应该第二种情况居多

![](https://i.imgur.com/x6KGL4u.png)

[data copy to sram](https://stackoverflow.com/questions/70633100/linker-copying-data-from-flash-memory-to-ram)

## 4. OS support feature

### 1.SVC异常

类似于riscv中的ecall指令，svc指令也可以作为引起syscall

```asm=
SVC #0x3 ; Call SVC function 3
```

* #### 为什么不使用普通的软件中断？
> Although it is possible to trigger an interrupt using software by writing to NVIC (e.g., Software Trigger Interrupt Register, NVIC->STIR), the behavior is a bit different: Interrupts are imprecise. It means that a number of instructions could be executed after setting the pending status but before the interrupt actually takes place. On the other hand, SVC is precise. The SVC handler must execute after the SVC instruction, except when another higher-priority exception arrives at the same time.

* #### 为什么使用PendSVC作为context swtch的中断？

和svc中断不一样的是PendSV不是准确的，PendSV有最低的优先级，这一点保证他在所有的中断执行完之后才会执行，否则中断的执行可能会被context swtch打断并延迟执行。

![](https://i.imgur.com/vApw1u1.png)

使用PendSV来完成context swtch的例子：由下图可见，sysytick虽然打断了中断的执行但是他只是置位PendSV之后立即返回中断继续执行，中断执行完毕之后才开始context swtch最大程度保证实时性

![](https://i.imgur.com/bCvpJgl.png)

**PendSV**还被用来实现中断的上半段和下半段，上半段是**time critical**十分紧要的操作在**time critical**的操作完成之后只要置位**PendSV**将下半段不是**time critical**就交给**PendSV**来完成

![](https://i.imgur.com/24LMFqV.png)



## 5. Baremetal示例

Linux环境下baremetal编程需要以下工具链，openocd充当调试器上位机程序

- 软件： arm-none-eabi-gcc，openocd，gdb-multiarch
- 硬件： CMSIS-DAP调试器，Air32F103

```bash
# 项目文件结构
├── linkscript.ld
├── main.c
├── Makefile
├── openocd.cfg
└── startup.c
```

用c语言完成启动文件的编写，我感觉主要有几点需要注意。

1. 作为简单的验证程序，其实不用將中断向量表完整写出，但是至少包含SP地址以及Reset_Handler
2. 如果选择从link脚本中读取SP的值，C语言语法的使用
3. link脚本中注意SRAM和Flash的实际大小，注意代码运行地址和放置地址
4. link脚本中使用ALIGN()对齐,段结束的symbol应该在ALIGN()后

```c
// statrup.c
extern unsigned long _estack;
extern int main();

void Reset_Handler();
void NMI_Handler();
void Hardfault_Handler();
void Memfault_Handler();
void Busfault_Handler();
void Usagefault_Handler();
void SVCall_Handler();
void DebugMon_Handler();
void PendSV_Handler();
void Systick_Handler();

__attribute__ ((section(".vectors"), used))
unsigned int vector_table[] = 
{ 
	(unsigned int)&_estack,
	Reset_Handler,
	NMI_Handler,
	Hardfault_Handler,
	Memfault_Handler,
	Busfault_Handler,
	Usagefault_Handler,
	0,
	0,
	0,
	0,
	SVCall_Handler,
	DebugMon_Handler,
	0,
	PendSV_Handler,
	Systick_Handler,
};

void Reset_Handler()
{
	for(int i=0; i<=5; i++)
	{
		;
	}
	main();
	while(1);
}
void NMI_Handler()
{
	while(1);
}
void Hardfault_Handler()
{
	while(1);
}
void Memfault_Handler()
{
	while(1);
}
void Busfault_Handler()
{
	while(1);
}
void Usagefault_Handler()
{
	while(1);
}
void SVCall_Handler()
{
	while(1);
}
void DebugMon_Handler()
{
	while(1);
}
void PendSV_Handler()
{
	while(1);
}
void Systick_Handler()
{
	while(1);
}

// main.c
int main()
{
        while(1);
}
```

```c
/* linkscript.ld */
ENTRY(Reset_Handler)

MEMORY {
  flash(rx)  : ORIGIN = 0x08000000, LENGTH = 128K
  sram(rwx) : ORIGIN = 0x20000000, LENGTH = 32K  /* remaining 64k in a separate address space */
}
_estack     = ORIGIN(sram) + LENGTH(sram);    /* stack points to end of SRAM */

SECTIONS {
  . = 0x08000000;
  .vectors :  
  { 
	  KEEP(*(.vectors)) 
  } > flash
  .text : 
  { 
	  . = ALIGN(4);
	  *(.text) 
	  . = ALIGN(4);
  } > flash
  .rodata : 
  { 
	  . = ALIGN(4);
	  *(.rodata) 
	  . = ALIGN(4);
  } > flash

  .data :
  {
	  . = ALIGN(4);
	  _sdata = .;   /* .data section start */
	  *(.data)
	  . = ALIGN(4);
	  _edata = .;  /* .data section end */
  } > sram AT > flash

  .bss : {
      . = ALIGN(4);
      _sbss = .;              /* .bss section start */
      *(.bss)
      *(COMMON)
	  . = ALIGN(4);
	  _ebss = .;              /* .bss section end */
  } > sram
}
```

使用Makefile来自动化编译：

```
all: firmware.bin

startup.o: startup.c
        arm-none-eabi-gcc -c -mcpu=cortex-m3 -g startup.c -o startup.o

main.o: main.c
        arm-none-eabi-gcc -c -mcpu=cortex-m3 -g main.c -o main.o

firmware.elf: startup.o main.o
        arm-none-eabi-ld -T linkscript.ld -o firmware.elf startup.o main.o

firmware.bin: firmware.elf
        arm-none-eabi-objcopy -O binary firmware.elf firmware.bin

dbg: firmware.bin
        qemu-system-arm -S -M stm32vldiscovery -kernel firmware.bin -gdb tcp::3333 -nographic -monitor telnet:127.0.0.1:1234,server,nowait

clean:
        rm -rf firmware.* startup.o main.o
```

在实际上板测试之前可以使用QEMU来进行仿真，如果仿真通不过就别上板了。（注意qemu支持的SRAM和Flash大小应该和实际使用的板子有区别，需要修改link脚本）

上板调试，gdb的一些默认重复的命令可以提前写入.gdbinit文件中，之后启动gdb可以自动执行。

```bash
# 在窗口1打开openocd
# openocd会自动读取该文件夹中的openocd.cfg
# openocd.cfg里面主要指定了调试器和硬件的配置文件
$ cat ./openocd.cfg 
source [find interface/cmsis-dap.cfg]
source [find target/stm32f1x.cfg]

# 可以从输出中看到已经和板子识别上，并且在3333端口监听gdb链接
$ openocd
Open On-Chip Debugger 0.12.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "swd". To override use 'transport select <transport>'.
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Using CMSIS-DAPv2 interface with VID:PID=0x0d28:0x0204, serial=000000800671ff525154887767033308a5a5a5a597969908
Info : CMSIS-DAP: SWD supported
Info : CMSIS-DAP: Atomic commands supported
Info : CMSIS-DAP: FW Version = 2.2.3R
Info : CMSIS-DAP: Serial# = C36E08E4C284
Info : CMSIS-DAP: Interface Initialised (SWD)
Info : SWCLK/TCK = 1 SWDIO/TMS = 1 TDI = 0 TDO = 0 nTRST = 0 nRESET = 1
Info : CMSIS-DAP: Interface ready
Info : clock speed 1000000 kHz
Info : SWD DPIDR 0x2ba01477
Info : [stm32f1x.cpu] Cortex-M3 r2p0 processor detected
Info : [stm32f1x.cpu] target has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f1x.cpu on 3333
Info : Listening on port 3333 for gdb connections

# 在窗口2打开gdb-multiarch并读入elf文件
# 在gdb中输入 target remote localhost:3333
# 输入 monitor reset init 进行复位
$ gdb-multiarch ./firmware.elf 
GNU gdb (Ubuntu 15.0.50.20240403-0ubuntu1) 15.0.50.20240403-git
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./firmware.elf...
(gdb) target remote localhost:3333
Remote debugging using localhost:3333
main () at main.c:3
3		while(1);
(gdb) monitor reset init
[stm32f1x.cpu] halted due to debug-request, current mode: Thread 
xPSR: 0x01000000 pc: 0x0800012c msp: 0x20008000
(gdb) si
0x0800012e in Reset_Handler () at startup.c:96
```



