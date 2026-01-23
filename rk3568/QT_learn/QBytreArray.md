# 优点
### 一、不怕0
如果用c的char*，是把\0作为结束标志的，所以数据中最好不要出现0x00
### 二、自带深拷贝优化
如果在函数之间中传递QByteArray的适合，不会拷贝一遍，而是复制一个指针，只有在实际进行修改的适合才会复制
# 五大用法
## 一、添加(append)
```cpp
QByteArray buffer；
buffer.append(0xAA);
buffer.append("Hello");
```
## 二、查看长度(size)
```cpp
if(buffer.size()>4){
	//攒够四个字节再处理
}
```
## 三、移除(remove)
```cpp
buffer.remove(0,2);
```
## 四、查找(at或者[])
```cpp
if((unsigned char)buffer.at(0)== 0xAA){
	//发现帧头！
}
```
## 五、转换成16进制
```cpp
buffer.append(0xAA);
buffer.append(0x55);
// 打印出来是: "aa55"
qDebug()<<"收到的数据"<<buffer.toHex();
// 甚至可以加空格，打印成: "aa 55" (Qt 5.9+ 支持)
qDebug()<<"收到的数据"<<buffer.toHex('');

```