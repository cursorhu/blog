# STM32--Firmware Architecture part2：业务代码分析--o2link FWs 

# o2link FWs的架构区别

o2link FWs指三类：

- o2link original FW(gen2): 用于老项目的对外发布版FW
- o2link JEV323 FW: 在o2link original FW上，针对JEV323做了功能改动和架构改动
- o2link Skyways FW: 在o2link original FW上，针对Skyways做了功能改动

## Bootloader和Firmware结构

### bootloader和Firmware在Flash的分布

1. o2link original FW和o2link Skyways FW是分为bootloader和Firmware两部分，两者共同构成烧录的bin文件

- bootloader：放在Flash的0x0800_0000 ~ 0x0x0800_8000空间，空间32KB；用作USB上位机烧录Firmware到Flash功能。

  ![image-20240517103655180](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171036206.png)

- Firmware: 放在Flash的0x0800_8000~ 0x0801_0000空间，空间32KB；用作处理USB上位机下发的各种控制、读写请求。

  ![image-20240517103700830](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171037852.png)
  
  bootloader和Firmware所有代码在Flash的分布如下：

![image-20240517111755697](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171117725.png)

注意：ROM/RAM空间分布对应到.sct的配置内容需要特别小心：

Bootloader的.sct:

```
LR_IROM1 0x08000000 0x00008000  {    ; load region size_region
  ER_IROM1 0x08000000 0x00008000  {  ; load address = execution address
   *.o (RESET, +First)
   *(InRoot$$Sections)
   .ANY (+RO)
   .ANY (+XO)
  }
  RW_IRAM1 0x200000C0 0x00003F30  {  ; RW data
   .ANY (+RW +ZI)
  }
}

```

Firmware的.sct: 

```
LR_IROM1 0x08008000 0x00008000  {    ; load region size_region
  ER_IROM1 0x08008000 0x00008000  {  ; load address = execution address
   *.o (RESET, +First)
   *(InRoot$$Sections)
   .ANY (+RO)
   .ANY (+XO)
  }
  RW_IRAM1 0x200000C0 0x00003F30  {  ; RW data
   .ANY (+RW +ZI)
  }
}

```

如果没有正确配置.sct, 例如把Firmware的.sct LR/ER起始地址配成0x0800_0000,后面用JLink烧录时就报错：No Algorithm for 0x80000000~0x....，Flash program fail. 因为JLink发现program的地址和.sct指定的LR/ER地址不一致。

2.o2link JEV323 FW是简化后的架构，只包含firmware部分，不支持USB上位机烧录FW bin：

- Firmware: 放在Flash的0x0800_0000~ 0x0801_0000空间，空间64KB；用作处理USB上位机下发的各种控制、读写请求。

  ![image-20240517103730129](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171037152.png)

3.RAM空间的分布

上面bootloader+FW架构的Keil配置中，bootloader和FW的RAM空间都是从0xC0偏移开始的，不是从RAM的0x00，

而单FW架构，FW是从RAM的0开始。

原因是bootloader跳转执行FW时，需要从Flash拷贝中断向量表192bytes(0xC0)到RAM起始地址，所以FW代码的RAM数据区不划分这块空间。详见direct_jump_to_app()

### IAP和ICP的概念

为什么有两种代码结构分布？涉及到以下两种烧录Firmware的方式：参考STM32 RM0091文档

• IAP (in-application programming): IAP is the ability to re-program the flash memory of a microcontroller while the user program is running.

• ICP (in-circuit programming): ICP is the ability to program the flash memory of amicrocontroller using the JTAG protocol, the SWD protocol or the bootloader while thedevice is mounted on the user application board.

o2link作为成熟的产品，需要支持用户侧烧录firmware(IAP)，因此开发了USB接口的IAP烧录功能，这部分划分为bootloader。

