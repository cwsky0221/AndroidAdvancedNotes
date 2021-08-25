#### Gradle,Gradle Wrapper,Android Plugin for Gradle的区别

>Gradle是个构建系统，能够简化你的编译、打包、测试过程。熟悉Java的同学，可以把Gradle类比成Maven。
Gradle Wrapper的作用是简化Gradle本身的安装、部署。不同版本的项目可能需要不同版本的Gradle，手工部署的话比较麻烦，而且可能产生冲突，所以需要Gradle Wrapper帮你搞定这些事情。Gradle Wrapper是Gradle项目的一部分。
Android Plugin for Gradle是一堆适合Android开发的Gradle插件的集合，主要由Google的Android团队开发，Gradle不是Android的专属构建系统，但是有了Android Plugin for Gradle的话，你会发现使用Gradle构建Android项目尤其的简单。**Plugin for Gradle 相当于是基于Gradle开发的应用层插件，用于android项目构建的**



另外需要说明的一点是Gradle、Gradle Wrapper与Android Plugin for Gradle不一定要和Android Studio一起使用，你可以完全脱离Android Studio，使用三者独立进行Android项目的构建

下面说下实际的项目构建中，这三者的关系，当我们新建一个Android项目时，会自动创建一个gradle/wrapper目录，其中有两个文件：gradle-wrapper.jar/gradle-wrapper.properties，gradle-wrapper.jar是Gradle Wrapper的主体功能包。在Android Studio安装过程中产生gradle-wrapper.jar（如果默认安装的话会在C:\Program Files\Android\Android Studio\plugins\android\lib\templates\gradle\wrapper\gradle\wrapper\gradle-wrapper.jar）。然后每次新建项目，会将gradle-wrapper.jar拷贝到你的项目的gradle/wrapper目录中。gradle-wrapper.properties文件主要指定了该项目需要什么版本的Gradle，从哪里下载该版本的Gradle，下载下来放到哪里，如下图所示：
![](https://images2017.cnblogs.com/blog/611264/201801/611264-20180107182326299-1882484816.png)

**Gradle对应版本下载完成之后，Gradle Wrapper的使命基本完成了，**Gradle会读取build.gradle文件，该文件中指定了该项目需要的Android Plugin for Gradle版本是什么，从哪里下载该版本的Android Plugin for Gradle。如下图所示：
![](https://images2017.cnblogs.com/blog/611264/201801/611264-20180107184419674-1459842824.png)
