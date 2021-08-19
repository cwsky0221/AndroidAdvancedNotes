
### Android targetSdkVersion 原理

通常来说，一般Android项目都有三个配置项，compileSdkVersion、minSdkVersion 以及 targetSdkVersion。

compileSdkVersion 和 minSdkVersion 都非常好理解，
compileSdkVersion表示编译的 SDK 版本，一般用最新的sdk版本，这样我们可以使用到最新SDK的API方法和机制，用于编译器编译时检测，比如检测一些被弃用的api，不打入apk包中。
minSdkVersion表示应用兼容的最低 SDK 版本，低于这个版本的系统上不能安装应用

##### targetSdkVersion怎么理解

官方说法：targetSdkVersion 是 Android 系统提供前向兼容的主要手段，这是什么意思呢？随着 Android 系统的升级，某个系统的 API 或者模块的行为可能会发生改变，但是为了保证老 APK 的行为还是和以前兼容。只要 APK 的 targetSdkVersion 不变，即使这个 APK 安装在新 Android 系统上，其行为还是保持老的系统上的行为，这样就保证了系统对老应用的前向兼容性。

官方的例子，在 Android 4.4 (API 19）以后，AlarmManager 的 set() 和setRepeat() 这两个 API 的行为发生了变化。在 Android 4.4 以前，这两个 API 设置的都是精确的时间，系统能保证在 API 设置的时间点上唤醒 Alarm。因为省电原因 Android 4.4 系统实现了 AlarmManager 的对齐唤醒，这两个 API 设置唤醒的时间，系统都对待成不精确的时间，系统只能保证在你设置的时间点之后某个时间唤醒。

这时，虽然 API 没有任何变化，但是实际上 API 的行为却发生了变化，如果老的 APK 中使用了此 API，并且在应用中的行为非常依赖 AlarmManager 在精确的时间唤醒，例如闹钟应用。如果 Android 系统不能保证兼容，老的 APK 安装在新的系统上，就会出现问题。

Android 系统是怎么保证这种兼容性的呢？这时候 targetSdkVersion 就起作用了。APK 在调用系统 AlarmManager 的 set() 或者 setRepeat() 的时候，系统首先会查一下调用的 APK 的 targetSdkVersion 信息，如果小于 19，就还是按照老的行为，即精确设置唤醒时间，否者执行新的行为。

我们来看一下 Android 4.4 上 AlarmManger 的一部分源代码：

```
private final boolean mAlwaysExact;  
AlarmManager(IAlarmManager service, Context ctx) {  
    mService = service;
    final int sdkVersion = ctx.getApplicationInfo().targetSdkVersion;
    mAlwaysExact = (sdkVersion < Build.VERSION_CODES.KITKAT);
}
```

看到这里，发现其实 Android 的 targetSdkVersion 并没有什么特别的，系统使用它也非常直接,仅仅是用过下面的 API 来获取 targetSdkVersion，来判断是否执行哪种行为：
```
getApplicationInfo().targetSdkVersion;
```
所以，如果 Android 系统升级，发生这种兼容行为的变化时，一般都会在原来的保存新旧两种逻辑，并通过 if-else 方法来判断执行哪种逻辑

那么，你会不会问：假如set() 和setRepeat() 这两个 API 在Android 11(API 30)上面有发生变化了呢？

很明显，这个api的源码中会有类似的判断：
```
private final boolean mAlwaysExact;  
AlarmManager(IAlarmManager service, Context ctx) {  
    mService = service;
    final int sdkVersion = ctx.getApplicationInfo().targetSdkVersion;
    mAlwaysExact = (sdkVersion < Build.VERSION_CODES.KITKAT);
    if(sdkVersion < Build.VERSION_CODES.KITKAT){
        //小于19，用最原始的逻辑
    }else if(sdkVersion < Build.VERSION_CODES.R){//小于30
        //大于19，小于30，用中间修改的逻辑
    }else{//大于等于30
        //大于等于30，用最新修改的逻辑
    }
}
```
所以，所以一般minSdkVersion <targetSdkVersion<= compileSdkVersion，咱们的应用如果在某个特定的targetSdkVersion下测试充分，就基本不用担心兼容问题，如果改动了targetSdkVersion，就需要做完整测试了

但是还有一个问题：

Android 6.0新增加了动态权限申请，我们的targetSdkVersion是5.0，如果我们运行在Android 6.0的设备上怎么办？
这个不是api的变动，如果你的代码里处理了Android 6.0的动态权限处理，那么可以的，没有如果，需要更新应用处理了







