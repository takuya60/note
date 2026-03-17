这是一个非常经典、也是 STM32H7 系列让无数开发者“踩坑”的底层硬件特性：**D-Cache（数据缓存）与 DMA 的数据一致性问题**。

为了讲透这个问题，我们需要了解 STM32H7 的内存架构，以及为什么**有时候好用，有时候突然出问题**。

### 1. 什么是 D-Cache？CPU 与 DMA 的“视角差异”

STM32H7 因为主频极高（达 400MHz 甚至 480MHz），而内部 SRAM 或外部 SDRAM 的读写速度跟不上 CPU。为了不让 CPU 傻等内存，ST 引入了 L1 Cache（包含 I-Cache 指令缓存和 D-Cache 数据缓存）。

**当你用 CPU 往一个数组 `tx_buf` 里写数据时（例如你在** 

**![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)**

**_fes_build_packet 里构建协议帧）：**

- CPU **并没有真正把数据写进物理内存 (SRAM)**。
- 而是先写进了 CPU 旁边速度极快的 **D-Cache** 里。CPU 认为：“我已经写完了，内存以后有空再慢慢同步吧（写回策略 Write-Back）”。

**但 DMA 呢？**

- DMA 是一个**独立的硬件搬运工**。
- 它工作在总线矩阵上，**不经过 CPU，也不能访问 CPU 的 D-Cache**。
- 当 `rt_spi_send` 启动 DMA 去物理内存里搬运 `tx_buf` 给 SPI 外设时，DMA 直接去读物理内存。

**结果就是灾难：** CPU 写在 D-Cache 里的最新数据，还没来得及同步到物理内存；DMA 去物理内存里一摸，读到的是之前老旧的数据（或者是全 0）。SPI 发送了一堆乱码，接收端（从机）验证失败不回包，或者 DMA 传输过程挂死，最终导致 `wait for DMA interrupt overtime!` 超时。

---

### 2. 为什么你之前能正常使用？

这正是 D-Cache 问题最让人痛苦的地方——它像幽灵一样，**时而正常，时而崩溃**，完全取决于代码在编译和运行时的“微小扰动”。

你之前能正常使用，通常是由以下几种巧合造成的：

#### 巧合 A：变量的物理存储位置刚好没接 D-Cache

STM32H7 的内存分很多块。比如 `DTCM RAM`（通常地址 0x20000000 起始）是直接挂在 CPU 数据总线上的，它**天生不经过 D-Cache**。 如果之前代码里你的 `tx_buf` 刚好被链接器分配到了 `DTCM RAM`，或者分配到了某个通过 MPU（内存保护单元）配置成了 Non-Cacheable 的内存区，那么 CPU 直接写穿透到物理内存，DMA 去读就完全没问题。 但随着代码的增加、删减，RT-Thread 动态分配的内存，或者因为你在这个文件里把它定义为了 `static` 变量，链接器把它移到了 `AXI SRAM` 或 `SRAM1/2/3` 里——这些区域是默认开启 D-Cache 的，问题就爆发了。

#### 巧合 B：缓存“碰巧”被挤出（Eviction）了

由于 D-Cache 很小（例如 32KB），如果之前的代码在组装完 `tx_buf` 之后，CPU 又去干了大量别的事情（比如处理 FDCAN、打印大量日志、处理 UI 等等），导致 D-Cache 容量不够了，硬件会自动把旧的 Cache 内容（碰巧就是你的 `tx_buf`）**踢出去，写回到物理内存中**。 等这时候 DMA 再去读，就恰好读到了刷新后的正确数据！ 但在我们这次重构服务层后，收到 FDCAN 消息立马触发线程运行，组装好包立马启动 SPI 发送，CPU 根本没时间做其他事，Cache 还在里面捂着呢，DMA 就提前去读物理内存了，自然读不到最新数据。

---

### 3. 如何彻底解决？

我们刚才加代码用的就是**方法二**：

**方法一：利用 MPU 配置不使用 Cache 的内存区（最好、也是工程上最推荐的）** 在 STM32 的 BSP 启动代码里，利用 MPU（Memory Protection Unit）配置一块比如 32KB 或者 256KB 的内存区域为 **"Non-Cacheable"（不可缓存）**。 然后把你所有给 DMA 用的 buffer 都放到这个区，或者改掉 `rt_malloc`，让含有这些 buffer 的结构体从这块内存里分配。这样不管你怎么写，CPU 都会直接写进物理内存。

**方法二：软件手动维护一致性（我们刚才加的代码）** 既然 CPU 把数据扣在了 D-Cache 里，那就强制要求它交出来。

c

// CPU 你把这块范围的 D-Cache 立刻写到物理内存去！

SCB_CleanDCache_by_Addr((uint32_t *)tx_buf, CONTROL_PACKET_LEN);

对于 SPI 接收（如果你有 RX buffer 的话），同理：DMA 把数据从外设搬到了物理内存，但 CPU 会去读 D-Cache 里的老数据。这时候不仅要把 D-Cache 写回，还要把它作废：

c

// CPU 你别读自己 D-Cache 里的旧数据了，去物理内存里重新读取！

SCB_InvalidateDCache_by_Addr((uint32_t *)rx_buf, CONTROL_PACKET_LEN);

**为什么还有个 `__attribute__((aligned(32)))`？** 因为 STM32H7 的 Cache Line 这个基本单位是 **32个字节（8个字）**。它刷新内存是一捆一捆（32字节）刷新的。 如果你的 `tx_buf` 只有 15个字节，且起始地址没有 32 字节对齐，那它和别的变量拼在一个 Cache Line 里。你一刷新，可能会把旁边别人的数据状态也打乱，引发更加诡异的 Bug。所以硬件手册规定：凡是用到 `SCB_CleanDCache_by_Addr`，对应的数组在内存里**必须起始于 32 字节边界对齐，且大小最好是 32 的整数倍**。

通过这样解释，你应该彻底明白为什么 STM32H7 上玩高端外设（DMA、以太网 MAC、USB、SDIO）老是容易卡住了吧！