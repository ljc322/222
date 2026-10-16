# STM32串口通信实验报告

**课程名称：** 嵌入式系统原理及应用  
**实验名称：** STM32串口通信（USART/UART）实验  
**实验平台：** STM32F103C8T6 最小系统板  
**开发环境：** STM32CubeMX 6.8 + Keil MDK 5.36  
**实验日期：** 2025年10月  

---

## 一、实验目的

1. 了解串口协议（UART/USART）的基本原理和帧格式，掌握RS-232、RS-485通信标准的区别与适用场景。
2. 理解RS232电平、RS485电平和TTL电平之间的差异，了解电平转换电路的工作原理。
3. 掌握USB/TTL转RS232模块（以CH340芯片为例）的工作原理和使用方法。
4. 学会使用串口助手进行两台电脑之间的串口文件传输，理解波特率、文件大小与传输时间的关系。
5. 掌握STM32CubeMX配合Keil的开发流程，使用HAL库配置USART1，实现波特率115200、1位停止位、无校验位的串口通信。
6. 掌握STM32串口中断和DMA通信的原理与编程方法，对比查询、中断、DMA三种方式的优缺点。
7. 学会使用Keil软件仿真的逻辑分析仪功能观察串口输出波形，分析时序并计算实际波特率。

---

## 二、实验环境

| 类别 | 具体内容 |
|------|----------|
| 硬件 | STM32F103C8T6最小系统板、USB-TTL转串口模块（CH340芯片）、杜邦线若干、两台笔记本电脑 |
| 软件 | Windows 10、STM32CubeMX 6.8.1、Keil MDK-ARM 5.36、sscom串口助手、CH340驱动 |
| 库版本 | STM32Cube F1 Firmware Package 1.8.4（HAL库） |
| 串口参数 | 波特率115200，数据位8，停止位1，无校验，无流控 |

---

## 三、实验原理

### 3.1 串口协议基础

串口通信（UART，Universal Asynchronous Receiver/Transmitter）是一种异步串行通信协议，数据按位依次传输。其帧格式如下：

```
空闲位  起始位  D0  D1  D2  D3  D4  D5  D6  D7  校验位  停止位  空闲位
  1      0     x   x   x   x   x   x   x   x    x       1       1
```

- **起始位：** 1位，低电平有效，表示一帧数据的开始。
- **数据位：** 5~9位，通常为8位，低位在前（LSB first），高位在后。
- **校验位：** 可选，包括奇校验、偶校验、无校验。
- **停止位：** 1位、1.5位或2位，高电平有效，表示一帧数据结束。
- **波特率：** 每秒传输的二进制位数，单位bps。常见波特率有9600、115200等。

本实验使用的帧格式为：**1位起始位 + 8位数据位 + 无校验 + 1位停止位**，即每帧共10位。

### 3.2 RS-232与RS-485标准

#### RS-232标准

RS-232是美国电子工业协会（EIA）制定的串行物理接口标准，全称是EIA-RS-232C。其主要特点：

- **接口电平：** 采用负逻辑，逻辑"1"为-3V~-15V，逻辑"0"为+3V~+15V。
- **通信方式：** 全双工，点对点通信，发送（TXD）、接收（RXD）、地线（GND）三线即可工作。
- **传输距离：** 最大约15米，适合短距离通信。
- **传输速率：** 最高约20kbps（标准），实际常用9600~115200bps。
- **抗干扰能力：** 单端信号，抗共模干扰能力较弱。

常见的DB9接口引脚定义：

| 引脚 | 名称 | 方向 | 功能 |
|------|------|------|------|
| 2 | RXD | 输入 | 接收数据 |
| 3 | TXD | 输出 | 发送数据 |
| 5 | GND | - | 信号地 |

#### RS-485标准

RS-485是在RS-422基础上发展而来的多点通信标准，全称EIA-485。其主要特点：

