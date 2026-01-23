exit和_exit的区别是_exit直接进内核
而exit会进行清理，刷新缓冲区
- _exit是系统调用，直接进入内核
- exit是库函数，会先调用终止函数，再刷新IO缓冲区，最后进入内核
return和exit的区别，return是C语言语句，而exit是库函数