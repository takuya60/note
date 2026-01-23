正常情况下，我们可以先`open()`之后，对文件进行读写
但是我们在操纵硬件的情况下，不仅仅是简单的读写，还需要对读写进行设置
所以就需要ioctl，即Input/Output Control
- 常规通道`open/read`用于传输数据
- 控制通道`ioctl`用于传输控制指令
## 函数原型
```c
int ioctl(int fd, unsigned long request, ...);
```
- fd 文件描述符
- request 暗号 内核驱动只认这个暗号
- ...通常是一个指针，指向需要配置的数值或者结构体
### request
unsigned long 是32位二进制数
![[Pasted image 20251216214629.png]]
## SPI的应用
在SPI里面，常用的request有
# 1.SPI_IOC_MESSAGE(N)
```c
struct spi_ioc_transfer trs[2]; // 注意：这是数组！
memset(trs, 0, sizeof(trs));

// 第 1 节车厢：发命令
uint8_t cmd = 0x05;
trs[0].tx_buf = (unsigned long)&cmd;
trs[0].len = 1;
// 关键点：通常不需要手动设置 CS 保持，因为同一个 MESSAGE 内核默认保持 CS 低电平

// 第 2 节车厢：读数据
uint8_t buffer[4];
trs[1].rx_buf = (unsigned long)buffer;
trs[1].len = 4;

// 告诉内核：我有 2 个动作包，必须连贯执行！
ioctl(fd, SPI_IOC_MESSAGE(2), trs); // 注意这里传的是数组首地址
```
## `memset`作用
在定义一个新的局部变量的时候，比如这里的结构体，内容是随机的垃圾值
memset可以把一块内存的所有字节刷成一个值
`void *memset(void *ptr, int value, size_t num);`
-  `ptr`: 要清洗的内存起始地址。
- `value`: 刷成什么值？（通常是 0）。
- `num`: 刷多大面积？（字节数）。

## `spi_ioc_transfer`结构体
![[Pasted image 20251216222221.png]]
# 2.SPI_IOC_WR_MODE
对应SPI四种模式，默认是第一种，信号平时低电平，上升沿采样
```cpp
uint8_t mode = SPI_MODE_0;
if (ioctl(fd, SPI_IOC_WR_MODE, &mode) < 0) {
    // 报错
}
```
# 3.SPI_IOC_WR_MAX_SPEED_HZ
设置时钟频率
参数类型uint32_t  单位赫兹
- RK3568 通常支持很高（几十 MHz）。
- M0 单片机可能较慢。
- 建议从 **1MHz (1000000)** 开始调试，稳定后再加到 5MHz 或 10MHz。
```c
uint32_t speed = 1000000; // 1MHz
if (ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed) < 0) {
    // 报错
}
```