mkdir -p /mnt/sdcard 
mount /dev/mmcblk0p1 /mnt/sdcard 
ls /mnt/sdcard

strings output/rockchip_hd_rk3506g_evm_nand/target/usr/lib/libSDL-1.2.so* | grep -i "fbcon"