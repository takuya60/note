硬盘中除了数据区，还有inode区，用来存放`inodtable`(inode表)，inode表里存放了很多inode，不同inode对应不同的文件
inode里有关于这个文件的信息，比如字节大小，文件类型，权限...但是没有文件名
![[Pasted image 20251202222937.png]]

每个inode有自己的编号，linux中可以用ls -il查看inode编号

## 打开文件之后

这里的PCB是Process Control Block进程控制块，一个程序就是一个进程，PCB里面有个指针指向fd，fd再指向文件表，文件表里的inode指针再指向inode

![[Pasted image 20251202223514.png]]

# 错误处理
## errno
函数的常见错误会有编号，出现错误时，把这个编号赋值给errno变量，errno是全局变量
具体使用，需要先包含errno.h ，但是errno只是一个int，如果想知道具体含义，
需要用strerror函数
```cpp
#include <errno.h>
#include <string.h>
printf("Error:%s\n",strerror(errno))
```
## perror

这个**用的最多**，不需要传入errno，自己就能输出字符串
```cpp
//使用示例
if(fd==-1)
{
	perror("open error")
	//这里的参数，是自己在错误输出之前想加上的话，可以不写
	return -1
}
```
