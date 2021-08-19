## ANR触发机制

ANR是一套监控Android应用响应是否及时的机制，可以把发生ANR比作是引爆炸弹，那么整个流程包含三部分组成：

埋定时炸弹：中控系统(system_server进程)启动倒计时，在规定时间内如果目标(应用进程)没有干完所有的活，则中控系统会定向炸毁(杀进程)目标。
拆炸弹：在规定的时间内干完工地的所有活，并及时向中控系统报告完成，请求解除定时炸弹，则幸免于难。
引爆炸弹：中控系统立即封装现场，抓取快照，搜集目标执行慢的罪证(traces)，便于后续的案件侦破(调试分析)，最后是炸毁目标。
常见的ANR有service、broadcast、provider以及input

Service Timeout:比如前台服务在20s内未执行完成，后台服务Timeout时间是前台服务的10倍，200s；
BroadcastQueue Timeout：比如前台广播在10s内未执行完成，后台60s
ContentProvider Timeout：内容提供者,在publish过超时10s;
InputDispatching Timeout: 输入事件分发超时5s，包括按键和触摸事件。

来简单分析下源码，ANR触发流程其实可以比喻成埋炸弹和拆炸弹的过程，
以后台Service为例
#### 埋炸弹
Context.startService
调用链如下：
AMS.startService 
ActiveServices.startService  
ActiveServices.realStartServiceLocked  
ActiveServices.realStartServiceLocked

```
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
    ...
    //1、这里会发送delay消息(SERVICE_TIMEOUT_MSG)
    bumpServiceExecutingLocked(r, execInFg, "create");
    try {
        ...
        //2、通知AMS创建服务
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
    } 
    ...
}

```
注释1的bumpServiceExecutingLocked内部调用scheduleServiceTimeoutLocked
```
    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        ...
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        // 发送deley消息，前台服务是20s，后台服务是10s
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }

```

注释2通知AMS启动服务之前，注释1处发送Handler延时消息，埋下炸弹，如果10s内（前台服务是20s）没人来拆炸弹，炸弹就会爆炸，即ActiveServices#serviceTimeout方法会被调用

#### 拆炸弹
启动一个Service，先要经过AMS管理，然后AMS会通知应用进程执行Service的生命周期， ActivityThread的handleCreateService方法会被调用
```
-> ActivityThread#handleCreateService

    private void handleCreateService(CreateServiceData data) {
        try {
           ...
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
             //1、service onCreate调用
            service.onCreate();
            mServices.put(data.token, service);
            try {
            	//2、拆炸弹在这里
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
    
    }
```

注释1，Service的onCreate方法被调用，
注释2，调用AMS的serviceDoneExecuting方法，最终会调用到ActiveServices. serviceDoneExecutingLocked

```
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
              boolean finishing) {

...
	//移除delay消息
	mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
...
            
 }

```

#### 引爆炸弹
假设Service的onCreate执行超过10s，那么炸弹就会引爆，也就是

ActiveServices#serviceTimeout方法会被调用
```
引爆炸弹
假设Service的onCreate执行超过10s，那么炸弹就会引爆，也就是

ActiveServices#serviceTimeout方法会被调用
```

所有ANR，最终都会调用AppErrors的appNotResponding方法

AppErrors #appNotResponding

