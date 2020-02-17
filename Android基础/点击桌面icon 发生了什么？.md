点击桌面icon 发生了什么？

Launcher发生事情可以先不谈， 单说App的首次启动流程， 首先Linux会通过Zygote  fork 出一个新的进程供App 使用， 然后是main 方法的准备工作：①Looper初始化②ActivityThread 的创建③Looper开始循环  并接受四大组件各种生命周期事件 并回调对应组件的生命周期方法， 直至MainActivivty 走到  onCreate  onStart onResume 用户看到页面；





