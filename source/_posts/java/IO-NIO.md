---
title: Java IO/NIO 对比
date: 2020-04-15 15:59:27
tags:
    - java
    - io
    - nio
categories:
    - java
    - IO/NIO
---
#### JAVA IO VS NIO
* JDK 1.4 之前，java.io 包，面向流的I/O系统（字节流或者字符流）
  * 系统一次处理一个字节
  * 处理速度慢
* JDK 1.4 提供，java.nio 包，面向块的I/O系统
  * 系统一次处理一个块
  * 处理速度快