```
    final void appNotResponding(ProcessRecord app, ActivityRecord activity,
            ActivityRecord parent, boolean aboveSystem, final String annotation) {
          ...
          
          //1、写入event log
          // Log the ANR to the event log.
          EventLog.writeEvent(EventLogTags.AM_ANR, app.userId, app.pid,
                    app.processName, app.info.flags, annotation);
           ...
          //2、收集需要的log，anr、cpu等，StringBuilder凭借
	        // Log the ANR to the main log.
	        StringBuilder info = new StringBuilder();
	        info.setLength(0);
	        info.append("ANR in ").append(app.processName);
	        if (activity != null && activity.shortComponentName != null) {
	            info.append(" (").append(activity.shortComponentName).append(")");
	        }
	        info.append("\n");
	        info.append("PID: ").append(app.pid).append("\n");
	        if (annotation != null) {
	            info.append("Reason: ").append(annotation).append("\n");
	        }
	        if (parent != null && parent != activity) {
	            info.append("Parent: ").append(parent.shortComponentName).append("\n");
	        }
	
	        ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);

	       ...
        // 3、dump堆栈信息，包括java堆栈和native堆栈，保存到文件中
        // For background ANRs, don't pass the ProcessCpuTracker to
        // avoid spending 1/2 second collecting stats to rank lastPids.
        File tracesFile = ActivityManagerService.dumpStackTraces(
                true, firstPids,
                (isSilentANR) ? null : processCpuTracker,
                (isSilentANR) ? null : lastPids,
                nativePids);

        String cpuInfo = null;
        ...

		    //4、输出ANR 日志
        Slog.e(TAG, info.toString());
        if (tracesFile == null) {
             // 5、没有抓到tracesFile，发一个SIGNAL_QUIT信号
            // There is no trace file, so dump (only) the alleged culprit's threads to the log
            Process.sendSignal(app.pid, Process.SIGNAL_QUIT);
        }

        StatsLog.write(StatsLog.ANR_OCCURRED, ...)
        // 6、输出到drapbox
        mService.addErrorToDropBox("anr", app, app.processName, activity, parent, annotation, cpuInfo, tracesFile, null);

        ...

        synchronized (mService) {
            mService.mBatteryStatsService.noteProcessAnr(app.processName, app.uid);
           //7、后台ANR，直接杀进程
            if (isSilentANR) {
                app.kill("bg anr", true);
                return;
            }

           //8、错误报告
            // Set the app's notResponding state, and look up the errorReportReceiver
            makeAppNotRespondingLocked(app,
                    activity != null ? activity.shortComponentName : null,
                    annotation != null ? "ANR " + annotation : "ANR",
                    info.toString());

            //9、弹出ANR dialog，会调用handleShowAnrUi方法
            // Bring up the infamous App Not Responding dialog
            Message msg = Message.obtain();
            msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
            msg.obj = new AppNotRespondingDialog.Data(app, activity, aboveSystem);

            mService.mUiHandler.sendMessage(msg);
        }
    }
```
主要流程如下：
1、写入event log
2、写入 main log
3、生成tracesFile
4、输出ANR logcat（控制台可以看到）
5、如果没有获取到tracesFile，会发一个SIGNAL_QUIT信号，这里看注释是会触发收集线程堆栈信息流程，写入traceFile
6、输出到drapbox
7、后台ANR，直接杀进程
8、错误报告
9、弹出ANR dialog，会调用 AppErrors#handleShowAnrUi方法。
ANR触发流程小结

```
ANR触发流程，可以比喻为埋炸弹和拆炸弹的过程，
以启动Service为例，Service的onCreate方法调用之前会使用Handler发送延时10s的消息，Service 的onCreate方法执行完，会把这个延时消息移除掉。
假如Service的onCreate方法耗时超过10s，延时消息就会被正常处理，也就是触发ANR，会收集cpu、堆栈等信息，弹ANR Dialog。
```

service、broadcast、provider 的ANR原理都是埋定时炸弹和拆炸弹原理，
但是input的超时检测机制稍微有点不同，需要等收到下一次input事件，才会去检测上一次input事件是否超时，input事件里埋的炸弹是普通炸弹，需要通过扫雷来排查。

#### ANR分析方法
触发ANR之后，一般我们会拉取anr日志： adb pull /data/traces.txt(文件名可能是anr_xxx.txt)

导致ANR的情况，实际项目中，可能有很多情况会导致ANR，例如死锁，内存不足、CPU被抢占、系统服务没有及时响应等等。

#### ANR 监控

抓取系统traces.txt 上传
1、当监控线程发现主线程卡死时，主动向系统发送SIGNAL_QUIT信号。
2、等待/data/anr/traces.txt文件生成。
3、文件生成以后进行上报。
这个方案在 《手Q Android线程死锁监控与自动化分析实践》 这篇文章中有详细介绍，
看起来好像可行，不过有以下两个问题：
1、traces.txt 里面包含所有线程的信息，上传之后需要人工过滤分析
2、很多高版本系统需要root权限才能读取 /data/anr这个目录

ANRWatchDog

其源码只有两个类，核心是ANRWatchDog这个类，继承自Thread，它的run 方法如下，看注释处
```
    public void run() {
        setName("|ANR-WatchDog|");

        long interval = _timeoutInterval;
       // 1、开启循环
        while (!isInterrupted()) {
            boolean needPost = _tick == 0;
            _tick += interval;
            if (needPost) {
               // 2、往UI线程post 一个Runnable，将_tick 赋值为0，将 _reported 赋值为false                      
              _uiHandler.post(_ticker);
            }

            try {
                // 3、线程睡眠5s
                Thread.sleep(interval);
            } catch (InterruptedException e) {
                _interruptionListener.onInterrupted(e);
                return ;
            }

            // If the main thread has not handled _ticker, it is blocked. ANR.
            // 4、线程睡眠5s之后，检查 _tick 和 _reported 标志，正常情况下_tick 已经被主线程改为0，_reported改为false，如果不是，说明 2 的主线程Runnable一直没有被执行，主线程卡住了
            if (_tick != 0 && !_reported) {
                ...
                if (_namePrefix != null) {
                    // 5、判断发生ANR了，那就获取堆栈信息，回调onAppNotResponding方法
                    error = ANRError.New(_tick, _namePrefix, _logThreadsWithoutStackTrace);
                } else {
                    error = ANRError.NewMainOnly(_tick);
                }
                _anrListener.onAppNotResponding(error);
                interval = _timeoutInterval;
                _reported = true;
            }

        }

    }

```

