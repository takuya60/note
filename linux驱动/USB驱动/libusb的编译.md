为了使用usb，我们直接在应用层进行开发就行，使用的是libusb库
首先，我们需要在buildroot里面勾选上libusb
然后实际写代码的时候，vscode的配置要进行修改
```c
{
    "configurations": [
        {
            "name": "RK3506-Linux-App",
            "includePath": [
                "${workspaceFolder}/**",
                "/home/alientek/rk3506_linux6.1_sdk_v1.2.0/buildroot/output/latest/staging/usr/include"
            ],
            "defines": [],
            "compilerPath": "/home/alientek/rk3506_linux6.1_sdk_v1.2.0/buildroot/output/latest/host/bin/arm-buildroot-linux-gnueabihf-gcc",
            "cStandard": "c17",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-arm64"
        }
    ],
    "version": 4
}
```
之前的define，include和编译器，是开发内核的时候使用的，现在是应用层开发。
这里的include和编译器使用buildroot的