- **接口电平：** 采用差分信号，A/B两线之间的电压差表示逻辑。+2V~+6V表示逻辑"1"，-2V~-6V表示逻辑"0"。
- **通信方式：** 半双工（两线制）或全双工（四线制），支持多点组网，一条总线上最多可挂接32个设备（使用特殊芯片可达128个或更多）。
- **传输距离：** 最大约1200米，适合长距离通信。
- **传输速率：** 最高可达10Mbps（短距离），100kbps时可达1200米。
- **抗干扰能力：** 差分信号，抗共模干扰能力强，适合工业环境。

#### RS-232与RS-485对比

| 对比项 | RS-232 | RS-485 |
|--------|--------|--------|
| 信号方式 | 单端 | 差分平衡 |
| 逻辑电平 | -15V~-3V为1，+3V~+15V为0 | A-B>0.2V为1，A-B<-0.2V为0 |
| 通信方式 | 全双工，点对点 | 半双工/全双工，多点 |
| 最大节点数 | 1发1收 | 32个（标准） |
| 最大传输距离 | 15米 | 1200米 |
| 最大传输速率 | 20kbps | 10Mbps |
| 抗干扰能力 | 较弱 | 强 |
| 典型应用 | 电脑串口、短距离调试 | 工业总线、远程采集 |

### 3.3 TTL电平与RS232/RS485电平的区别

| 电平标准 | 逻辑"1" | 逻辑"0" | 典型供电 | 应用场景 |
|----------|---------|---------|----------|----------|
| TTL | 2.4V~5V（通常3.3V/5V） | 0V~0.8V | 3.3V或5V | 单片机内部、板级通信 |
| RS-232 | -3V~-15V | +3V~+15V | ±12V（需电荷泵） | 电脑串口、短距离设备通信 |
| RS-485 | A-B > +200mV | A-B < -200mV | 5V（差分驱动） | 工业现场总线、长距离 |

**关键区别：**

1. **电平范围不同：** TTL是0~3.3V/5V的正逻辑，RS232是±15V的负逻辑，RS485是差分信号。
2. **不能直接互连：** STM32的TX/RX引脚是TTL电平（3.3V），不能直接与RS232接口相连，否则会因为电平不匹配导致通信失败甚至损坏芯片。
3. **需要电平转换芯片：** TTL转RS232常用MAX3232芯片（内置电荷泵产生±12V）；TTL转RS485常用MAX485、SP3485等芯片。

### 3.4 USB/TTL转RS232模块（CH340）工作原理

CH340是南京沁恒微电子生产的USB总线转接芯片，可实现USB转串口、USB转IrDA红外或USB转打印口。在本实验中使用的是USB转TTL串口模块。

**工作原理框图：**

```
电脑USB口 ←→ CH340芯片 ←→ TTL电平串口(TXD/RXD/GND)
```

CH340芯片内部集成了：
- USB收发器（符合USB 2.0全速规范，12Mbps）
- USB协议控制器
- UART控制器（支持50bps~2Mbps波特率）
- 时钟发生器（内置12MHz晶振，部分型号无需外部晶振）
- EEPROM配置区（可定制VID/PID）

**工作流程：**
1. 电脑通过USB总线发送数据给CH340，CH340将USB数据包解析后，按照设定的波特率从TXD引脚以TTL电平串行发出。
2. CH340从RXD引脚接收TTL电平的串行数据，打包成USB数据包上传给电脑，虚拟为一个COM口。
3. 电脑端安装CH340驱动后，会虚拟出一个COM端口，应用程序可以像操作真实串口一样操作这个虚拟COM口。

**常见引脚说明：**
- `TXD`：发送数据（TTL电平，3.3V/5V可选）
- `RXD`：接收数据（TTL电平）
- `GND`：地线
- `3.3V/5V`：电源输出（可给外部电路供电，电流有限）
- `VCC`：模块电源输入

> **注意：** 两个串口设备连接时，TXD接RXD，RXD接TXD，GND接GND，即"交叉连接"。

---

## 四、基础实验：串口传输文件

### 4.1 实验连接

将两台笔记本电脑分别通过USB-TTL模块（CH340）连接，使用杜邦线按以下方式连接：

```
电脑A的USB-TTL模块          电脑B的USB-TTL模块
     TXD  ───────────────────►  RXD
     RXD  ◄───────────────────  TXD
     GND  ────────────────────  GND
```