> An important requirement for most Flash-memory-based systems is the ability to update firmware when installed in the end product. This ability is referred to as in-application programming (IAP).
>
> The IAP code uses the USB to:
>
> ● Download a binary file from the USB HID to the STM32F0xx's internal Flash memory.
>
> ● Upload the STM32F0xx's internal Flash memory content (starting from the defined user 
>
> application address) into a binary file.
>
> ● Execute the user program.

（实质上这不是真正意义的bootloader，仅仅是firmware update功能；如果firmware代码在SRAM运行，这部分功能完全可以做到Firmware代码中去，不用占用32KB空间）

jev323 firmware目前是内部测试用，因此不需要IAP，用Jlink的ICP方式烧录。全部Flash空间(64KB)可用于业务流程。

### bootloader和firmware的执行流程

参考o2link spec:

- bootloader基本逻辑是：每次上电RESET时，先执行bootloader判断当前Flash的firmware区域（app）有没有valid FW能执行？如果有，就跳转firmware的main去执行；如果没有，bootloader启动IAP流程，响应USB上位机的erase flash、program firmware的指令，完成以后再跳转执行firmware指令；

- firmware在执行时，如果收到USB上位机的IAP命令(USB_IAP_JUMP_TO_BOOT)，就是要跳转到bootloader，准备IAP去下载新的firmware bin；其他情况不会跳转到bootloader。

![image-20240517105536135](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171055206.png)

## bootloader代码分析（重要+难点）

整个bootloader代码和Firmware流程都是main初始化+While1轮询USB请求的结构，区别在于执行流程。

### bootloader校验FW

bootloader的main初始化系统时钟后，就立即check FW是否valid：

```
int main(void)
{
	HAL_Init();
	SystemClock_Config();
	//判断FW是否valid
	check_if_jump_to_app();
```

```
check_if_jump_to_app():

    if(*(uint32_t *)RAM_FROM_APP_FLAG_ADDR != RAM_FROM_APP_FLAG_DATA){
            *(uint32_t *)RAM_FROM_APP_FLAG_ADDR = 0;
            if((*(uint32_t *)APP_FILE_EDN_ADDR == APP_FILE_END_DATA) && 
                ( *(uint32_t *)APP_FILE_START_ADDR == APP_FILE_START_DATA) &&
                (pin_state == GPIO_PIN_SET)) //PB4
                direct_jump_to_app();
	}
```

校验FW有效包括三个条件都要满足：

1. 查看RAM 0x3f3C位置(0x20003f3C)的DWORD是否为RAM_FROM_APP_FLAG_DATA(0x6a756d70)，然后清0。这个flag是USB上位机下发USB_IAP_JUMP_TO_BOOT时调用jump_to_boot()设置的，这个USB请求在bootloader或FW阶段都可能被发起。

​      目的：确认是上位机发起的jump to boot，而不是其他原因比如CPU异常reset进入的boot。

2. 查看Flash的FW区域（0x0800_8000开始）的开始（0x08008014）和尾部区域（0x0800fffc）的两个DWORD是否分别为0x00617070和0x00656e64。

   目的：确认Flash的FW是valid，确认尾部是确保数据完整

3. 查看PB4 pin是否为高。

   目的：根据原例图，可能是防止和one-wire功能冲突？待确认

   ![image-20240517113801021](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171138048.png)

