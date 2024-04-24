---
title: 一次Android自动化辅助工具分析
date: 2024-04-24T00:00:00Z
lang: zh
duration: 12min
description: 针对"aibote"自动化辅助工具的分析。
---

[[toc]]

近期有客户找我逆向一款软件的卡密，主要是一个针对某红薯软件的部分自动操作功能，辅助工具能自动以特定规则去回复别人/私聊。
![qt-person1](/images/aibote-reverse/qt-person1.png)

## 脱壳

拿到apk首先看有没有壳，这里因为客户已经告诉我了有壳。打开发现是某数字加固，那就直接脱壳把。
![jadx-360](/images/aibote-reverse/jadx-360.png)
脱壳后
![jadx-source](/images/aibote-reverse/jadx-source.png)

在仔细的阅读部分代码后，我发现这个工具可能也是使用第三方的自动化框架类写的，例如**AutoJS**，但是这个的代码很明显不是。

通过packageName我觉得可能可以去搜索一下**com.aibot.client**

![aibote-web](/images/aibote-reverse/aibote-web.png)

看样子基本确定是个框架，然后就可以去找到官网了。

## 分析

阅读部分代码以及结合客户给我发送的视频，软件会通过一个Host和Port然后就能使用脚本，和以往的辅助工具不同的是这款工具脚本逻辑竟然在服务端，有点意思。

可以先从页面按钮的点击事件开始下手，看看相关的代码逻辑。

![aibote-start-btn](/images/aibote-reverse/aibote-start-btn.png)

```java
FrameProtocol.setUpSocket(ScriptActivity.frameServerIp, i);
```

主要还是这一行，将用户输入的Host和Port进行了连接，这里猜测是**TCP**通讯。

点进去就看到了是TCP了，那这里可以大胆的猜测是tcp传输脚本指令了。

```java
public static void setUpSocket(String str, int i) {
  try {
    mSocket = new Socket();
    mSocket.connect(new InetSocketAddress(str, i), 5000);
    if (mSocket.isConnected()) {
      mSocketOS = mSocket.getOutputStream();
      mSocketIS = mSocket.getInputStream();
      isClose = false;
      isSaveBitmap = true;
    }
  } catch (IOException e) {
    e.printStackTrace();
    try {
      Socket socket = mSocket;
      if (socket != null) {
          socket.close();
      }
    } catch (IOException e2) {
      e2.printStackTrace();
    }
    isClose = true;
  }
}
```

## 抓包

分析完上述操作后可以抓包验证下，这里就使用**Wireshark**进行分析。

开启Wireshark，打开助手启动脚本，停止抓包。

![qt-home](/images/aibote-reverse/qt-home.png)
筛选下IP
![wireshark-ip](/images/aibote-reverse/wireshark-ip.png)

通过阅读抓包数据，看到一个服务端发送过来的数据。

![wireshark-getAndroidId](/images/aibote-reverse/wireshark-getAndroidId.png)

有一个getAndroidId，查阅官网发现有相应的指令。

![aibote-androidId](/images/aibote-reverse/aibote-androidId.png)

![aibote-hex](/images/aibote-reverse/aibote-hex.png)

根据官网的给出的数据格式，我们就可以推断接下来的一些数据了。

![wireshark-textview](/images/aibote-reverse/wireshark-textview.png)

```
14/2/9/1/3/3/3createTextView20xxxx0120200100
```

这段翻译过来就是<br>
14 = **createTextView** 14个字符长度<br>
2 = createTextView后面的ID20，也就是2个字符长度<br>
9 = 这里先让大家猜想一下，这段其实是创建一个内容为"注册码"的文本框，但是为什么是9的长度呢？<br>
1 = 左上角坐标x，所以得出这里的值应该是0<br>
3 = 左上角坐标y，长度3，所以是0后面的3位，也就是120<br>
3 = 右上角x,200<br>
3 = 右上角y,100<br>

上述的坐标也可能是宽高信息之类的，具体不太重要没有详细去判断。

e6b3a8e5868ce7a081

再次回到上面的9长度，到底是啥呢。其实是byte数据。

![java](/images/aibote-reverse/java.png)
