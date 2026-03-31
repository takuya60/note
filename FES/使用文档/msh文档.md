```c
msh /> stim init                # 初始化(如果没自动初始化)
msh /> stim pair 1              # 第1路通道
msh /> stim time 100 200 200 50 # 100Hz, 正负脉宽200us, 死区50us
msh /> stim amp 500 500         # 正负幅值均设为 500uA
msh /> stim start               # 启动输出
msh /> stim status              # 查看当前输出状态和电压/电流反馈
msh /> stim stop                # 停止输出
```