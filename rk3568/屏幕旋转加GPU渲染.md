

直接运行qt的话，会非常卡
我们在运行之前先设置变量，让QT能够输出渲染信息
`export QSG_INFO=`
再运行程序
`./demo1`
![[Pasted image 20251215125900.png]]
llvmpipe代表正在使用纯CPU渲染，这是因为Debian桌面服务Wayland正在用屏幕，霸占了GPU
所以需要先关闭桌面服务
# 停止桌面服务Debian
`sudo systemctl stop lightdm`
如果要重新开启
`sudo systemctl start lightdm`
# 桌面黑屏后，再运行程序

我们之前在桌面设置里面的横屏已经没用了，因为桌面服务已经关闭了
这里想要显示横屏的程，有两种方法
### 1.在QT程序的代码里旋转
在主页面里加一个Item
```cpp
        width: 1280
        height: 800
        // 居中定位
        anchors.centerIn: parent
        // 旋转 90 度
        rotation: 90
```
### 2.创建JSON 配置文件
![[Pasted image 20251215130722.png]]

再启动QT程序的时候
必须在后面加上-platform eglfs
`sudo ./demo01 -platform eglfs`