ANRWatchDog 的原理是比较简单的，概括为以下几个步骤

开启一个线程，死循环，循环中睡眠5s
往UI线程post 一个Runnable，将_tick 赋值为0，将 _reported 赋值为false
线程睡眠5s之后检查_tick和_reported字段是否被修改
如果_tick和_reported没有被修改，说明给主线程post的Runnable一直没有被执行，也就说明主线程卡顿至少5s**（只能说至少，这里存在5s内的误差）**。
将线程堆栈信息输出

#### ANRWatchDog 缺点
ANRWatchDog 会出现漏检测的情况，看图
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ae96474849043f4acb5719e2bf3e797~tplv-k3u1fbpfcp-zoom-crop-mark:1304:0:0:0.awebp)

如上图这种情况，红色表示卡顿，


假设主线程卡顿了2s之后，ANRWatchDog这时候刚开始一轮循环，将_tick 赋值为5，并往主线程post一个任务，把_tick修改为0


主线程过了3s之后不卡顿了，将_tick赋值为0


等到ANRWatchDog睡眠5s之后，发现_tick的值是0，判断为没有发生ANR。而实际上，主线程中间是卡顿了5s，ANRWatchDog误差是在5s之内的（5s是默认的，线程的睡眠时长）


针对这个问题，可以做一下优化。

ANRWatchDog 漏检测的问题，根本原因是因为线程睡眠5s，不知道前一秒主线程是否已经出现卡顿了，如果改成每间隔1秒检测一次，就可以把误差降低到1s内。
接下来通过改造ANRWatchDog ，来做一下优化，命名为ANRMonitor。
我们想让子线程间隔1s执行一次任务，可以通过 HandlerThread来实现
流程如下：
核心的Runnable代码

```
    @Volatile
    var mainHandlerRunEnd = true
    
    //子线程会间隔1s调用一次这个Runnable
    private val mThreadRunnable = Runnable {
        
        blockTime++
        //1、标志位 mainHandlerRunEnd 没有被主线程修改，说明有卡顿
        if (!mainHandlerRunEnd && !isDebugger()) {
            logw(TAG, "mThreadRunnable: main thread may be block at least $blockTime s")
        }

        //2、卡顿超过5s，触发ANR流程，打印堆栈
        if (blockTime >= 5) {
            if (!mainHandlerRunEnd && !isDebugger() && !mHadReport) {
                mHadReport = true
                //5s了，主线程还没更新这个标志，ANR
                loge(TAG, "ANR->main thread may be block at least $blockTime s ")
                loge(TAG, getMainThreadStack())
                //todo 回调出去，这里可以按需把其它线程的堆栈也输出
                //todo debug环境可以开一个新进程，弹出堆栈信息
            }
        }

        //3、如果上一秒没有卡顿，那么重置标志位，然后让主线程去修改这个标志位
        if (mainHandlerRunEnd) {
            mainHandlerRunEnd = false
            mMainHandler.post {
            	mainHandlerRunEnd = true
            }
            
        }

	    //子线程间隔1s调用一次mThreadRunnable
        sendDelayThreadMessage()

    }


```

#### 死锁监控

##### 获取blocked状态的线程
这个比较简单，一个for循环就搞定，不过我们要的线程id是native层的线程id，Thread 内部有一个native线程地址的字段叫 nativePeer，通过反射可以获取到
```
        Thread[] threads = getAllThreads();
        for (Thread thread : threads) {
            if (thread.getState() == Thread.State.BLOCKED) {
                long threadAddress = (long) ReflectUtil.getField(thread, "nativePeer");
                // 找不到地址，或者线程已经挂了，此时获取到的可能是0和-1
                if (threadAddress <= 0) {
                    continue;
    					  }
    						...后续
            }
        }

```
##### 获取该线程想要竞争的锁（native层函数）
##### 获取这个锁被哪个线程持有（native层函数）
##### 有了关系链，就可以找出造成死锁的线程


#### 形成闭环
前面分开讲了卡顿监控、ANR监控和死锁监控，我们可以把它连接起来，在发生ANR的时候，将整个监控流程形成一个闭环

发生ANR
获取主线程堆栈信息
检测死锁
获取死锁对应线程堆栈信息
上报到服务器
结合git，定位到最后修改代码的同学，给他提一个线上问题单

