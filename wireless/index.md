# 无线协议学习笔录


## 概述 {#概述}


## 从一次正常的连接说起 {#从一次正常的连接说起}

先从一次简单的连接看起。测试环境就是一台无线 STA 和一个 AP，以及一台用来抓包的 PC 。

```sh

  +--------+               +-------+
  |  STA   |               |  AP   |
  +--------+               +-------+

             +---------+
             | capture |
             | PC      |
             +---------+

```

通过 PC 用 wireshark 来抓取 STA 和 AP 之间的无线数据报文。


### 管理报文 {#管理报文}

先看看管理报文

![Wireless Management Packets](/ox-hugo/wireless.img.5ee5486e.png "Wireless Management Packets")


### 数据报文 {#数据报文}


## 附录 {#附录}


### Linux 抓无线包 {#linux-抓无线包}


### Wireshark 解密无线包 {#wireshark-解密无线包}