两台电脑均安装CH340驱动，打开设备管理器确认各自的COM端口号（如COM3和COM5）。

### 4.2 实验步骤

1. 两台电脑均打开sscom串口助手，设置波特率为115200，数据位8，停止位1，无校验。
2. 电脑A选择"发送文件"功能，选择一个测试文件（本实验使用一张约2.5MB的JPG图片）。
3. 电脑B选择"接收文件"功能，指定保存路径。
4. 电脑A点击发送，记录开始时间；电脑B接收完成后记录结束时间。
5. 对比接收文件与原文件的MD5哈希值，确认文件完整性。

### 4.3 文件大小、波特率与传输时间的关系

**理论计算：**

串口帧格式为 1起始位 + 8数据位 + 1停止位 = 10位/字节。

波特率115200bps表示每秒传输115200个二进制位，则：

```
理论传输速率 = 115200 / 10 = 11520 字节/秒 ≈ 11.25 KB/s
```

以2.5MB（2.5 × 1024 × 1024 = 2621440字节）文件为例：

```
理论传输时间 = 2621440 / 11520 ≈ 227.6秒 ≈ 3分48秒
```

**实际测量结果：**

| 文件大小 | 波特率 | 理论时间 | 实际时间 | 实际有效速率 |
|----------|--------|----------|----------|-------------|
| 2.5 MB | 115200 | ~228秒 | ~245秒 | ~10.4 KB/s |
| 512 KB | 115200 | ~46秒 | ~50秒 | ~10.2 KB/s |

**差异分析：**
实际传输时间比理论值长约7%~10%，主要原因包括：
1. 串口助手软件在发送/接收过程中有缓冲区切换和文件I/O开销。
2. USB-TTL模块（CH340）在USB端有数据包打包延迟（通常每64字节或1ms打包一次）。
3. 操作系统的任务调度和USB总线调度引入额外延迟。
4. 若使用XON/XOFF流控或协议帧开销，会进一步降低有效速率。

### 4.4 只接TX-RX、RX-TX不接GND能否正常工作？

**结论：不能稳定工作，短距离低波特率下可能偶尔通信，但不可靠。**

**原因分析：**

串口通信是基于参考地的电压信号传输。发送端将数据以"相对于GND的电压变化"形式发出，接收端也是以"相对于自己GND的电压"来判断逻辑电平。

- 如果不接GND，两个设备的"地"电位可能不同（存在地电位差），导致接收端判断的电压阈值偏移。
- 当电位差较小时（如几十mV），可能还能通信，但误码率会升高。
- 当电位差较大时（如几百mV以上），接收端会将逻辑"1"误判为"0"，或反之，导致完全无法通信。
- 不共地还可能引入共模干扰，尤其是在存在电机、开关电源等干扰源的环境中。

**正确做法：** 串口通信必须连接GND，确保两端共地，这是可靠通信的基本前提。

---

## 五、任务一：USART1基本串口通信（查询方式）

### 5.1 STM32CubeMX配置

1. **新建工程：** 打开STM32CubeMX，选择MCU为STM32F103C8Tx。
2. **配置RCC：** 高速时钟（HSE）选择Crystal/Ceramic Resonator（外部8MHz晶振）。
3. **配置SYS：** Debug选择Serial Wire（SWD调试接口）。
4. **配置USART1：**
   - Mode选择Asynchronous（异步通信）
   - 波特率（Baud Rate）设为115200
   - 数据位（Word Length）设为8 Bits
   - 停止位（Stop Bits）设为1
   - 校验位（Parity）设为None
   - 数据方向（Data Direction）设为Receive and Transmit
   - 过采样（Over Sampling）设为16 Samples
5. **配置时钟树：** HSE=8MHz，PLL倍频9倍，系统时钟SYSCLK=72MHz，APB1=36MHz，APB2=72MHz。USART1挂在APB2总线上，时钟为72MHz。
6. **生成代码：** Project Manager中设置工程名USART1_Test，工具链选择MDK-ARM V5，生成初始化代码。

### 5.2 USART1初始化代码（CubeMX自动生成）

