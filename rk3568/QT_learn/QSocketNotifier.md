## 作用
把Linux的系统中断，转换为QT的信号和槽
==//这里的socket相当于文件描述符fd==
# 使用流程
## 1.头文件
在头文件中
```cpp
#include <QsockerNotifier>
private slots:
	//这里的socket相当于文件描述符fd
	void onReadActived(int socket);
private：
	int m_fd;
	//监听器指针
	QsocketNotifier *m_notifier;

```
## 2.初始化与连接
通常在打开设备文件后立刻进行
```cpp
//在构造函数中
m_fd=-1;
m_notifier = nullptr;
//打开设备
//O_RDWR：读写模式  O_NONBLOCK:非堵塞模式（必须加）
m_fd = open("/dev/ttyRPMSG0", O_RDWR | O_NONBLOCK);
//创建监听器
m_notifier = new QSocketNotifier(m_fd,QSocketNotifier::Read,this);
//连接信号槽
connect(m_notifier,&QSocketNotifier::activated,
this,&RPMsgBackend::onReadActivated);
//开始监听
m_notifier->setEnabled(true);
```
## 3.处理数据（实现槽函数）
```cpp
因为进行了connet，所以notifier监听到之后，会自动触发这个函数
void onReadActived(int socket)
{
	//这里的socke就是m_fd
	//buffer是用来存读取的数据的
	ssize_t = read(scoket,buffer,sizeof(buffer))；

}
```
## 4.清理资源
```cpp
//在析构函数中
RPMsgBackend::~RPMsgBackend() { 
// 1. 先停用监听器 
if (m_notifier) {
 m_notifier->setEnabled(false);
  delete m_notifier; 
  // 这一步会自动断开 connect } 
  // 2. 再关闭文件 
if (m_fd > 0) { 
  close(m_fd); 
	 }
  }

```