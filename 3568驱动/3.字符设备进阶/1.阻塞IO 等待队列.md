### 定义并初始化等待队列
头文件wait.h
两种方法
第一种
```c
wait_queue_head_t m_queue;
init_wait_queue_head(&m_queue);
```
第二种 *（常用）*
```c
// 虽然原型里面参数是指针 但这里的m_queue并不是指针，后面如果要用指针，还得&
DECLEAR_WAIT_QUEUE_HEAD(m_queue);
函数原型
DECLEAR_WAIT_QUEUE_HEAD(wait_queue_head_t *q);
```

#### 休眠
```c
// 这里参数是队列头
wait_event_interruptbile(m_queue,condition);
```
![[Pasted image 20260128095038.png]]
#### 唤醒
![[Pasted image 20260128095144.png]]