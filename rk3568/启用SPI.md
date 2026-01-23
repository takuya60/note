开发板在默认情况下，SPI是不开启的
`ls /dev/spidev*`
发现什么都没有输出
所以需要我们自己添加dtbo，而不需要重新编译整个设备树
首先，我们需要进入boot文件夹下
`cd /boot`
进入之后，会发现有`mmc0_extlinux`,`mmc1_extlinux`两个文件，0是通过emmc启动的引导配置，1是通过sd卡启动的引导配置
因为默认是从emmc启动，所以我们需要修改
`mmc0_extlinux`里面的`extlinux.conf`
进去之后，在`fdtoverlays /.....`这行后面加上`/dtbs/rk3568-spi3-m1.dtbo`
dtbs这个文件夹也在boot下面，这里面存放了正点原子提供的dtbo文件，我们需要用到的`rk3568-spi3-m1.dtbo`就在这里面
