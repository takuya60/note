
 cat /sys/kernel/debug/gpio 查看gpio分配情况

![[Pasted image 20260405230942.png]]
![[Pasted image 20260405230957.png]]

```c

gpiochip0: GPIOs 0-31, parent: platform/ff940000.gpio, gpio0:
 gpio-8   (                    |PS Triangle (Triangl) in  hi IRQ ACTIVE LOW
 gpio-9   (                    |my touch touch gpio ) in  lo IRQ
 gpio-11  (                    |PS Square (Square)  ) in  lo IRQ ACTIVE LOW
 gpio-12  (                    |PS Circle (O)       ) in  lo IRQ ACTIVE LOW

gpiochip1: GPIOs 32-63, parent: platform/ff870000.gpio, gpio1:
 gpio-43  (                    |run                 ) out hi ACTIVE LOW
 gpio-44  (                    |my touch touch gpio ) out hi
 gpio-52  (                    |PS Cross (X)        ) in  lo IRQ ACTIVE LOW
 gpio-53  (                    |D-Pad Right         ) in  lo IRQ ACTIVE LOW
 gpio-54  (                    |D-Pad Left          ) in  lo IRQ ACTIVE LOW
 gpio-55  (                    |D-Pad Down          ) in  lo IRQ ACTIVE LOW
 gpio-56  (                    |D-Pad Up            ) in  lo IRQ ACTIVE LOW
 gpio-57  (                    |ili9341-always-on-re) out hi
 gpio-58  (                    |dc                  ) out hi
 gpio-59  (                    |reset               ) out hi ACTIVE LOW

gpiochip2: GPIOs 64-95, parent: platform/ff1c0000.gpio, gpio2:

gpiochip3: GPIOs 96-127, parent: platform/ff1d0000.gpio, gpio3:
 gpio-102 (                    |cd                  ) in  hi IRQ ACTIVE LOW
 gpio-103 (                    |Start               ) in  lo IRQ ACTIVE LOW
 gpio-104 (                    |Select              ) in  lo IRQ ACTIVE LOW
 gpio-105 (                    |Desk/Game Mode Switc) in  lo IRQ ACTIVE LOW

```
dmesg | grep -iE "snd|asoc|simple|sai|max98"
