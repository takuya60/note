是标准C语言库，基于系统调用的IO库之上
需要include stdio.h
标准IO库比IO库可移植性更好，因为不同操作系统的系统调用不一样
标准IO库有自己的stdio缓冲区，性能和效率更好

## fopen
```cpp
#include <stdio>
FILE *fopen (const char* path, const char*mode);
```
#### mode
- r 只读，对应O_RDONLY
- r+ 可读可写，对应O_RDWR
- w 只写，如果文件已经存在 把长度截断为0，若不存在则创建，相当于O_WRONLY | O_CREAT 
| O_TRUNC
- w+ 相比于w，是可读可写，其他都一样
- a 只写，以追加内容打开，不存在则创建，相当于O_WRONLY | O_CREAT 
| O_APPEND
- a+ 可读可写，其他与a一样
## fclose
```cpp
int fclose(FILE *stream);
```
## fread

```cpp
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
```
ptr：fread()将读取到的数据存放在参数 ptr 指向的缓冲区中 
size：fread()从文件读取 nmemb 个数据项，每一个数据项的大小为 size 个字节，所以总共读取的数据大 小为 nmemb * size 个字节。
nmemb：参数 nmemb 指定了读取数据项的个数。
stream：FILE 指针
## fwrite
```cpp
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

ptr：将参数 ptr 指向的缓冲区中的数据写入到文件中。
size：参数 size 指定了每个数据项的字节大小，与 fread()函数的 size 参数意义相同。 
nmemb：参数 nmemb 指定了写入的数据项个数，与 fread()函数的 nmemb 参数意义相同。 stream：FILE 指针。

# 使用案例
```cpp
#include <stdio.h>
#include <stdlib.h>
int main (void)
{
char buf[50]={0};
FILE *fp =NULL;
int size;

if(NULL==(fp=open(./file.txt,w+)))
{
	perror("open error");
	exit(-1);
}
print("成功打开");



if(sizebuf>write(buf,1,sizeof(buf),fp))
{
	print("write error");
	fclose(fp);
	exit(-1);
}
print("成功写入");

if(10>fread(buf,1,10,fp))
{
	print("read error");
	fclose(fp);
	exit(-1);
}
print("成功读取");
fclose(fp);
}
```
## fseek
和Iseek用法一样
但是需要注意，库函数fwrite，fread读写时，位置偏移量会自动改变
成功返回1，错误返回0
```cpp
int fseek(FILE *stream, long offset, int whence);
```
## ftell
用于获取文件当前的读写位置偏移量
```cpp
long ftell(FILE *stream);
```
## ferror
ferror()用于测试参数 stream 所指文件的错误标志

## clearerr
clearerr()用于清除 end-of-file 标志和错误标志

# 格式化IO
## 格式化输出
- print() 写到标准输出
- fprint() 写到由FLIE * 指定的文件
- dprintf() 写到由fd 指定的文件
- sprintf() 写到参数 buf 所指定的缓冲区中、
- printf()函数可能会发生缓冲区溢出的问题，存在安全隐患，为了解决这个问题，引入了 snprintf()函数
