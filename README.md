# ReactNative Android通信原理

本文基于release 0.29,从源码角度剖析react native for android的java<=>js通信机制。

## 整体结构

rn分为三大模块,java层、js层和bridge,java和js通过bridge实现通信,由于java不能直接调用JavaScriptCore,bridge层由C/C++实现,整体结构图如下。<br>

![](pic/1.png)<br>

## 基本原理

结构图如下<br>

![](pic/2.png)<br>

先来看rn初始化的过程, java => js<br>

### 初始化bridge

![](pic/3.jepg)<br>