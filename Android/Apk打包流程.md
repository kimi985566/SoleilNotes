## 1 APK 打包过程

![](../asset/apk签名过程.jpg)

* 打包资源文件，生成 R.java 文件
  * aapt 工具（aapt.exe） -> AndroidManifest.xml 和 布局文件 XMl 都会编译 -> R.java -> AndroidManifest.xml 会被 aapt 编译成二进制
  * res 目录下资源 -> 编译，变成二进制文件，生成 resource id -> 最后生成 resouce.arsc（文件索引表）
  
* 处理 aidl 文件，生成相应的 Java 文件
  
  * aidl 工具（aidl.exe）
  
* 编译项目源代码，生成 class 文件

* 转换所有 class 文件，生成 classes.dex 文件
  
  * dx.bat
  
* 打包生成 APK 文件
  
  * apkbuilder 工具打包到最终的 .apk 文件中
  
* 对APK文件进行签名

* 对签名后的 APK 文件进行对齐处理（正式包）
  
  * 对 APK 进行对齐处理，用到的工具是 zipalign

## 什么是Dex文件

- JVM 是 JAVA 虚拟机，用来运行 JAVA 字节码程序。
- Dalvik 是 Google 设计的用于 Android平台的运行时环境，适合移动环境下内存和处理器速度有限的系统。
- ART 即 Android Runtime，是 Google 为了替换 Dalvik 设计的新 Android 运行时环境，在Android 4.4推出。ART 比 Dalvik 的性能更好。
- Android 程序一般使用 Java 语言开发，但是 Dalvik 虚拟机并不支持直接执行 JAVA 字节码，所以会对编译生成的 .class 文件进行翻译、重构、解释、压缩等处理，这个处理过程是由 dx 进行处理，处理完成后生成的产物会以 .dex 结尾，称为 Dex 文件。
- Dex 文件格式是专为 Dalvik 设计的一种压缩格式。所以可以简单的理解为：Dex 文件是很多 .class 文件处理后的产物，最终可以在 Android 运行时环境执行。