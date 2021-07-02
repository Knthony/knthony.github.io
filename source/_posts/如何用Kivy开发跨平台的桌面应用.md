---
title:  如何用Kivy开发跨平台的桌面应用
date: 2021/4/8 12:04:23
categories:
- Tech
tags:
- python
---

### Kivy 
Kivy 是可以构建跨平台（IOS，MACOS，Android，Windows）桌面应用的python框架。


> 具体介绍请看官方文档 https://kivy.org/


### 安装Kivy
1. 首先在创建一个python的虚拟环境。
   `python3 -m venv env`

2. 用pip安装
    `pip  install  -i  https://pypi.doubanio.com/simple/  --trusted-host pypi.doubanio.com kivy`


### 创建一个简单的应用

```
from typing import Mapping
from kivy.app import App
from kivy.uix.widget import Widget

class PongGame(Widget):
    pass

class PongApp(App):
    def build(self):
        return PongGame()

if __name__ == '__main__':
    PongApp().run()
```

运行上面的程序，我们完成了一个Kivy的APP，包含了一Widget类的实例，通过这个实例得到应用UI的根元素，会得到下图中的黑色背景的窗口，

![](https://ftp.bmp.ovh/imgs/2021/04/41fb17e0d4ad3c11.png)


### 添加简单的制图样式

我们需要用`.kv`后缀的文件来定义应用的样式，由于我们的程序类名是`PongApp`，在当前目录下创建`pong.kv`

```
#:kivy 1.0.9

<PongGame>:    
    canvas:
        Rectangle:
            pos: self.center_x - 5, 0
            size: 10, self.height
            
    Label:
        font_size: 70  
        center_x: root.width / 4
        top: root.top - 50
        text: "0"
        
    Label:
        font_size: 70  
        center_x: root.width * 3 / 4
        top: root.top - 50
        text: "0"
```
> 需要注意的是，kv文件的文件名一定要和APP的类名一致

再次运行程序，会看到中间有一条白线分割窗口，并且两边分别有一个显示分数的0

![](https://ftp.bmp.ovh/imgs/2021/04/d0e44328f2bca881.png)


