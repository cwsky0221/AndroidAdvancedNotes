### OKHttp 原理

#### 概要：
![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghbbmhed3mj314807yq4v.jpg?imageslim )
![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghbc45yq1aj317k0mkacr.jpg?imageslim)

#### 一、基本使用：
```
        String url = xxx;
        //构建者模式创建Request请求，设置url
        Request request = new Request.Builder().url(url).build();
        //创建Call会话
        Call call = mClient.newCall(request);
        //异步请求入队
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                
            }
            @Override
            public void onResponse(Call call, Response response) throws IOException {               
                //body只能取一次，Response就会关闭，所以要用临时变量接收
                String result = response.body().string();
                //回调在子线程，要操作UI的话需切回主线程
                runOnUiThread(() -> {
                    mBinding.tv.setText(result);
                });
            }
        });
```

#### 二、核心流程：

Call的实例是RealCall,开始方法：

``` 
//RealCall.java
void enqueue(Callback responseCallback) {
    //回调eventListener
    transmitter.callStart();
    //用AsyncCall封装Callback，由调度中心dispatcher安排就绪
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}

//Dispatcher.java
enqueue(AsyncCall call) {
    synchronized (this) {
        //飞机进入就绪跑道
        readyAsyncCalls.add(call);
        if (!call.get().forWebSocket) {
            //查找飞往上海的AsyncCall
            AsyncCall existingCall = findExistingCallWithHost(call.host());
            //复用上海的计数器callsPerHost，用于统计同一城市的航班
            if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
        }
    }
    //飞机进入起飞跑道
    promoteAndExecute();
}

```
跟进promoteAndExecute

```
//Dispatcher.java
boolean promoteAndExecute() {
    //收集可以执行的AsyncCall
    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
        for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
            AsyncCall asyncCall = i.next();
			//64个起飞跑道被占满，跳出
            if (runningAsyncCalls.size() >= maxRequests) break;
            //飞往上海的航班达到5个，留在就绪跑道就行，跳过
            if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue;
			//离开就绪跑道
            i.remove();
            //上海航班计数器+1
            asyncCall.callsPerHost().incrementAndGet();
            //把AsyncCall存起来
            executableCalls.add(asyncCall);
            //进入起飞跑道
            runningAsyncCalls.add(asyncCall);
        }
        isRunning = runningCallsCount() > 0;
    }
	//把可以执行的AsyncCall，统统起飞
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
        AsyncCall asyncCall = executableCalls.get(i);
        //真正请求了，executorService()返回了一个线程池
        asyncCall.executeOn(executorService());
    }
    return isRunning;
}

```
继续跟进executeOn方法

AsyncCall就是一个Runnable，run方法里调了execute方法, 请求就在这个runnable中进行
```
//AsyncCall.java
void executeOn(ExecutorService executorService) {
    boolean success = false;
    try {
        //线程池运行Runnable，执行run，run会调用AsyncCall.execute
        executorService.execute(this);
        success = true;
    } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        transmitter.noMoreExchanges(ioException);
        //失败回调
        responseCallback.onFailure(RealCall.this, ioException);
    } finally {
        if (!success) {
            //结束航班
            client.dispatcher().finished(this);
        }
    }
}

//AsyncCall.java
void execute() {
    try {
        //得到Response，抵达目的地, 此方法重点，处理拦截器，最终获取response
        Response response = getResponseWithInterceptorChain();
        //成功（一般response.isSuccessful()才是真正意义上的成功）
        responseCallback.onResponse(RealCall.this, response);
    } catch (IOException e) {
        //失败
        responseCallback.onFailure(RealCall.this, e);
    } catch (Throwable t) {
        cancel();
        IOException canceledException = new IOException("canceled due to " + t);
        canceledException.addSuppressed(t);
        //失败
        responseCallback.onFailure(RealCall.this, canceledException);
        throw t;
    } finally {
        //结束航班，callsPerHost减1，runningAsyncCalls移除AsyncCall
        client.dispatcher().finished(this);
    }
}

```
回调都在子线程里完成，所以Activity里要切回主线程才能操作UI。至此，核心流程就结束了。
![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghcdsro2vdj31940b4jtr.jpg?imageslim)

#### 三、拦截器：
前边得到Response的地方，调了getResponseWithInterceptorChain，进去看看

```
//RealCall.java
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList<>();
    //添加自定义拦截器
    interceptors.addAll(client.interceptors());
    //添加默认拦截器
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        //添加自定义网络拦截器（在ConnectInterceptor后面，此时网络连接已准备好）
        interceptors.addAll(client.networkInterceptors());
    }
    //添加默认拦截器，共4+1=5个
    interceptors.add(new CallServerInterceptor(forWebSocket));
    //创建拦截器链
    Interceptor.Chain chain =
        new RealInterceptorChain(interceptors, transmitter, null, 0,
                                 originalRequest, this, client.connectTimeoutMillis(),
                                 client.readTimeoutMillis(), client.writeTimeoutMillis());
    //放行
    Response response = chain.proceed(originalRequest);
    return response;
}

```
拦截器链基于责任链模式，即不同的拦截器有不同的职责，链上的拦截器会按顺序挨个处理，在Request发出之前，Response返回之后，插入一些定制逻辑，这样可以方便的扩展需求。当然责任链模式也有不足，就是只要一个环节阻塞住了，就会拖慢整体运行（效率）；同时链条越长，产生的中间对象就越多（内存）

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghbgbi291zj30is0ieq45.jpg)