### bootloader跳转到FW代码的过程

   ```
   direct_jump_to_app():
   
   #define  APPLICATION_ADDRESS   (0x08000000 + 0x8000) //FW在Flash的起始地址 
   
   void direct_jump_to_app(void)
   {
   	int i;
   	uint32_t JumpAddress;
   	pFunction Jump_To_Application;
   	
   	__disable_irq(); 
   	
   	//拷贝Firmwware的192bytes的中断向量表到SRAM
   	for(i = 0; i < 48; i++)
   	{
   		*((uint32_t*)(0x20000000 + (i << 2)))=*(__IO uint32_t*)(APPLICATION_ADDRESS + (i<<2));
   	}
   	
   	 /* Test if user code is programmed starting from address "APPLICATION_ADDRESS" */
   	 //判断栈指针是否位于SRAM
   	if (((*(__IO uint32_t*)APPLICATION_ADDRESS) & 0x2FFE0000 ) == 0x20000000)
       { /* Jump to user application */
         
         	//设置函数指针，跳转到Firmware的RESET入口
   		JumpAddress = *(__IO uint32_t*) (APPLICATION_ADDRESS + 4);
   		Jump_To_Application = (pFunction) JumpAddress;
   		
   		/* Initialize user application's Stack Pointer */
   		__set_MSP(*(__IO uint32_t*) APPLICATION_ADDRESS);
   		
   		//执行跳转
   		Jump_To_Application();
   	}  
   	else
   	{
   		__enable_irq();
   		return ;
   	}
   }
   ```

**（1）拷贝Firmwware的192bytes的中断向量表到SRAM**

**Q1：为什么要拷贝？中断向量表放在Flash中不能执行吗？**

Cortex M0的限制：Flash的中断向量表一定要放在Flash开始的地方，不能relocation到Flash的其他偏移地址，参考Reference Manual RM0091：

![image-20240517172328661](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171723698.png)

Firmware中断向量表是放在Flash的32KB offset的地方，是不能被硬件使用的；

RM0091给出Cortex M0对此问题的方案：将中断向量表拷贝到SRAM的0地址，再设置SYSCFG register，remap SRAM空间作为CPU 0地址。这样CPU异常、中断发生时，就能进入SRAM的中断向量表。

**Q2：为什么只需要拷贝中断向量表的192bytes，而不是拷贝整个Firmware的32KB？SRAM空间都remap为CPU 0地址了，Flash中的Firmware代码不拷贝到SRAM还能执行到吗？**

这里要分析MCU的PC指针取指令的流程：

1. 在bootloader开始阶段，PC指针取指令都是在Flash 起始地址~32KB之间取bootloader指令执行
2. bootloader拷贝FW中断向量表到SRAM的0地址，并设置CPU memory空间为SRAM空间 (注释1)
3. bootloader跳转，注意看上面代码，跳转到Flash的Firmware空间(Flash 32KB~64KB)的Firmware入口，也就是说，PC指针还是从Flash取指令，只不过指令是Firmware的main
4. Firmware执行main初始化和while1，PC指针始终在while1中转圈
5. 如果中断或者异常发生，硬件跳转到SRAM的Firmware中断向量表，取中断回调指令执行，这个中断回调指令还是在Flash的Firmware空间(Flash 32KB~64KB)，中断返回后，PC指针恢复之前在Firmware while1里的位置。

根据以上分析，PC指针仅仅在中断发生时需要用跳到SRAM的中断向量表，其他时间都在Flash的Firmware区域取指令，所有Firmware代码都能被执行到。因此SRAM remap不影响Flash的代码执行，不需要拷贝Firmware代码到SRAM (要拷贝FW到SRAM以提高执行速度也行，要改Firmware编译的基地址为SRAM)。

（注释1）CPU remap实际在main才设置（但应该在bootloader里设置），代码如下：

```
__HAL_REMAPMEMORY_SRAM();

#define __HAL_REMAPMEMORY_SRAM                __HAL_SYSCFG_REMAPMEMORY_SRAM
#define __HAL_SYSCFG_REMAPMEMORY_SRAM()         do {SYSCFG->CFGR1 &= ~(SYSCFG_CFGR1_MEM_MODE); \
                                             SYSCFG->CFGR1 |= (SYSCFG_CFGR1_MEM_MODE_0 | SYSCFG_CFGR1_MEM_MODE_1); \
                                            }while(0) 
```

**关于CPU空间的remapping，有两个概念需清楚：**

1. CPU空间remap到SRAM还是Flash，并不影响CPU对Flash和SRAM的访问；

   不管谁被remap为CPU memory空间，pc取指令都可以用0x0800_0000 + offset访问Flash，0x2000_0000 + offset访问SRAM

   ![image-20240517174752681](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171747713.png)

