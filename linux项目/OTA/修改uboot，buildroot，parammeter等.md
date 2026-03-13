详见瑞芯微文档`Rockchip_Developer_Guide_Linux_Recovery_CN`
## 修改uboot
uboot/configs/rk3506_deconfig 
增加了
```c
CONFIG_AVB_LIBAVB=y
CONFIG_AVB_LIBAVB_AB=y
CONFIG_AVB_LIBAVB_ATX=y
CONFIG_AVB_LIBAVB_USER=y
CONFIG_RK_AVB_LIBAVB_USER=y
CONFIG_ANDROID_AB=y
```
## 修改buildroot 
buildroot/configs/rk3506_deconfig
```c
BR2_PACKAGE_RECOVERY=y
BR2_PACKAGE_RECOVERY_BOOTCONTROL=y
BR2_PACKAGE_RECOVERY_RETRY=y  
BR2_PACKAGE_RECOVERY_USE_UPDATEENGINE=y
BR2_PACKAGE_RECOVERY_UPDATEENGINEBIN=y
BR2_PACKAGE_RECOVERY_NO_UI=y
```
`BR2_PACKAGE_RECOVERY_RETRY=y  `代表设置引导方式为retry模式，不配置则默认为 successful_boot模式

`BR2_PACKAGE_RECOVERY_USE_UPDATEENGINE=y #使用新升级程序` `BR2_PACKAGE_RECOVERY_UPDATEENGINEBIN=y #编译新升级程序`
这个指的是静默升级程序


./build.sh external/recovery

./build.sh

zip -r firmware.zip *