在TreatmentManager.h中
```cpp
Q_PROPERTY(int remainingTime READ remainingTime NOTIFY timeUpdated)
```

这句话代表`timeUpdated()` 这个信号在 `TreatmentManager` 对象上被 emit 时，QML 会重新调用 `remainingTime()` 并刷新界面
有三个关键点
- NOTIFY 信号必须由**这个 QObject 自己发出
- QML **不会追踪 getter 内部访问了谁**
- QML **不会因为别的对象发信号而自动刷新**
## 之前的实现方式：
```cpp
Q_PROPERTY(int remainingTime READ remainingTime NOTIFY timeUpdated)
```
这里的`remainingTime()`直接读取`m_service`里面的time
```cpp
int TreatmentManager::remainingTime() const
{
    return m_service->remainingTime();
}
```
然后把`TreatmentService`里面的timeUpdated更新信号连接到`TreatmentManager`的timeUpdated上面
```cpp
connect(m_service, &TreatmentService::timeUpdated,
        this, &TreatmentManager::timeUpdated);

```
但是由于Q_PROPERTY在`TreatmentManager` 里面，所以QML只认识`TreatmentManager` ，**不知道** `remainingTime()` 实际上是去读了 `m_service`。
如果`TreatmentManager`和`TreatmentService`在不同线程，这样是不安全的
## 改进
应该在`TreatmentManager`有一份time的缓存
```cpp
class TreatmentManager : public QObject
{
    Q_OBJECT
    // 这里不变
    Q_PROPERTY(int remainingTime READ remainingTime NOTIFY timeUpdated)

private:
    int m_remainingTime = 0;
};
```
数据先更新，再 emit NOTIFY，QML 读到的一定是已更新后的值
```cpp
connect(m_service, &TreatmentService::timeUpdated,
        this, [this](int t){
            m_remainingTime = t;
            emit timeUpdated();
        });
```
getter 只读自己的成员，不再去读`m_service`的
```cpp
int TreatmentManager::remainingTime() const
{
    return m_remainingTime;
}
```