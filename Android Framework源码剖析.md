# Android Framework源码剖析

## Zygote进程

1. 了解Zygote的作用
2. 熟悉Zygote的启动流程
3. 深刻理解Zygote的工作原理

### Zygote的作用

+ 启动SystemServer（常用类、JNI函数、主题资源、共享库直接从Zygote继承，不需要自己重新加载，提升性能 ）
+ 孵化应用进程

#### 启动三段式

**进程启动 - 准备工作 - LOOP**

### Zygote的启动流程

1. 进程是怎么启动的？
2. 进程启动之后做了什么？

**init进程通过 init.rc（启动配置文件）知道启动Zygote**

#### 启动进程

+ fork + handle
+ fork + execve 子进程重新加载一个新的二进制程序，继承的父进程二进制资源会被清理

####  信号处理 - SIGCHLD

父进程fork出子进程， 如果子进程挂了， 父进程会收到**SIGCHLD**信号可以去做一些处理

###  Zygote进程启动之后做了什么？

#### Native世界

1. **启动Android虚拟机**
2. **注册Android 的JNI函数**
3. **进入Java世界**

#### Java世界

1. **预加载资源 Preload Resources**
2. **fork启动 System Server**
3. **进入Loop循环**

**Tips：**

1. Zygote fork要单线程
2. Zygote的IPC没有采用binder，使用本地Socket

 **Q：**

1. 孵化应用进程这种事为什么不交给SystemServer来做， 而专门设计一个Zygote
2. Zygote的IPC通信机制为什么不采用binder？如果采用binder的话会有什么问题？

## Android系统的启动

1. Android有哪些主要的系统进程？
2. 这些系统进程是怎么启动的？
3. 进程启动之后主要做了些什么事？

### 系统进程

+ zygote
+ servicemanager
+ surfaceflinger
+ media
+ . . . 

#### Zygote是如何启动的

+ init进程fork出zygote进程
+ 启动虚拟机， 注册jni函数
+ 预加载系统资源
+ 启动SystemServer
+ 进入Socket Loop

#### SystemServer是怎么启动的？

Q：

+ 系统服务是怎么启动的？
  + 系统服务怎么发布， 让应用程序可见？ 
  + 系统服务跑在什么线程？
  + 为什么系统服务不都跑在binder线程里？
  + 为什么系统服务不都跑在自己私有的工作线程里呢？
  + 泡在binder线程和跑在工作线程， 如何取舍

+ 怎么解决系统服务之间的相互依赖？

  分批启动 AMS PMS PKMS

  分阶段启动 

#### 桌面的启动

### Android启动流程

+ zygote是怎么启动的？
+ systemServer是怎么启动的？
+ 系统服务是怎么启动的？

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4fda786855a4e6bbf007af3bf6c3655~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8665a516d0744678834affffeb4f3c7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 如何添加一个系统服务

+ 了解如何使用系统服务
+ 了解系统服务调用的基本原理
+ 了解服务的注册原理

### 如何使用系统服务

### 如何注册系统服务

### 什么时候注册的系统服务

### 独立进程的系统服务

### 启用binder机制

+ 打开binder驱动
+ 映射内存，分配缓冲区
+ 启动binder线程，进入binder loop

## 系统服务和bind的应用服务有什么区别

+ 启动方式上有什么区别
+ 注册方式上有什么区别
+ 使用方式上有什么区别

### 启动方式上有什么区别

#### 系统服务的启动

#### 应用服务的启动

### 注册方式上有什么区别

#### 系统服务的注册

#### 应用服务的注册

### 使用方式上有什么区别

#### 系统服务的使用

#### 应用服务的使用

## ServiceManager的启动和工作原理

+ ServiceManager启动流程是怎样的
+ 怎么获取ServiceManager的binder对象
+ 怎么向ServiceManager添加服务
+ 怎么从ServiceManager获取服务

### ServiceManager的启动

+ 启动进程
+ 启动binder机制
+ 发布自己的服务
+ 等待并响应请求

## 应用进程如何启动的

+ 了解Linux下进程启动的方式
+ 熟悉应用进程启动的基本流程
+ 深入理解应用进程启动的原理

### 应用进程启动原理

#### 什么时候出发的进程启动？谁发起的？

+ 进程是谁启动的？怎么启动的？

## 应用是怎么启动binder机制

+ 了解binder是用来干什么的？
+ 应用里面哪些地方用到了binder机制？
+ 应用的大致启动流程是怎么样的？
+ 一个进程是怎么启动binder机制的？

### 怎么启动binder机制

1. 打开binder驱动
2. 映射内存，分配缓冲区
3. 注册binder线程
4. 进入binder loop

## 对Application的理解

+ 了解Application的作用
+ 熟悉Application的类继承关系以及生命周期
+ 深入理解Application的初始化原理

### Application有什么作用

+ 保存应用进程内的全局变量
+ 初始化操作
+ 提供应用上下文

### Application的生命周期

+ 构造函数
+ attachBaseContext
+ onCreate

### Application的初始化

## 对Context的理解

+ 了解Context的作用
+ 熟悉Context初始化流程
+ 深入理解不同应用组件之间的Context的区别

**Q：**

1. 应用里面有多少个Context？不同的Context之间有什么区别？
2. Activity里的this和getBaseContext有什么区别？
3. getApplication和getApplicationContext有什么区别？
4. 应用组件的构造，onCreate、attachBaseContext调用顺序？

