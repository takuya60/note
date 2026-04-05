本文档在万象奥科瑞芯微3506g开发板上成功使用TinyDRM框架驱动ILI9341屏幕。开发板具体型号HD-RK3506-EVM
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

## 2.2TinyDRM相关
menuconfig之后，进入device drivers 再进入 Graphics support
然后打开
- Enable legacy fbdev support for your modesetting driver
- Direct Rendering Manager (XFree86 4.1.0 and higher DRI support
- DRM support for ILI9341 display panels
- DRM support for MIPI DBI compatible panels
然后进入Frame buffer Devices 打开Support for frame buffer devices
**注意**  这里的DRM support for ILI9341 display panels是根据你的屏幕的型号来的，如果没有，就需要自己编写驱动。这里的驱动路径在`sdk/kernel/drivers/gpu/drm/tiny`下面
上面设备树的compatible可以直接进对应的驱动文件里面去看
修改完之后记得，把默认配置进行覆盖，这里要覆盖的文件需要跟据开发板而定
cp .config arch/arm/configs/vanxoak_hd_rk3506g_evm_nand_defconfig 
**这里修改完之后先不要进行编译**

## 3.默认配置修改
上面修改完如果直接进行编译，DRM support for ILI9341 display panels是不会被编译的，
因为编译的时候加载了一个名为 `rk3506-display.config` 的补丁包，这个补丁包里会对我们上面的配置进行覆盖。所以我们需要在sdk目录下
`gedit kernel/arch/arm/configs/rk3506-display.config`
删掉下面这两句
    CONFIG_TINYDRM_ILI9341 is not set
    CONFIG_DRM_PANEL_MIPI_DBI is not set
然后加上
```c
CONFIG_DRM_PANEL_MIPI_DBI=y
CONFIG_TINYDRM_ILI9341=y
```
这时候再`./build kernel`就行了

# 3.上电测试
因为tinyDRM有省电功能，所以reset引脚只有在启动一些图形化界面的时候才会被拉高
如果我们想要进行快速测试，我们需要把reset引脚连在3.3电源上
如果·在/dev/下面可以看到spi0.0节点，说明spi设备树挂载成功
执行cat /sys/bus/spi/devices/spi0.0/modalias之后，应该出现compatible例如spi:yx240qv29，代表驱动成功加载。
然后执行下面的命令，屏幕应该会出现花屏，
`dd if=/dev/urandom of=dev/fb0 bs=1024 count=150`

