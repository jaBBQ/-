## Glide笔记

Glide基本概念

1. Model：表示数据来源
2. Data：从数据源中加工成Data
3. Resource：对于解码后的资源
4. TransformedResource：转换后的资源
5. TranscodedResource：转码完成后的资源
6. Target：要显示的目标



 加载参数

1. load 指定图片的URL
2. placeholder 指定图片未成功加载前显示的图片
3. error 指定图片加载失败显示的图片
4. override 指定图片的尺寸
5. fitCenter、centerCrop 指定图片缩放类型
6. skipMemoryCache 跳过内存缓存
7. diskCacheStrategy 磁盘缓存策略
8. priority 指定优先级， Glide将会作为一个准则， 尽可能处理请求， 但不保证一定按照优先级展示
9. into 指定显示图片的imageview



with方法

基础工作， 获取RequestManager来管理请求  ， 绑定组件生命周期， 来进行响应处理



load方法

做一些初始化操作， 获得DrawableTypeRequest对象， 通过该对象可以获得图片请求的Request



Glide缓存

内存

防止重复将图片读取到内存当中

1. LRUCache
2. 弱引用



硬盘

防止重复从网络、硬盘复制下载图片