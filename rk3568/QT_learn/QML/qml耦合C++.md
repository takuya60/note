## 在C++的类中使用Q_PROPERTY
```cpp
Q_PROPERTY(type name
           READ getFunction
           [WRITE setFunction]
           [RESET resetFunction]
           [NOTIFY notifySignal]
           [DESIGNABLE bool]
           [SCRIPTABLE bool]
           [STORED bool]
           [USER bool]
           [CONSTANT]
           [FINAL])
```
```cpp
Q_PROPERTY(Runstate currentState READ currentState NOTIFY runstateChanged)
```
- Runstate是type name，可以是一个int，bool，结构体等等
- currentState是属性名称，是qml里面使用的名称
- READ currentState 读取函数，指定一个成员函数来获取这个
-  NOTIFY runstateChanged 通知信号(可选)，当属性改变的时候发出的信号，需要在后面signal里面定义
# c++中
## 1.上下文属性注册
```cpp
// C++ 中注册对象
IBackend* backend =new RK3568Bakend();
TreatManager* treatmanager=new TreatManager(backend);
QQmlApplicationEngine engine;
engine.rootContext()->setContextProperty("treatmentManager", treatmentManager);
```
最后一行`"treatmentManager"`是qml中使用的名字，第二个参数是treatmanager指针

### 2.qmlRegisterType 注册
使用的时候，必须是无参构造函数
```cpp
// 注册类型
qmlRegisterType<TreatmentManager>("MyCompany", 1, 0, "TreatmentManager");
```
- TreatmentManager是要注册的c++类
- 1，0是主版本和次版本
- "MyCompany"是URI
-  "TreatmentManager"是QML类型名
## 在qml里面访问
```cpp
import MyCompany 1.0

TreatmentManager {
    id: treatmentManager
}

Text{
	text: treatmentManager.currentstate===TreatmentManager.Running ?"运行中" : "待机"
}
```
### 3.qmlRegisterUncreatableType注册
不会像qmlRegisterType一样创建实例
```cpp
qmlRegisterUncreatableType<TreatmentManager>("MyCompany", 1, 0, "TreatmentManager", "TreatmentManager should not be created in QML");
```
最后的参数是错误信息