2. CPU remap只影响”MCU的0地址在哪个设备空间“，和启动位置相关；

   SYSCFG register的CPU memory mapping定义：

   ![image-20240517174553060](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171745110.png)

   注意该SYSCFG register配置会被reset，即reset启动后的CPU space是BOOT0 pin和nBOOT1 register共同决定的：

   ![image-20240517175619616](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171756653.png)

**Q3：Firmware和bootloader的中断向量表的指令应该差不多，为什么不能公用一套中断向量表？**

这个问题涉及到编译和链接：bootloader和Firmware的中断向量表的指令还是有区别，因为中断回调不同，导致必须要分两套中断向量表；

两套中断向量表编译出的基础地址不一样：如下图bootloader中断向量表指令都是基于0x0800_8000，FW的都是0x0800_0000。这个基础地址是.sct链接文件指定。

![image-20240517185911311](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405171859345.png)

**（2）跳转到Firmwware指令**

跳转的条件判断是个难点：为什么要判断FW代码的首个DWORD的值是否在SRAM空间？

```
if (((*(__IO uint32_t*)APPLICATION_ADDRESS) & 0x2FFE0000 ) == 0x20000000)
```

FW代码的首个DWORD的值是什么：

参考FW的startup.s：是__initial_sp符号，找不到具体指令

```
__Vectors       DCD     __initial_sp                   ; Top of Stack
                DCD     Reset_Handler                  ; Reset Handler
```

__initial_sp符号符号是什么：

FW的startup.s只能找到声明：

```
Stack_Size		EQU     0x500
                AREA    STACK, NOINIT, READWRITE, ALIGN=3
Stack_Mem       SPACE   Stack_Size
__initial_sp
```

对此代码的解释：

__initial_sp is a label which takes the origin (ORG) value of the assembler after it allocates the space. Look at a .LST or .MAP file.

参考：https://community.st.com/t5/stm32-mcus-products/about-initial-sp/td-p/551812

基于此解释，查看FW的.map，找到symbol的分布：

最后一个Data symbol是uwTick，尾部地址是0x20003514 + 4 = 0x20003518；

__initial_sp符号的起始地址正好是0x20003518 + 0x500（startup.s指定的Stack_Size），因此验证了以上解释。

```
Global Symbols

    Symbol Name                              Value     Ov Type        Size  Object(Section)
	......
	
    huart1                                   0x20001a3c   Data         132  usart.o(.bss.huart1)
    huart2                                   0x20001ac0   Data         132  usart.o(.bss.huart2)
    huart3                                   0x20001b44   Data         132  usart.o(.bss.huart3)
    one_wire_data                            0x20001fcc   Data         152  one_wire.o(.bss.one_wire_data)
    uart_rx_fifo_buf                         0x20002064   Data        5120  main.o(.bss.uart_rx_fifo_buf)
    uwTick                                   0x20003514   Data           4  stm32f0xx_hal.o(.bss.uwTick)
    __initial_sp                             0x20003a18   Data           0  startup_stm32f072xb.o(STACK)
```

基于以上，__initial_sp 是编译器自动形成的值，作为RAM中的栈顶位置。

bootloader设置Stack_Size为0x500，编译器就在RAM中把所有全局变量排列完后，在加0x500作为栈空间，也就是说这个值最后是取决于代码数据占的RAM空间的，并不是固定的RAM最尾部的地址。

注：Stack_Size值应该根据.map情况，设置成和RAM可用栈空间接近，不然RAM空间没充分利用，形成爆栈。

所以FW的第一个指令保存了RAM中的栈顶（栈起始地址），第二个指令才是RESET。

前面代码是bootloader对__initial_sp 判断是否在RAM空间，因为跳转时要设置栈指针的安全性判断：

```
/* Initialize user application's Stack Pointer */
__set_MSP(*(__IO uint32_t*) APPLICATION_ADDRESS);
```

