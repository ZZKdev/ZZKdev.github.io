---
title: 硬件hacker用digispark制作BadUSB
date: 2018-08-28 22:23:01
tags: [硬件, 安全]
---
# BadUSB是什么？
BadUSB是一种通过重写U盘固件伪装成USB输入设备（键盘，网卡）用于恶意用途的usb设备，这种攻击方式十分猥琐，杀毒软件由于无法接触到usb固件，所以对它也无可奈何，你总不能拒绝你的键盘吧。。。

![](https://upload-images.jianshu.io/upload_images/10461798-5bea83b4d5667284.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 使用工具
1. digispark 开发板（可以再某宝上购买9块一个）

![](https://upload-images.jianshu.io/upload_images/10461798-6c71aac2ee566bec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. arduino IDE
3. digispark驱动
# 步骤
## 首先把我们的开发工具下好
[arduino IDE下载](https://www.arduino.cc/en/Main/Software)
[digispark驱动下载](https://github.com/digistump/DigistumpArduino/releases)
## 然后在arduino IDE加入对digispark开发板的支持
在文件->首选项中加入附加开发板管理
``http://digistump.com/package_digistump_index.json``
![](https://upload-images.jianshu.io/upload_images/10461798-d95baecd41f7496b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在工具->开发板中打开开发板管理器，找到Digistump AVR Boards，然后安装它！这个安装过程可能会有点久如果不开代理的话( >﹏<。)～

![](https://upload-images.jianshu.io/upload_images/10461798-523bc692904287d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 现在就可以开始coding啦！
其实还没有。。还差一点点，请在工具中选中开发板Digispark (Default - 16.5mHz)┌(ㆆ㉨ㆆ)ʃ
现在可以coding了 ʅ(‾◡◝)ʃ 

![](https://upload-images.jianshu.io/upload_images/10461798-9091afd17ab43493.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在coding之前先解释一下在arduino中的编程，它一开始默认会有两个函数一个是setup()一个是loop()，在插入我们的开发板后，它会执行一次setup()函数，然后循环执行loop()函数。对了还有arduino是使用c语言进行编程的(๑¯◡¯๑)
现在将我们的代码写上

```
#include "DigiKeyboard.h"
void setup() {
  // put your setup code here, to run once:
  DigiKeyboard.delay(2000);
  DigiKeyboard.sendKeyStroke(KEY_R, MOD_GUI_LEFT);
  DigiKeyboard.delay(300);
  DigiKeyboard.println("cmd");
  DigiKeyboard.delay(300);
  DigiKeyboard.println("echo hacked");
}

void loop() {
  // put your main code here, to run repeatedly:

}
```

点击验证一下看下有没有错误，虽然我知道没有错误(๑¯◡¯๑)

![](https://upload-images.jianshu.io/upload_images/10461798-6340ff80a9f7c516.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

太棒了！没有错误no error, no warning（其实我没有试）
好我们继续
点击上传按钮

![](https://upload-images.jianshu.io/upload_images/10461798-7d9338a09aecc434.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果没有错误的话，然后它会在下方显示

![](https://upload-images.jianshu.io/upload_images/10461798-f4e2431f0527146e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后请在60秒之内插入你准备好的digispark开发板（如果之前就插入了就再插一遍），没问题的话它就会谢谢你(̿▀̿ ̿Ĺ̯̿̿▀̿ ̿)，就像这样

![](https://upload-images.jianshu.io/upload_images/10461798-d5281956396d570f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在插入digispark开发板后，它会自动打开cmd，并打印一句hacked！

![](https://upload-images.jianshu.io/upload_images/10461798-e3ae17eaa6dcec73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在我们已经制作完成了我们的badUSB，虽然它现在并不具备什么攻击功能⚆_⚆，但是我们只要对代码进行简单的修改就可以做到植入后门啊，拿wifi密码啊，开远程端口啊什么的，具体的代码在这里就不提供了，不过可以到下面链接查看攻击代码并尝试翻译成arduino的代码（也可以用脚本翻译，具体操作大家搜索一下就有了）（简单点你可以把代码中的“cmd"，换成"shutdown /s /t 1"再把下面两行删除，就可以实现插上去电脑关机的功能啦）
https://github.com/hak5darren/USB-Rubber-Ducky/wiki/Payloads

搞完之后可以去3d打印给你的badUSB换一个好看的外壳，毕竟光一块板子直接插你电脑上一看就很可疑Ծ‸Ծ

# 总结
以后在外面捡到u盘不要随便往电脑上插！！(´°̥̥̥̥̥̥̥̥ω°̥̥̥̥̥̥̥̥｀)。甚至有可能插完之后你的电脑就报废了（如果是usbkill的话）

