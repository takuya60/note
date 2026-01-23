# pread和pwrite

把先移动Iseek再write合并成一步，没法打断，成为原子操作
调用时，在原本的read函数上加了一个offset参数，用来调整读写位置
**但是这个offset不会移动Iseek**
```cpp
ret = pread(fd, buffer, sizeof(buffer), 1024);
```

# fcntl和ioctl
## fcntl
系统调用
其中cmd是一系列操作命令
```cpp
int fcntl(int fd, int cmd, ... /* arg */ )
```
![[Pasted image 20251202231729.png]]
除了dup2，fcntl也可以用来复制文件描述符
```cpp
fd2 = fcntl(fd1, F_DUPFD, 0);
//这里的第三个参数，指的是新的fd的最小值
```
还可以查看或者修改open时的flags标志
```cpp
ret = fcntl(fd, F_SETFL, flag | O_APPEND);
```
## ioctl
一般用于操作特殊文件或者硬件外设，比如获取led信息

# 文件截断
使用系统调用 truncate()或 ftruncate()可将普通文件截断为指定字节长度
```cpp
int truncate(const char *path, off_t length);
int ftruncate(int fd, off_t length);
```
ftruncate()使用文件描述符 fd 来指定目标文件，而 truncate()则直接使用文件路 径 path 来指定目标文件，其功能一样。