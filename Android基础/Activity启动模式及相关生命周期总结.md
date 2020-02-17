### Activity启动模式及相关生命周期总结

#### 启动模式：

1.standard(默认启动模式)：每次启动activity， 无论栈里是否有这个activity的实例， 都会创建新的实例

2.singleTop：  启动activity的时候 会去检查栈顶是否是这个activity的实例， 如果栈顶是这个activity的实例 则直接复用 否则创建新的activity实例；

3.singleTask: 启动activity的时候 会去检查栈内是否有这个activity的实例， 如果有 则会清除此activity实例之上其他activity实例 将此activity实例至于栈顶并复用， 如果没有 则创建新的activity实例；

4.singleInstance： 启动activity的时候 会先创建一个新的任务栈， 然后在这个任务栈中创建此activity的实例， 以后再次启动这个activity的时候 则会直接复用之前的实例

activity生命周期：

onCreate()  onStart()  onResume()  onPause() onStop() onDestroy()

启动A

onCreate()  onStart()  onResume()

A启动B

ActivityA onPause()

ActivityB onCreate()

ActivityB onStart()

ActivityB onResume()

ActivityA onStop()

B返回A

ActivityB onPause()

ActivityA onRestart()

ActivityA onStart() 

ActivityA onResume()

ActivityB onStop()

ActivityB onDestroy()

此时如果按back键：

ActivityA onPause() --> onStop() --> onDestroy()

此时如果按home键：

ActivityA onPause() --> onStop()   再次通过任务栏打开app  ActivityA onRestart() --> onStart() --> onResume()

切换横竖屏的时候activity 生命周期:

onPause -->onSaveInstanceState(保存数据) -->onStop -->onDestroy -->onCreate-->onStart -->

onRestoreInstanceState(恢复数据)-->onResume -->onPause -->onStop -->onDestroy 

7.0以上还会首先回调onConfigurationChanged() 方法；

#### Activity的启动模式关联的还有TaskAffinity (manifest种配置的):

通常一个应用一般只有一个栈，进入或退出activity都会导致栈结构的push或pop，默认的栈名称是包名；

但是还是可以给单独的activity 指定特定的栈， 方式是通过在manifest种给activity 设置taskAffinity属性， taskAffinity代表着activity所处的栈， 这样子 就可以实现一个应用多个栈结构  配合着activity的launchMode可以实现多种进出activity模式。

