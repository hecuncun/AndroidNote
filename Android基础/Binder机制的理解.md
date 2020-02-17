#### Binder机制的理**解**

1.binder是一种Android  中IPC（进程间通信）方式；

2.binder 通信采用C/S架构，客户端进程通过获取到服务器端进程的代理，并通过这个代理接口方法中读写数据来实现进程间的数据传递。

3.AIDL是binder的实例 ， 也是binder的应用。

**binder机制的组成：**

1.client

2.server

3.serviceManager

流程：server端先向serviceManager中注册service ---》 serviceManager负责管理这些service并向client提供接口和服务 ---》 client要与那个service进行通信，先从serviceManager中获取到该service相关信息， 建立通信 然后才能交互。

