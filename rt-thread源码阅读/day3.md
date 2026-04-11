## thread中的resume函数
作用，唤醒一个线程
唤醒之前，首先压迫进行上锁，调用rt_sched_lock （&slvl）
这传入的是sched lock level的指针，这个是 调度器锁状态 存了cpu寄存器
rt_sched_lock这个函数负责关中断，调用的是rt_hw_interrupt_disable
这个关中断的函数是跟据硬件决定的，会把当前线程的状态存下来，通过返回值，存到slvl里面