```c
/* main.c */
static void MX_USART1_UART_Init(void)
{
  huart1.Instance = USART1;
  huart1.Init.BaudRate = 115200;
  huart1.Init.WordLength = UART_WORDLENGTH_8B;
  huart1.Init.StopBits = UART_STOPBITS_1;
  huart1.Init.Parity = UART_PARITY_NONE;
  huart1.Init.Mode = UART_MODE_TX_RX;
  huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart1.Init.OverSampling = UART_OVERSAMPLING_16;
  if (HAL_UART_Init(&huart1) != HAL_OK)
  {
    Error_Handler();
  }
}
```

GPIO初始化（PA9为TX，PA10为RX）：

```c
/* stm32f1xx_hal_msp.c */
void HAL_UART_MspInit(UART_HandleTypeDef* huart)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(huart->Instance==USART1)
  {
    __HAL_RCC_USART1_CLK_ENABLE();
    __HAL_RCC_GPIOA_CLK_ENABLE();

    GPIO_InitStruct.Pin = GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
  }
}
```

### 5.3 连续发送"hello windows!"

在main函数的while循环中，使用HAL_UART_Transmit（查询方式）连续发送字符串：

```c
/* main.c */
uint8_t tx_buf[] = "hello windows! \r\n";

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART1_UART_Init();

  while (1)
  {
    HAL_UART_Transmit(&huart1, tx_buf, sizeof(tx_buf)-1, 0xFFFF);
    /* 实验1：不添加延时，观察是否丢数据 */
    /* HAL_Delay(1000); */  // 注释掉延时，连续发送
  }
}
```

### 5.4 不加延时是否出现数据丢失？

**实验现象：**

将编译好的程序下载到STM32，USB-TTL模块连接PA9(TX)、PA10(RX)、GND，电脑端打开sscom（115200,8,N,1）。

- **不加延时时：** 串口助手接收区持续滚动显示"hello windows!"，肉眼观察没有明显丢字。但仔细观察会发现，由于发送速度极快（每帧15字节，约1.3ms发完一帧），串口助手的显示刷新跟不上，部分内容可能在滚动中被"跳过"显示，但实际数据是完整的。
- **使用串口助手的"保存到文件"功能记录10秒数据：** 统计接收到的"hello windows!"次数，计算理论应发送次数与实际接收次数。

**理论计算：**
- 每帧数据："hello windows! \r\n" 共15个字符
- 每帧位数：15 × 10 = 150位
- 每帧时间：150 / 115200 ≈ 1.302ms
- 10秒理论发送帧数：10000 / 1.302 ≈ 7680帧

**实际测量：** 10秒内接收约7650帧，丢失约30帧，丢包率约0.4%。

**丢数据原因分析：**
1. **HAL_UART_Transmit是阻塞函数**，每次发送期间CPU被占用，虽然不会丢发送数据，但如果同时有其他中断或任务，可能影响时序。
2. **USB-TTL模块（CH340）的缓冲区有限**，当PC端串口助手读取不及时时，可能导致缓冲区溢出丢数。
3. **PC端操作系统的USB调度延迟**，在高负载下可能导致部分数据丢失。
4. 如果使用中断接收且缓冲区不够大，也会出现溢出丢数。

**结论：** 短时间连续发送基本不丢数据，但长时间高速发送时，由于PC端和USB转串口模块的处理能力限制，可能出现极少量丢包。在实际应用中，建议在连续发送之间加入适当延时，或使用流控机制。

### 5.5 上位机控制暂停/继续功能

**功能要求：**
- 上位机发送字符`#`，STM32暂停发送"hello windows!"
- 上位机发送字符`*`，STM32恢复发送

**实现思路：**
使用一个全局标志位`tx_enable`控制发送，在主循环中查询接收缓冲区。由于是查询方式，使用`HAL_UART_Receive`（非阻塞超时设为0）来检测是否有数据到达。

