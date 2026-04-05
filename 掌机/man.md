mkdir -p /mnt/sdcard
mount /dev/mmcblk0p1 /mnt/sdcard
cd /mnt/sdcard
export SDL_VIDEODRIVER=fbcon
export SDL_FBDEV=/dev/fb0
export SDL_NOMOUSE=1
./pcsx
strings output/rockchip_hd_rk3506g_evm_nand/target/usr/lib/libSDL-1.2.so* | grep -i "fbcon"



mkdir -p /mnt/usb
mount /dev/sda1 /mnt/usb
export SDL_VIDEODRIVER=fbcon
export SDL_FBDEV=/dev/fb0
export SDL_NOMOUSE=1
export SDL_VIDEO_YUV_HWACCEL=0
fbset -depth 16
cd /mnt/usb/ps
./pcsx -cdfile roms/re3.chd < /dev/tty1