**（3）Jump_To_Application函数指针**

这里不详细分析。STM32 bootloader跳转FW有模板代码，参考原厂固件库代码。

## Flash烧录问题（重要）

### 用Keil的JLink烧录Flash

Keil内置安装JLink，Keil烧录.bin到开发板的Flash，实际是调用内置的JLink烧录。

对于Bootloader和Firmware，需要正确配置烧录区域：

-  Address Range： .bin文件烧录到Flash的区域(一般是Flash空间)；这个区域应该和Keil项目配置的ROM区域一致
- Erase Sectors：只擦除选中的Flash Address Range的sectors
- RAM for Algorithm：这个跟烧录的.bin运行时RAM没关系，是指烧录程序本身要占用的RAM，参考：https://www.keil.com/support/man/docs/ulinkme/ulinkme_su_ram_for_algorithm.htm

o2link的bootloader：

![image-20240520104651534](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201046576.png)

o2link的firmware：

![image-20240520104700405](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201047437.png)

### 如何确认Flash正确烧录

参考： [J-Flash读取STM32内部程序，导出Hex/Bin文件](https://blog.csdn.net/lnfiniteloop/article/details/134575496?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-134575496-blog-118443669.235%5Ev43%5Epc_blog_bottom_relevance_base4&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-134575496-blog-118443669.235%5Ev43%5Epc_blog_bottom_relevance_base4&utm_relevant_index=2)

JLink安装，需要安装包里的USB驱动：SEGGER\JLink_V796e\USBDriver\x64\dpinst_x64.exe

使用JLink读Flash并比较：

1. JLink: Target -> Connect
2. 读Flash(一般Range或者Entire chip)

![image-20240520111220863](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201112900.png)

![image-20240520111250598](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201112614.png)

3. 保存数据到.bin

![image-20240520111256831](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201112851.png)

4. 比较bootloader.bin和从Flash读出的数据.bin是否一致：

使用Winmerge比较二进制文件：

左侧bootloader.bin，右侧Flash读出的bootloader；

![image-20240520110440432](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201104475.png)

可见bootloader真实数据约0x7524 bytes；Flash擦除整个bootloader区域0~0x8000, 所以Flash读的后部分数据为0xFF。

Firmware区域比较同理，JLink的Flash读出区域改成0x08008000~0x08010000

## 特殊的编译和代码修改记录

编译问题：

1. Firmware编译无法输出.bin文件但Keil没报错，输出了ER$$.ARM.__at_0x0800fffc文件

   ![image-20240520115554066](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405201155089.png)

   原因：main定义了以下section，但链接器找不到这个符号，所以生成bin时报error

   ```
   const uint32_t file_end __attribute__((section(".ARM.__at_0x0800fffc"))) = 0x00656e64;
   ```

   目前没找到根本性的解决办法；因为代码没用到这个file_end，所以注释掉这个定义。这个定义地址本身是合理的，是Firmware的Flash区域的最后一个DWORD。

   参考：

   [Methods of placing functions and data at specific addresses](https://developer.arm.com/documentation/dui0474/m/scatter-loading-features/root-execution-regions/methods-of-placing-functions-and-data-at-specific-addresses?lang=en)

   [ARM asmlink User Guide](https://documentation-service.arm.com/static/63eb51fc9567172d4e2aa918)

代码问题：

bootloader+Firmware只支持用USB上位机更新Firmware，不支持JLink烧录Firmware，因为bootloader校验Dword不通过；所以需要修改bootloader代码：

```
void check_if_jump_to_app(void):

#ifdef SKYWAYS_TEST
	direct_jump_to_app(); //这里直接跳转，不校验是USB上位机发起的跳转
#else
	if (*(uint32_t *)RAM_FROM_APP_FLAG_ADDR != RAM_FROM_APP_FLAG_DATA)
	{
		*(uint32_t *)RAM_FROM_APP_FLAG_ADDR = 0;
		if ((*(uint32_t *)APP_FILE_EDN_ADDR == APP_FILE_END_DATA) &&
			(*(uint32_t *)APP_FILE_START_ADDR == APP_FILE_START_DATA) &&
			(pin_state == GPIO_PIN_SET))
			direct_jump_to_app();
	}
#endif
```

# Skyways业务代码分析

在《STM32--Firmware Architecture part1：开发环境和HAL API应用》中已经分析了整体的Firmware-USB上位机之间的请求处理流程，这里针对Skyways Firmware具体分析业务流程的差异点。

## UART

### USB下发数据给UART（TX, no buffer）

调用过程：

```
CUSTOM_HID_OutEvent_FS -> write_uart_function() -> HAL_UART_Transmit()
```

Skyways版本的UART TX代码有几点需要注意：

1. usb_send_buf[0] |= 0x80;表示错误，用于通知USB上位机。Tx一次发送超过60bytes, 或者HAL_UART_Transmit有Timeout，则上报USB上位机有错。
2. 以下代码的UART返回数据没发送给USB，和o2link Spec不一致：UART没有返回USB：0101+buffer data.

```
void write_uart_function()
{
	uint8_t ret = 0;
	uint32_t i;
	length = usb_send_buf[2] << 8 | usb_send_buf[3];
	if(length > MAX_UART_WRITE_LENGTH){ //60bytes
		usb_send_buf[0] |= 0x80;
		usb_send_buf[2] = UART_PARAMETER_ERROR;
		usb_send(usb_send_buf,USB_TIMEOUT_TIME);
		return;
	}
	
	cdc_receive_flag = CDC_FLAG_HID;
	ret = HAL_UART_Transmit(&huart2,(uint8_t *)&usb_send_buf[4],length,UART_TIMEOUT_TIME);

	if(ret)
	{
		usb_send_buf[0] |= 0x80;
		usb_send_buf[2] = ret;
		usb_send(usb_send_buf,USB_TIMEOUT_TIME);
	}
	//usb_send(usb_send_buf,USB_TIMEOUT_TIME); //这里和o2link Spec不一致，UART没有返回USB：0101+buffer.
}
```

### USB从UART接收数据（RX, 1KB buffer, DMA）

代码流程在《STM32--Firmware Architecture part1：开发环境和HAL API应用》的"UART2部分"有详细分析。

应用上的结论：UART2 DMA使用UART IDLE frame作为传输完成中断的触发源，只要应用上保证一次UART读数据中没有异常的IDLE frame，则UART2 DMA IDLE frame产生的完成中断可作为一次完整的UART数据传输结束标志。

## SPI

Skyways的SPI data transmission底层操作在的"8.1 usb_to_spi"有详细描述，这里看到以下区别：

- 发起spi数据传输之前，Deinit了I2C，把I2C的SDA/SCL两个pin作为GPIO输入模式拉高。
- 完成spi数据传输之后，重新init了I2C到100K速度.

```
void usb_handle_process(void):

case USB_TO_SKYWAY_SPI_WRITE:
		HAL_I2C_DeInit(&hi2c1);
		i2c_gpio_fun();
		usb_to_spi_convert_skyway_write();
		MX_I2C1_Init(100000, 7);
		break;
case USB_TO_SKYWAY_SPI_READ:
		HAL_I2C_DeInit(&hi2c1);
		i2c_gpio_fun();
		usb_to_spi_convert_skyway_read();
		MX_I2C1_Init(100000, 7);
		break;
```

根据Skyways和MCU的连接，SPI和I2C并没有复用；Skyways和MCU的SPI通信也没要求对I2C的pin做什么特殊操作（测试SPI read、write甚至都没连接I2C），因此猜测此处代码只是早期开发时，预防I2C和SPI同时使用时有冲突，实际没这个需求。 -- 下个版本删除此I2C代码，测试SPI read、write.

## CAN

TODO

## one-wire

TODO