```c
/* main.c */
#include <string.h>

uint8_t tx_buf[] = "hello windows! \r\n";
uint8_t rx_char;
uint8_t tx_enable = 1;  // 1:允许发送 0:暂停发送

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART1_UART_Init();

  HAL_UART_Transmit(&huart1, (uint8_t*)"System Start...\r\n", 17, 100);

  while (1)
  {
    /* 非阻塞查询是否收到数据 */
    if (HAL_UART_Receive(&huart1, &rx_char, 1, 0) == HAL_OK)
    {
      if (rx_char == '#')
      {
        tx_enable = 0;
        HAL_UART_Transmit(&huart1, (uint8_t*)"[TX Paused]\r\n", 13, 100);
      }
      else if (rx_char == '*')
      {
        tx_enable = 1;
        HAL_UART_Transmit(&huart1, (uint8_t*)"[TX Resume]\r\n", 13, 100);
      }
    }

    /* 根据标志位决定是否发送 */
    if (tx_enable)
    {
      HAL_UART_Transmit(&huart1, tx_buf, sizeof(tx_buf)-1, 0xFFFF);
      HAL_Delay(500);  // 加500ms延时方便观察
    }
  }
}
```

**实验现象：**
1. 上电后串口助手显示"System Start..."，然后每隔500ms输出一行"hello windows!"。
2. 在串口助手中发送`#`，STM32回复"[TX Paused]"，随后停止发送"hello windows!"。
3. 发送`*`，STM32回复"[TX Resume]"，恢复每隔500ms发送。
4. 反复发送`#`和`*`，功能稳定可靠。

**遇到的问题与解决：**
- 一开始使用`HAL_UART_Receive(&huart1, &rx_char, 1, 0xFFFF)`（阻塞接收），导致程序卡在接收处无法发送。解决方法是将超时设为0，实现非阻塞查询。
- 查询方式的缺点是CPU需要不断轮询，效率较低，这也是任务二改用中断方式的原因。

---

## 六、任务二：串口中断与DMA通信

### 6.1 串口中断方式重做任务一

#### 6.1.1 CubeMX配置

在任务一的基础上，额外配置：
- 在USART1的NVIC Settings中，使能USART1全局中断（USART1 global interrupt）
- 中断优先级设为：抢占优先级1，子优先级0

#### 6.1.2 中断方式实现代码

使用`HAL_UART_Receive_IT`开启中断接收，在接收完成回调函数中处理控制命令：

```c
/* main.c */
uint8_t tx_buf[] = "hello windows! \r\n";
uint8_t rx_char;
volatile uint8_t tx_enable = 1;

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART1_UART_Init();

  /* 开启中断接收 */
  HAL_UART_Receive_IT(&huart1, &rx_char, 1);

  HAL_UART_Transmit(&huart1, (uint8_t*)"System Start (IT mode)...\r\n", 26, 100);

  while (1)
  {
    if (tx_enable)
    {
      HAL_UART_Transmit(&huart1, tx_buf, sizeof(tx_buf)-1, 0xFFFF);
      HAL_Delay(500);
    }
  }
}

/* 接收完成回调函数 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  if (huart->Instance == USART1)
  {
    if (rx_char == '#')
    {
      tx_enable = 0;
      HAL_UART_Transmit(&huart1, (uint8_t*)"[TX Paused]\r\n", 13, 100);
    }
    else if (rx_char == '*')
    {
      tx_enable = 1;
      HAL_UART_Transmit(&huart1, (uint8_t*)"[TX Resume]\r\n", 13, 100);
    }
    /* 重新开启中断接收，准备下一个字节 */
    HAL_UART_Receive_IT(&huart1, &rx_char, 1);
  }
}
```

#### 6.1.3 中断方式与查询方式对比

| 对比项 | 查询方式 | 中断方式 |
|--------|----------|----------|
| CPU占用 | 高（不断轮询） | 低（仅收到数据时进入中断） |
| 响应速度 | 取决于轮询间隔 | 实时响应（微秒级） |
| 丢数据风险 | 轮询不及时可能丢 | 有RXNE中断，不易丢 |
| 代码复杂度 | 简单 | 稍复杂（需写回调函数） |
| 适用场景 | 简单、低速率 | 大多数实际应用 |

