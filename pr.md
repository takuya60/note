## 拉取/合并请求描述：(PR description)

[
#### 为什么提交这份PR (why to submit this PR)

我在使用stm32H7的fdcan驱动时遇到了无法发送与接收的问题，并且发现没有ISO-CANFD常用的5M速率
#### 你的解决方案是什么 (what is your solution)

1.解决了发送数据时没有对是否进行数据位加速(BRS)进行判断的问题，新增了一个if判断，判断上层传来的pmsg->brs，如果是1，则把TxHeader.BitRateSwitch置为FDCAN_BRS_ON；如果是0，则置为FDCAN_BRS_OFF

2.解决了在数据接收时对数据位数判断错误，原来是直接把RxHeader.DataLength，这样只能传入0-15这几个数，所以需要做对应的转换，比如把15转换成64，代表实际上有64位数据。在开头新增了字节数查询表的函数

#### 请提供验证的bsp和config (provide the config and bsp) 



- BSP:
\rt-thread\bsp\stm32\libraries\HAL_Drivers\drivers\drv_fdcan.c

- .config:

https://gitee.com/User_lwh/fes/tree/master/04_Software/04_STM32H7B0/project

- action:

]

<!-- 以下的内容不应该在提交PR时的message修改，修改下述message，PR会被直接关闭。请在提交PR后，浏览器查看PR并对以下检查项逐项check，没问题后逐条在页面上打钩。
The following content must not be changed in the submitted PR message. Otherwise, the PR will be closed immediately. After submitted PR, please use a web browser to visit PR, and check items one by one, and ticked them if no problem. -->

### 当前拉取/合并请求的状态 Intent for your PR

必须选择一项 Choose one (Mandatory):

- [ ] 本拉取/合并请求是一个草稿版本 This PR is for a code-review and is intended to get feedback
- [ ] 本拉取/合并请求是一个成熟版本 This PR is mature, and ready to be integrated into the repo

### 代码质量 Code Quality：

我在这个拉取/合并请求中已经考虑了 As part of this pull request, I've considered the following:

- [ ] 已经仔细查看过代码改动的对比 Already check the difference between PR and old code
- [ ] 代码风格正确，包括缩进空格，命名及其他风格 Style guide is adhered to, including spacing, naming and other styles
- [ ] 没有垃圾代码，代码尽量精简，不包含`#if 0`代码，不包含已经被注释了的代码 All redundant code is removed and cleaned up
- [ ] 所有变更均有原因及合理的，并且不会影响到其他软件组件代码或BSP All modifications are justified and not affect other components or BSP
- [ ] 对难懂代码均提供对应的注释 I've commented appropriately where code is tricky
- [ ] 代码是高质量的 Code in this PR is of high quality
- [ ] 已经使用[formatting](https://github.com/mysterywolf/formatting) 等源码格式化工具确保格式符合[RT-Thread代码规范](https://github.com/RT-Thread/rt-thread/blob/master/documentation/contribution_guide/coding_style_cn.md) This PR complies with [RT-Thread code specification](https://github.com/RT-Thread/rt-thread/blob/master/documentation/contribution_guide/coding_style_en.md)
- [ ] 如果是新增bsp, 已经添加ci检查到[.github/ALL_BSP_COMPILE.json](https://github.com/RT-Thread/rt-thread/blob/master/.github/ALL_BSP_COMPILE.json)  详细请参考链接[BSP自查](https://www.rt-thread.org/document/site/#/rt-thread-version/rt-thread-standard/development-guide/bsp-selfcheck/bsp_selfcheck)
