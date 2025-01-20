---
title: 成功把门禁的 NFC UID 复制到手机
date: 2017-03-01 00:35:10
tags: [NFC, Android]
categories: [Tech Notes]
authors: [alexchen]

---

其实就是照着 [如何利用Nexus 5伪造一张门禁卡](http://www.freebuf.com/news/topnews/80368.html) 和 [修改NFC手机的默认UID](http://www.enjoydiy.com/3539.html) 搞就行。
楼层里发现原始来源：[http://stackoverflow.com/questions/28409934/editing-functionality-of-host-card-emulation-in-android](http://stackoverflow.com/questions/28409934/editing-functionality-of-host-card-emulation-in-android)，厉害了

最近因为要提取 WIFI 密码而把手机 root 了，也用了有些年了，保修早就过了。正好手上有个门禁卡，也刚好再需要多一个，就把留在脑袋的一丝印象捞出来玩玩。

第一次漏了长度字节没加上，不成功之余，手机的 NFC 功能也不能工作，一度以为不兼容搞不定。后来发现后加上漏掉的一字节就好了。

虽然只能复制 Mifare M1 的 UID，但在我的情况里就已经够用了。其实x宝也是三四十块钱就能买一个简易的复制工具（需要配套可写 UID 的 NFC Tag）。
