mkdir -p /mnt/sdcard
mount /dev/mmcblk0p1 /mnt/sdcard
cd /mnt/sdcard
export SDL_VIDEODRIVER=fbcon
export SDL_FBDEV=/dev/fb0
export SDL_NOMOUSE=1
./pcsx
strings output/rockchip_hd_rk3506g_evm_nand/target/usr/lib/libSDL-1.2.so* | grep -i "fbcon"