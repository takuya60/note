# Window
- 宽度 高度
- maxmumHeight最大高度    maxmumWidth最大宽度
- minmumHeight    minmumWidth
- 标题
```cpp
titel:qsTr("title")
  ```
- visable 可见性
- x，y 相对于父控件的位置
- opcity 透明度
# Button
```cpp
Button{
id:btn1
x:100
y:100
width:20
height:40
objectname:btn1
background:rectangle{
	border.color : btn1.focus ? "blue" :"black"
	}
onClicked:{

}
}
```
# Componet
