### APP方法数超过64K导致编译报错问题解释：

##### 原因:

当我们的应用及所引用的库超过65536种方法时，会遇到一个编译错误，这是由于Android编译架构规定的引用限制所导致的。

##### 概述：

1. 如果 minSdkVersion 大于21 也就是Android5.0的时候 默认支持多dex文件；
2. 如果minSdkVersion小于21 也就是Android5.0以下的时候存在这个错误；

##### 解决方案：

1. 修改模块级build.gradle文件以启用多dex文件，并将多dex文件库添加为依赖项，核心配置如下：

   ```java
       android {
           defaultConfig {
               ...
               minSdkVersion 15
               targetSdkVersion 28
               multiDexEnabled true
           }
           ...
       }
   
       dependencies {
         //support包  
         implementation 'com.android.support:multidex:1.0.3'
         //androidx
         implementation 'androidx.multidex:multidex:2.0.1'
       }
       
   ```

2. 将Application配置成MultiDexApplication的子类

   - 在manifest.xml中配置<application>的android:name 属性 如下所示：

     ```xml
         <?xml version="1.0" encoding="utf-8"?>
         <manifest xmlns:android="http://schemas.android.com/apk/res/android"
             package="com.example.myapp">
             <application
                     android:name="android.support.multidex.MultiDexApplication" >
                 ...
             </application>
         </manifest>
         
     ```

   - 自定义自己的Application类 并将MyApplication 继承自MultiDexApplication 如下所示：

     ```java
         public class MyApplication extends MultiDexApplication { ... }
     ```

   - 如果应用已经继承自其他（第三方）Application了 ，可以在Application的attachBaseContext()方法中调用MultiDex.install(this)来启用多dex文件的支持：

     ```java
         public class MyApplication extends SomeOtherApplication {
           @Override
           protected void attachBaseContext(Context base) {
              super.attachBaseContext(base);
              MultiDex.install(this);
           }
         }
     ```

     