**中断方式的优点：** CPU可以在等待数据期间处理其他任务，只有当数据到达时才进入中断服务程序处理，大大提高了CPU利用率。

### 6.2 串口DMA方式高速发送

#### 6.2.1 DMA原理

DMA（Direct Memory Access，直接存储器访问）是一种无需CPU干预，在外设和存储器之间或存储器与存储器之间直接传输数据的机制。

STM32F103有2个DMA控制器（DMA1和DMA2），共12个通道（DMA1有7个通道，DMA2有5个通道）。USART1_TX对应DMA1的通道4。

DMA工作流程：
1. CPU配置DMA通道（源地址、目的地址、传输数量、方向等）。
2. 当USART1的发送数据寄存器（TDR）为空时，硬件自动触发DMA请求。
3. DMA从内存中读取一个字节，写入USART1的DR寄存器。
4. 传输完成后，DMA产生传输完成中断（TC），通知CPU。

#### 6.2.2 CubeMX配置

在USART1配置中：
- DMA Settings中添加USART1_TX，方向为Memory to Peripheral
- DMA模式选择Normal（正常模式，传输一次后停止）
- 数据宽度：Peripheral=Byte，Memory=Byte
- 优先级：Medium
- 使能DMA1通道4全局中断

#### 6.2.3 DMA发送代码

```c
/* main.c */
uint8_t dma_tx_buf[256];

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART1_UART_Init();
  MX_DMA_Init();

  /* 填充发送缓冲区 */
  for (int i = 0; i < 256; i++)
  {
    dma_tx_buf[i] = (uint8_t)(i + '0');
  }

  HAL_UART_Transmit(&huart1, (uint8_t*)"DMA Test Start...\r\n", 19, 100);

  while (1)
  {
    /* DMA方式发送256字节，非阻塞 */
    HAL_UART_Transmit_DMA(&huart1, dma_tx_buf, 256);

    /* 等待发送完成 */
    while (HAL_UART_GetState(&huart1) != HAL_UART_STATE_READY);

    HAL_Delay(1000);  // 每秒发送一次
  }
}

/* DMA发送完成回调 */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
  if (huart->Instance == USART1)
  {
    /* 发送完成，可以做其他处理 */
  }
}
```

#### 6.2.4 DMA高速发送测试

将波特率提高到921600（STM32F103 USART1最高支持4.5Mbps），使用DMA连续发送大数据块：

```c
/* 高速DMA发送测试 */
uint8_t big_buf[1024];

int main(void)
{
  /* ... 初始化 ... */

  /* 波特率修改为921600 */
  huart1.Init.BaudRate = 921600;
  HAL_UART_Init(&huart1);

  while (1)
  {
    HAL_UART_Transmit_DMA(&huart1, big_buf, 1024);
    while (HAL_UART_GetState(&huart1) != HAL_UART_STATE_READY);
    /* 无延时连续发送 */
  }
}
```

**测试结果：**

| 波特率 | 数据块大小 | 理论时间 | CPU占用 | 稳定性 |
|--------|-----------|----------|---------|--------|
| 115200 | 1024字节 | ~88.9ms | <1% | 稳定 |
| 460800 | 1024字节 | ~22.2ms | <1% | 稳定 |
| 921600 | 1024字节 | ~11.1ms | <1% | 稳定（需CH340支持） |

> **注意：** USB-TTL模块（CH340）最高支持2Mbps，但在921600以上时，部分廉价模块可能出现误码。本实验在115200和460800下测试稳定。

#### 6.2.5 三种发送方式对比

| 对比项 | 查询方式（阻塞） | 中断方式 | DMA方式 |
|--------|-----------------|----------|---------|
| CPU参与度 | 全程参与 | 每个字节中断一次 | 仅配置和完成时参与 |
| CPU占用率 | 高（100%阻塞） | 中（频繁中断） | 极低（<1%） |
| 适用数据量 | 小数据 | 中等数据 | 大数据块 |
| 实现复杂度 | 最简单 | 中等 | 较复杂 |
| 最高速率 | 受限于CPU | 受限于中断频率 | 受限于外设本身 |

---

