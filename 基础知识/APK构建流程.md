#### apk 构建流程

###### Android是如何构建一个APK的?

![](https://upload-images.jianshu.io/upload_images/3606778-b978b976f8d0f591.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/486/format/webp)

流程概述：

```
工程的资源文件(res文件夹下的文件)，通过AAPT打包成R.java类(资源索引表)，以及.arsc资源文件
如果有aidl，通过aidl工具，打包成java接口类
R.java和aidl.java通过java编译成想要的.class文件。
源码class文件和第三方jar或者library通过dx工具打包成dex文件。dx工具的主要作用是将java字节码转换成Dalvik字节码，在此过程中会压缩常量池，消除一些冗余信息等。
apkbuilder工具会将所有没有编译的资源，.arsc资源，.dex文件打包到一个完成apk文件中中。
签名，5中完成apk通过配置的签名文件(debug和release都有)，jarsigner工具会对齐签名。得到一个签名后的apk,signed.apk
zipAlign工具对6中的signed.apk进行对齐处理，所谓对齐，主要过程是将APK包中所有的资源文件距离文件起始偏移为4字节整数倍，这样通过内存映射访问apk文件时的速度会更快。对齐的作用主要是为了减少运行时内存的使用。
```

总结:

输入：res文件夹所有的资源(layout\drawable\string\array等)，asset下的资源，AndroidManifest.xml，Android.jar文件
工具： aapt 地址(/your sdk path/build-tools/your build tools version/aapt)
输出：res下的资源都会被编译成一个资源索引文件resource.arsc以及一个R.java类。asset下的资源不会编译，直接压缩进apk。
