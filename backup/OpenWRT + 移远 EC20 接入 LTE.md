## 前言

在 [OpenWRT Stable Release 编译 ](https://blog.motofans.club/post/6.html) 这篇博文中我提到了我在使用 EC20 实现一个 CPE 设备。

在本篇文章中，将会介绍如何在 OpenWRT 中使用 EC20 进行联网以及接收短信。

EC20 有多个版本，包括不限于下面这张图片（截取自京东移远旗舰店）。



我不想花费太多的时间调查网络上流传的各种版本哪一个更强大，以及强大在哪里，所以我从上面选了一个能够满足我使用需求的版本（访问网络和接收短信），即 CEFHLG-128-SNNS，这个版本除了不支持 GPS 看起来都 OK，它在二手市场上卖 40 人民币。

## 访问网络

https://openwrt.org/docs/guide-user/network/wan/wwan/ltedongle

<!-- ##{"timestamp":1696953600}## -->