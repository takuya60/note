本文档在万象奥科瑞芯微3506g开发板上成功使用TinyDRM框架驱动ILI9341屏幕。开发板具体型号HD-RK3506-EVM
这篇教程会比较详细，是因为本人水平不高，写下这篇文档的主要目的是为了让自己记住
# 1.设备树
这里我的开发板的设备树路径是`kernel6.1/arch/arm/boot/dts/vanxoak-hd-rk3506-evm-v1.dts`
这里设备树配置不同厂家的可能不同，我这里参考了万象奥科文档
这里需要注意，tinyDRM的参数与fbftf不同，比如rotation等
```c
&spi0{
	status = "okay";
	pinctrl-names = "default";
	pinctrl-0 = <&rm_io7_spi0_csn0 &rm_io3_spi0_mosi &rm_io6_spi0_miso &rm_io1_spi0_clk>;// 这一项跟据实际开发板的dtsi写

	display@0 {
		compatible = "adafruit,yx240qv29";
		reg = <0>;
		spi-max-frequency = <30000000>;
        rotation = <90>;
		dc-gpios= <&gpio1 RK_PD2 GPIO_ACTIVE_HIGH>; 
        reset-gpios = <&gpio1 RK_PD1 GPIO_ACTIVE_LOW>;
	};

};
```
# 2.内核配置
## 2.1spi相关
因为我们使用的是spi屏幕，所以要打开spi支持，进入内核目录 `make ARCH=arm menuconfig`
进入device drivers 再进入SPI support
这里需要打开这两个选项
- Rockchip SPI controller driver
- User mode SPI device driver support
打开config之后，配置应该如下(这里的配置结果和万象奥科文档一样，但是文档里没有提打开Rockchip SPI controller driver,只说了User mode SPI device driver support ）
```c
CONFIG_SPI=y
CONFIG_SPI_MASTER=y
CONFIG_SPI_SPIDEV=y
CONFIG_SPI_ROCKCHIP=y
```
验证成功：烧录开机之后，在/dev/下面可以看到spi0.0节点
## 2.2TinyDRM相关
menuconfig之后，进入device drivers 再进入 Graphics support
然后打开
- Enable legacy fbdev support for your modesetting driver
- Direct Rendering Manager (XFree86 4.1.0 and higher DRI support
- DRM support for ILI9341 display panels
- 
