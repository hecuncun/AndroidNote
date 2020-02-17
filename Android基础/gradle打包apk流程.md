#### gradle打包apk流程：

1.JAVA编译器将项目代码和依赖的第三方库代码编译成.class文件；

2..class文件和三方库文件通过dex工具生成可执行的.dex文件；

3.apkBuilder将.dex文件和编译后的资源文件生成未签名的apk文件；

4.JarSigner和ZipAlign对文件进行签名和对齐，生成最终的apk文件；