先看责任链的核心，proceed方法：
```
//RealInterceptorChain.java
Response proceed(Request request, Transmitter transmitter,Exchange exchange)
    throws IOException {
    //传入index + 1，可以访问下一个拦截器
    RealInterceptorChain next = 
        new RealInterceptorChain(interceptors, transmitter, exchange,
                                 index + 1, request, call, connectTimeout, 
                                 readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    //执行第一个拦截器，同时传入next
    Response response = interceptor.intercept(next);
    //等所有拦截器处理完，就能返回Response了
    return response;
}

```
拦截器的调用顺序如上图：

自定义拦截器-》RetryAndFollowUpInterceptor-》BridgeInterceptor-》CacheInterceptor-》ConnectInterceptor-》CallServerInterceptor

获取response后返回顺序相反


下面简要分析下各个拦截器的功能。

一、RetryAndFollowUpInterceptor：

负责重试和重定向。

```
//最大重试次数
static final int MAX_FOLLOW_UPS = 20;

Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Transmitter transmitter = realChain.transmitter();
    int followUpCount = 0;
    while (true) {
        //机长为Request准备一个连接
        //主机、端口、协议都相同时，连接可复用
        transmitter.prepareToConnect(request);
        //放行，让后面的拦截器执行
        Response response = realChain.proceed(request, transmitter, null);
        //后面的拦截器执行完了，拿到Response，解析看下是否需要重试或重定向，需要则返回新的Request
        Request followUp = followUpRequest(response, route);
        if (followUp == null) {
            //新的Request为空，直接返回response
            return response;
        }
        RequestBody followUpBody = followUp.body();
        if (followUpBody != null && followUpBody.isOneShot()) {
            //如果RequestBody有值且只许被调用一次，直接返回response
            return response;
        }
        if (++followUpCount > MAX_FOLLOW_UPS) {
            //重试次数上限，结束
            throw new ProtocolException("Too many follow-up requests: " + followUpCount);
        }
        //将新的请求赋值给request，继续循环
        request = followUp;
    }
}

```
二、BridgeInterceptor：

桥接，负责把应用请求转换成网络请求，把网络响应转换成应用响应，说白了就是处理一些网络需要的header，简化应用层逻辑。
```
Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    RequestBody body = userRequest.body();
    if (body != null) {
        requestBuilder.header("Content-Type", contentType.toString());
        //处理Content-Length、Transfer-Encoding
        //...
    }
    //处理Host、Connection、Accept-Encoding、Cookie、User-Agent、
    //...
    //放行，把处理好的新请求往下传递，得到Response
    Response networkResponse = chain.proceed(requestBuilder.build());
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
	//处理新Response的Content-Encoding、Content-Length、Content-Type、gzip
    //返回新Response
    return responseBuilder.build();
}

```
三、CacheInterceptor：

负责管理缓存，使用okio读写缓存。

```
InternalCache cache;

Response intercept(Chain chain) throws IOException {
    //获取候选缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    //创建缓存策略
    CacheStrategy strategy = 
        new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    //网络请求
    Request networkRequest = strategy.networkRequest;
    //缓存Response
    Response cacheResponse = strategy.cacheResponse;
    //如果网络请求和缓存Response都为空
    if (networkRequest == null && cacheResponse == null) {
        //返回一个504的Response
        return new Response.Builder().code(504).xxx.build();
    }
    //如果不使用网络，直接返回缓存
    if (networkRequest == null) {
        return cacheResponse.newBuilder()
            .cacheResponse(stripBody(cacheResponse)).build();
    }
    //放行，往后走
    Response networkResponse = chain.proceed(networkRequest);
    if (cacheResponse != null) {
        //获取到缓存响应码304，即缓存可用
        if (networkResponse.code() == HTTP_NOT_MODIFIED) {
            Response response = cacheResponse.newBuilder().xxx.build();
            //更新缓存，返回
            cache.update(cacheResponse, response);
            return response;
        }
    }
    //获取网络Response
    Response response = networkResponse.newBuilder().xxx.build();
    //写入缓存，返回
    cache.put(response);
    return response;
}

```
四、ConnectInterceptor：

负责创建连接Connection。
```
Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //机长创建一个交换器Exchange
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
    //放行，给下一个拦截器
    return realChain.proceed(request, transmitter, exchange);
}

```
五、CallServerInterceptor：

负责写请求和读响应。最后一个拦截器

```
Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Exchange exchange = realChain.exchange();
    Request request = realChain.request();
    //写请求头
    exchange.writeRequestHeaders(request);
    Response.Builder responseBuilder = null;
    //处理请求体body...
    //读取响应头
    responseBuilder = exchange.readResponseHeaders(false);
    //构建响应
    Response response = responseBuilder
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
    //读取响应体
    response = response.newBuilder()
        .body(exchange.openResponseBody(response))
        .build();
    return response;
}

```

#### 四、缓存策略：

#### 五、线程池：
