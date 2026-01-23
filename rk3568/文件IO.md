
# open

## flag
- `O_RDWR`：**R**ea**D** **WR**ite，表示既要读也要写（双向通信）
- **`O_NONBLOCK`**：**非阻塞模式**。
```cpp
// 这里的 flags 是重点
int fd = open("/dev/rpmsg0", O_RDWR | O_NONBLOCK);
```

