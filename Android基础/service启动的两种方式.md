#### service启动的两种方式

1.startService()

生命周期：

onCreate()  ---> onStartConmmad() --> onDestroy()

调用者与service之间没有绑定关系， 调用者退出， service不一定退出， 除非显式的调用stopService() 才会停止onDestroy；多次启动service， 如果service未创建，则创建service 走onCreate方法， 如果service已启动，则会调用onStartConmmad方法。

2.bindService()

生命周期：

onCreate() --> onBind() -->onUnbind() --> onDestroy()

调用者和service之前存在绑定关系，调用者退出， service也会直接退出；多次调用bindService(),service不会多次创建， 如果service未创建， 则创建service 并onBindService ，如果已创建， 则调用onBindService方法。