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
// 这里传入的是指针名字
DECLEAR_WAIT_QUEUE_HEAD(m_queue);
函数原型
DECLEAR_WAIT_QUEUE_HEAD(wait_queue_head_t *q);
```