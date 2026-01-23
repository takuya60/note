用于多线程同步，假如正在通过SPI进行读取数据，现在又有一个指令是写数据，程序就会出错
所以在写数据之前应该上锁
在本项目中 fd相当于是所有线程共享的
## QMutexLocker
先在类中创建一个QMutex对象
然后用QMutexLocker上锁
```cpp
void RK3568Backend::goodMethod() {
    // 创建即上锁
    QMutexLocker locker(&m_mutex); 
    
    // 干活...
    if (someError) {
        return; // 放心返回！locker 被销毁时会自动调用 m_mutex.unlock()
    }
    // 函数结束，locker 销毁，自动解锁
}
```