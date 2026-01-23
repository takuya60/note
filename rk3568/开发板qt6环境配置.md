# 方式选择
正点原子提供的buildroot是基于瑞芯微的buildroot的，瑞芯微的buildroot是2018版本，所以如果想自己构建文件系统，无法配置qt6
正点原子AtomPi-CA1 卡片电脑在拿到手的适合，上面是有debian11系统的，所以我们选择直接在debian11上直接下载qt6，把代码直接放在开发板上进行编译
# qt6下载
我们直接使用qt的在线下载器
在线下载器可以去qt官方网站，也可以去清华镜像站下载
https://mirrors.tuna.tsinghua.edu.cn/qt/official_releases/online_installers/
在下载之后，先进入文件所在目录修改运行权限就可以直接运行
在运行的时候，建议使用国内镜像源加速
这里我nju，tsinghua，ustc的镜像源都试过，在使用tsinghua的时候，有概率会报错，所以建议用nju的，也有可能是因为我用的校园网的原因
```cpp
chmod +x qt-on...online.run
./qt-on...online.run  --mirror https://mirrors.nju.edu.cn/qt/

```
在填写qt账户的密码等之后，会出现类似这样的选择（没截图，图是偷的别的视频教程的）
他这个在线下载器是旧版本的，如果是新版本的，我们需要先勾选上qt6.10的desktop端，**然后再勾选自定义安装**。
![[Pasted image 20251214214342.png]]
点击下一步之后，会让选择安装的组件，我们这里先展开6.10，看一眼6.10都有哪些已经勾选的
然后在下面展开6.7.3版本，把类似的都勾选上，然后再把6.10的都取消掉，然后再把qtcreator取消勾选，因为我们不能使用新版本的qtcreator，打不开，下面会说原因
![[Pasted image 20251214214539.png]]
但是由于debian11的glibc 版本是 **2.31**，所以我们无法使用6.7以上的版本，以及新的qtcreator，下载下来之后也无法运行
所以我们接下来需要自己下载老版本的qtcreator，在终端中直接输入`sudo apt install qtcreator`
会自动下载老版本的qtcreator，然后想要运行的时候，打开终端，直接输入qtcreator即可运行

现在，运行qtcreator，点击上面的tools，再点击下面的Options（好像是这个，反正是最下面的），进入之后，先看一下左侧，应该是在`Kit`里面，然后在右面上方，点击`Qt Versions`，点击Add，进入刚刚安装的qt6.7.3的路径下的gcc_arm64/bin，选择qmake，如果之前下载的是比6.7高的版本，这里的qmake会打不开
然后在右边上面的Kit，点击Add，会让你配置name，Qt version等，这里Qt version直接选择我们刚刚添加的qt6.7.3，Compiler都选GCC，现在QT6的环境就配置完成了