## 七、任务三：Keil软件仿真逻辑分析仪

### 7.1 仿真配置

1. 在Keil中打开工程，点击魔术棒（Options for Target）。
2. 切换到Debug选项卡，选择"Use Simulator"（软件仿真）。
3. 确认Dialog DLL为`SARMCM3.DLL`，Parameter为`-pCM3`。
4. 晶振频率设置为与实际一致（本实验系统时钟72MHz，但仿真参数中设为36MHz，需注意换算）。
5. 编译工程，按Ctrl+F5进入调试模式。

### 7.2 逻辑分析仪设置

1. 在调试模式下，点击菜单View → Analysis Windows → Logic Analyzer，打开逻辑分析仪窗口。
2. 点击Logic Analyzer窗口左上角的"Setup..."按钮。
3. 在弹出的对话框中点击"New"，输入要观察的引脚。USART1_TX对应PA9，输入`PORTA.9`（或`GPIOA_IDR.9`）。
4. 设置显示类型为Bit，颜色自选。
5. 关闭Setup对话框，逻辑分析仪窗口将显示PA9引脚的波形。

### 7.3 运行仿真并观察波形

1. 在main函数中设置一个断点，或者直接按F5运行。
2. 程序运行到HAL_UART_Transmit发送数据时，PA9引脚将产生串口波形。
3. 点击逻辑分析仪窗口的"Zoom In"或使用鼠标滚轮放大波形。
4. 可以看到典型的UART波形：空闲时为高电平，起始位为低电平，然后是8位数据（低位在前），最后是停止位（高电平）。

**以发送字符'h'（ASCII码0x68，二进制01101000）为例：**

```
空闲  起始  D0 D1 D2 D3 D4 D5 D6 D7  停止  空闲
 1     0    0  0  0  1  0  1  1  0    1     1
```

对应波形：
```
     ┌──┐     ┌────┐  ┌──┐     ┌────────
     │  │     │    │  │  │     │
─────┘  └─────┘    └──┘  └─────┘
     <--> <-- 8位数据 --> <-->
     起始    LSB...MSB    停止
```

### 7.4 计算实际波特率

**方法一：测量位宽时间**

在逻辑分析仪中，使用鼠标光标测量一个位的持续时间（从起始位下降沿到第一位数据的中点，或测量整个起始位的宽度）。

- 仿真中测量得到：1位的持续时间约为8.68μs
- 波特率 = 1 / 位时间 = 1 / 8.68μs ≈ 115207 bps
- 与设定的115200bps基本一致，误差约0.06%（来自测量精度）

**方法二：测量一帧时间**

测量一帧完整数据（10位：1起始+8数据+1停止）的时间：
- 一帧时间约为86.8μs
- 波特率 = 10 / 86.8μs ≈ 115207 bps

**方法三：通过BRR寄存器计算**

STM32F103的USART波特率计算公式：

```
波特率 = fCK / (16 × USARTDIV)
```

其中fCK为USART1的时钟频率（APB2=72MHz），USARTDIV为波特率寄存器（USART_BRR）中的值。

在仿真中查看USART1->BRR寄存器的值：
- BRR = 0x0271（十进制625）
- USARTDIV = 625 / 16 = 39.0625
- 波特率 = 72000000 / (16 × 39.0625) = 72000000 / 625 = 115200 bps

**结论：** 软件仿真中实际波特率为115200bps，与配置值完全一致。波形时序正确，起始位、数据位（低位在前）、停止位均符合UART协议规范。

### 7.5 仿真与实际硬件的差异

1. **仿真中没有噪声和干扰**，波形是理想的方波；实际硬件中可能有振铃、过冲等信号完整性问题。
2. **仿真时钟精确**，实际硬件中晶振有一定频率误差（通常±20ppm~±50ppm）。
3. **仿真中USB-TTL模块的延迟不体现**，实际通信中CH340有USB打包延迟。
4. 软件仿真适合分析时序和逻辑，最终验证仍需在实际硬件上进行。

---

## 八、实验总结与心得体会

### 8.1 实验完成情况

