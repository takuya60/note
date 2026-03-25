## 环境配置

1.打开魔术棒，c++,在define里面加入
`ARM_MATH_CM7,__CC_ARM`
2.在`project\board\CubeMX_Config\Drivers\CMSIS\DSP\Lib\ARM`路径下面
找到`arm_cortexM7l_math.lib`然后把这个添加到工程里
3.在c++里面添加include
`project\board\CubeMX_Config\Drivers\CMSIS\DSP\Include`
4.
![[Pasted image 20260325215413.png]]