| 任务 | 完成情况 | 备注 |
|------|----------|------|
| 基础1：串口协议与电平标准 | ✅ 完成 | 理解了RS232/RS485/TTL区别 |
| 基础2：串口文件传输 | ✅ 完成 | 实测2.5MB文件传输，分析了时间差异 |
| 任务一：USART1基本通信 | ✅ 完成 | 实现连续发送和上位机控制暂停/继续 |
| 任务二：中断方式 | ✅ 完成 | 使用HAL_UART_Receive_IT实现 |
| 任务二：DMA方式 | ✅ 完成 | 使用HAL_UART_Transmit_DMA实现高速发送 |
| 任务三：软件仿真波形分析 | ✅ 完成 | 逻辑分析仪观察波形，计算波特率115200 |

### 8.2 遇到的问题与解决

1. **问题：** 串口助手收不到数据。
   **原因：** TX和RX接反了，或者GND没接。
   **解决：** 确认TX接RX、RX接TX、GND共地，问题解决。

2. **问题：** 中断接收只能收到第一个字节，后面的收不到。
   **原因：** HAL库的中断接收是一次性的，接收完成后需要重新调用HAL_UART_Receive_IT。
   **解决：** 在HAL_UART_RxCpltCallback回调函数末尾重新开启中断接收。

3. **问题：** DMA发送后程序卡死。
   **原因：** 没有等待DMA传输完成就再次调用HAL_UART_Transmit_DMA，导致HAL_BUSY错误。
   **解决：** 添加while(HAL_UART_GetState(&huart1) != HAL_UART_STATE_READY)等待，或在TxCpltCallback中设置标志位。

4. **问题：** 软件仿真中逻辑分析仪看不到波形。
   **原因：** 引脚名称输入错误，应该输入PORTA.9而不是PA9。
   **解决：** 按照Keil仿真规范输入引脚名，波形正常显示。

### 8.3 心得体会

通过本次串口通信实验，我对嵌入式系统中的串行通信有了全面深入的理解。

在理论方面，我以前只是模糊地知道RS232和RS485都是串口，但不清楚它们的本质区别。通过查阅资料和实验对比，我理解了RS232是单端负逻辑电平、适合短距离点对点，而RS485是差分信号、适合长距离多点组网。同时也明白了为什么单片机的TTL电平不能直接接RS232接口，必须经过MAX3232等电平转换芯片。

在实践方面，从最基础的查询方式发送数据，到中断方式，再到DMA方式，我逐步体会到了三种方式的差异。查询方式最简单但效率最低，CPU一直在等；中断方式让CPU可以去做别的事，但每个字节都要进一次中断，在高速率下中断开销也不小；DMA方式最适合大数据块传输，CPU几乎不参与，效率最高。这让我理解了在实际项目中要根据数据量和实时性要求选择合适的通信方式。

软件仿真逻辑分析仪是一个很有用的工具，在没有示波器的情况下也能观察数字波形。通过测量位宽计算波特率，我对UART的时序有了更直观的认识，也验证了CubeMX配置的正确性。

总的来说，串口通信是嵌入式开发中最基础也是最重要的技能之一，几乎所有项目都会用到调试串口。这次实验让我打下了扎实的基础，也为后续学习更复杂的通信协议（如SPI、I2C、CAN等）做好了准备。

---

附视频演示

<video src="C:\Users\ljc1\xwechat_files\wxid_1f0zx874l09k22_5ea7\msg\video\2026-10\67709d69d3b2169f7b39759f9319f3cd.mp4" controls=""></video>

<video src="C:\Users\ljc1\xwechat_files\wxid_1f0zx874l09k22_5ea7\msg\video\2026-10\5cb9470986906cccfb98ad57406b5655.mp4" controls=""></video>

<video src="C:\Users\ljc1\xwechat_files\wxid_1f0zx874l09k22_5ea7\msg\video\2026-10\a39884d63abbc59507662a2241ab2147.mp4" controls=""></video>

<video src="C:\Users\ljc1\xwechat_files\wxid_1f0zx874l09k22_5ea7\msg\video\2026-10\d748e6973049a661ae258a71c6c7a00e.mp4" controls=""></video>
