[TOC]

# 协程基本概念
## 协程是什么

[Coroutine维基百科](https://en.wikipedia.org/wiki/Coroutine)
> Coroutines are computer program components that generalize subroutines for non-preemptive multitasking, by allowing execution to be suspended and resumed. 
 
 
 简而言之： 协程是由程序可以自行控制**挂起**和**恢复**的程序

* 可以实现多任务的协作执行  -  你唱罢来我登场，一个挂起，另一个恢复
* 可以解决异步操作的控制流程的灵活转移    - 简化回调
 
 
```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```
    
## 协程的作用

* 异步代码同步化
* 降低异步程序设计的复杂度

注意： 协程本身并不能够使我们的代码异步，主要是可以控制异步代码的流程（通过挂起和恢复）
## 线程与协程

* 线程 Thread  -- 内核线程，操作系统调度的线程

* 协程  Coroutine -- 运行在内核线程之上，编程语言实现的协程   


kotlin协程的实现就是 CPS（Continuation Passing Style ），协程是解决程序控制流程问题的。
如果你想知道协程运行在哪里，任何代码都是运行在操作系统内核线程上的。

# 协程在Android中的应用
[https://developer.android.com/kotlin/coroutines?hl=zh-cn](https://developer.android.com/kotlin/coroutines?hl=zh-cn)


协程是我们在 Android 上进行异步编程的推荐解决方案。值得关注的特点包括：

![5fcf3b3502d1853d427d9147b32aa9e9](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/E42A2D05-DDE2-4D1A-A744-D5B338EDC495.png)


## 网络请求

### RxJava方式
```kotlin
@GET("users/{name}")
fun getUser(@Path("name") login: String): Single<User>

private val subscriptions: CompositeSubscription = CompositeSubscription()

fun useRxJava(view: View) {
    val subscribe = rxApi.getUser("zhy060307")
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe({
            //onSuccess    runOnMainThread
            try{
                tvName.text = it.name
                ...
            } catch(e:Excepton){
                 Logger.error(e)
            }    
        }, {
            //onError
            Logger.error(it)
        })

    subscriptions.add(subscribe)
}

override fun onDestroy() {
    super.onDestroy()
    subscriptions.clear()
}
```

### 协程方式

```kotlin
@GET("users/{name}")
suspend fun getUser(@Path("name") login: String): User

fun useCoroutine(view: View) {
    lifecycleScope.launch {
        try {
            val user = suspendApi.getUser("zhy060307")
            tvName.text = user.name
        } catch (e: Exception) {
            Logger.error(e)
        }
    }
}
```
#### 取消任务

```kotlin
private var requsetJab: Job? = null
fun useCoroutine(view: View) {
    requsetJab?.cancel()
    requsetJab = lifecycleScope.launch {
        try {
            val user = suspendApi.getUser("zhy060307")
            tvName.text = user.name
        } catch (e: Exception) {
            Logger.error(e)
        }
    }
}
```

可以手动取消协程，可以防止多次请求


**协程可以解决以下问题**

1，异步流程难写
* 异步代码同步化，简化代码，业务流程处理更容易理解

2，异常捕获困难
* 异常捕获，主流程和异步流程的异常都可以捕获到

3，内存泄漏
* lifecycleScope 管理UI生命周期，onDestroy()时自动取消作用域内的Job，防止内存泄漏


你还在用RxJava吗？
![b3f297e2fbaf73e525f6043b5995cac5](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/F510249A-C567-4A1C-9676-D0389B212822.png)

## 回调转挂起函数，RxJava改为协程

### Retrofit Call

使用suspendCancellableCoroutine函数创建可取消的挂起函数
```kotlin
@GET("users/{name}")
fun getUser(@Path("name") login: String): Call<User>


suspend fun <T> Call<T>.await(): T = suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
        cancel()
    }

    enqueue(object : Callback<T> {
        override fun onFailure(call: Call<T>, t: Throwable) {
            continuation.resumeWithException(t)
        }

        override fun onResponse(call: Call<T>, response: Response<T>) {
            response.takeIf { it.isSuccessful }?.body()?.also { continuation.resume(it) }
                ?: continuation.resumeWithException(HttpException(response))
        }

    })
}

//调用
lifecycleScope.launch {
    val user = callbackApi.getUser("kotlin").await()
    tvName.text = user.name
}
```

### OnClickListener 

```kotlin
suspend fun Context.alert(title: String, message: String): Boolean =
    suspendCancellableCoroutine { continuation ->
        AlertDialog.Builder(this)
            .setTitle(title)
            .setMessage(message)
            .setNegativeButton("No") { dialog, which ->
                dialog.dismiss()
                continuation.resume(false)
            }.setPositiveButton("Yes") { dialog, which ->
                dialog.dismiss()
                continuation.resume(true)
            }.setOnCancelListener {
                continuation.resume(false)
            }.create()
            .also { dialog ->
                continuation.invokeOnCancellation {
                    dialog.dismiss()
                }
            }.show()
    }
 
 
 lifecycleScope.launch {
    val myChoice = alert("Warning", "Confirm do this?")
    Toast.makeText(context,"My choice [$myChoice]",Toast.LENGTH_SHORT).show()
}
```
### 代码举例
```java
 public Observable<List<MediaBrowserCompat.MediaItem>> query(@NonNull MediaSource mediaSource,
                                                                @NonNull String parentId,
                                                                @NonNull Bundle options) {
        return Observable.create(emitter -> {
            fetchMediaBrowser(mediaSource).subscribe(new SingleObserver<MediaSource>() {
                @Override
                public void onSubscribe(Disposable d) {

                }

                @Override
                public void onSuccess(MediaSource mediaSource) {
                    mediaSource.subscribe(parentId, options, new MediaBrowserCompat.SubscriptionCallback() {
                        @Override
                        public void onChildrenLoaded(@NonNull String parentId,
                                                     @NonNull List<MediaBrowserCompat.MediaItem> children) {
                            super.onChildrenLoaded(parentId, children, new Bundle());
                        }

                        @Override
                        public void onError(@NonNull String parentId) {
                            super.onError(parentId, new Bundle());
                        }

                        @Override
                        public void onChildrenLoaded(@NonNull String parentId,
                                                     @NonNull List<MediaBrowserCompat.MediaItem> children,
                                                     @NonNull Bundle bundle) {
                            if (emitter.isDisposed()) {
                                Logger.t(TAG).d("query emitter is disposed");
                                return;
                            }

                            //Add for spotify, spotify should set null result, but it returns an empty list
                            if (children.isEmpty()) {
                                PlaybackStateCompat playbackState = MediaPlayController.getInstance().getPlaybackState(mediaSource);
                                Logger.t(TAG).d("playbackState:" + playbackState);
                                if (playbackState != null && playbackState.getErrorCode() == PlaybackStateCompat.ERROR_CODE_AUTHENTICATION_EXPIRED && playbackState.getExtras() != null) {
                                    emitter.onError(new LoginException(playbackState.getErrorMessage().toString(), playbackState.getExtras()));
                                    return;
                                }
                            }

                            boolean isComplete = bundle != null ? bundle.getBoolean(Protocols.KEY_PARSING_IS_COMPLETE, true) : true;
                            Logger.t(TAG).d("onChildrenLoaded: id= " + mediaSource.getAppId()
                                    + ", parentId=" + parentId + ",item size=" + children.size());
                            super.onChildrenLoaded(parentId, children, bundle);
                            if (bundle.getBoolean(Protocols.KEY_REQUEST_MEDIA_INFO, false) && children.size() == 0) {
                                Logger.t(TAG).d("query get media info failed");
                            } else {
                                emitter.onNext(children);
                            }

                            if (isComplete) {
                                emitter.onComplete();
                            } else {
                                MediaBrowserCompat.SubscriptionCallback callback = this;
                                options.putBoolean(Protocols.KEY_REQUEST_MEDIA_INFO, true);
                                Schedulers.io().scheduleDirect(new Runnable() {
                                    @Override
                                    public void run() {
                                        mediaSource.subscribe(parentId, options, callback);
                                    }
                                }, 5000, TimeUnit.MILLISECONDS);
                            }
                        }

                        @Override
                        public void onError(@NonNull String parentId, @NonNull Bundle options) {
                            super.onError(parentId, options);
                            Logger.t(TAG).d("onError: parentId=" + parentId);
                            PlaybackStateCompat playbackState = MediaPlayController.getInstance().getPlaybackState(mediaSource);
                            if (playbackState != null && playbackState.getErrorCode() == PlaybackStateCompat.ERROR_CODE_AUTHENTICATION_EXPIRED) {
                                if (playbackState.getExtras() != null) {
                                    emitter.onError(new LoginException(playbackState.getErrorMessage().toString(), playbackState.getExtras()));
                                    return;
                                }
                            }
                            emitter.onError(new Exception("onChildrenLoaded onError: parentId=" + parentId));
                        }
                    });
                }

                @Override
                public void onError(Throwable e) {
                    emitter.onError(e);
                }
            });
        });
    }
```



转换为挂起函数，整个流程清晰明了

```kotlin
 
 suspend fun query(mediaSource: MediaSource, parentId: String, options: Bundle)
        : MutableList<MediaBrowserCompat.MediaItem> {
    val mediaBrowser = fetchMediaBrowser(mediaSource)
    return subscribeMediaSource(mediaBrowser, parentId, options)
}

    
suspend fun fetchMediaBrowser(mediaSource: MediaSource)
        : MediaSource =
        suspendCancellableCoroutine<MediaSource> {
            //....
            it.resumeWith(Result.success(MediaSource()))

        }

suspend fun subscribeMediaSource(mediaSource: MediaSource, parentId: String, options: Bundle) =
        suspendCancellableCoroutine<MutableList<MediaBrowserCompat.MediaItem>> { continuation ->
            mediaSource.subscribe(parentId, options, object : SubscriptionCallback() {
                override fun onChildrenLoaded(parentId: String,
                                              children: MutableList<MediaBrowserCompat.MediaItem>,
                                              options: Bundle) {
                    super.onChildrenLoaded(parentId, children, options)
                    continuation.resumeWith(Result.success(children))
                    // if any exception
                    continuation.resumeWithException(erro)

                }
            })
        }
```

### 协程和RxJava的转换

RxJava和Coroutine的无缝切换
[kotlin-coroutines-rx](https://github.com/Kotlin/kotlinx.coroutines/tree/master/reactive) 

Coroutine 转换为 Rx2
![0885d1ea2d18fae5fb382f50a3ad6070](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/EA5CCEBE-C862-44CE-952F-90231EC74DC6.png)

Rx2 转换为 Coroutine

![7838e9465940816deb130cd762d6e011](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/D74CB3D7-AFBD-4A34-AFBD-8005DC28561C.png)


```kotlin
@GET("users/{name}")
fun getUser(@Path("name") login: String): Single<User>
//rx2->coroutine
val user = rxApi.getUser("google").await()


//coroutine->rx2
rxSingle(coroutineContext){suspendApi.getUser("google")}
```

## mvvm 中使用

![7f51fd305c90bf120b54c3b7afff5ec5](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/04B8608F-A2A3-426D-96F3-F78B6DC2C841.png)

Database.kt
```kotlin
// add the suspend modifier to the existing insertTitle
@Insert(onConflict = OnConflictStrategy.REPLACE)
suspend fun insertTitle(title: Title)
```

Network.kt

```kotlin 
// add suspend modifier to the existing fetchNextTitle
// change return type from Call<String> to String
// Suspend function support requires Retrofit 2.6.0 or higher
interface MainNetwork {
   @GET("next_title.json")
   suspend fun fetchNextTitle(): String
}
```

Repository.kt
```kotlin
suspend fun refreshTitle() {
   try {
       // Make network request using a blocking call
       val result = network.fetchNextTitle()
       titleDao.insertTitle(Title(result))
   } catch (cause: Throwable) {
       // If anything throws an exception, inform the caller
       throw TitleRefreshError("Unable to refresh title", cause)
   }
}
```

ViewModel.kt
```kotlin
fun refreshTitle() {
   viewModelScope.launch {
       try {
           _spinner.value = true
           // this is the only part that changes between sources
           repository.refreshTitle() 
       } catch (error: TitleRefreshError) {
           _snackBar.value = error.message
       } finally {
           _spinner.value = false
       }
   }
}
```
Using coroutines in higher order functions


```kotlin
//ViewModel.kt
fun refreshTitle() {
   launchDataLoad {
       repository.refreshTitle()
   }
}
private fun launchDataLoad(block: suspend () -> Unit): Job {
   return viewModelScope.launch {
       try {
           _spinner.value = true
           block()
       } catch (error: TitleRefreshError) {
           _snackBar.value = error.message
       } finally {
           _spinner.value = false
       }
   }
}
```

## Flow
Flow 库是在 Kotlin Coroutines 1.3.2 发布之后新增的库，也叫做异步流，类似 RxJava 的 Observable 、 Flowable 等等，所以很多人都用 Flow 与 RxJava 做对比，事实上Flow也是借鉴了RxJava的reactive响应式编程的思想

[https://developer.android.com/kotlin/flow](https://developer.android.com/kotlin/flow)

![8dec55dc9b8a2eb168ae846d996b1035](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/FFC78526-3D40-4614-8980-81E359557310.png)

### 线程切换
通过flowOn来切换上游的协程

### 异常处理

``` kotlin
suspend fun exception() {
    flow<Int> {
        emit(1)
        throw ArithmeticException("Div 0")
    }.catch { t: Throwable ->
        Logger.debug("caught error: $t")
    }.onCompletion { t: Throwable? ->
        Logger.debug("finally.")
    }.flowOn(Dispatchers.IO)
        .collect { Logger.debug(it) }

}
```

### 背压 Back Pressure


* buffer ：指定固定容量的缓存 ，可以缓解
* conflate: 保留最新的值
* collectLatest : 新值发射时取消之前的消费函数


```kotlin
suspend fun backPressure(){
    flow {
        emit(1)
        delay(50)
        emit(2)
    }.collectLatest { value ->
        println("Collecting $value")
        delay(100) // Emulate work
        println("$value collected")
    }
}


Collecting 1
Collecting 2
2 collected
```
### 操作符

map，flatMap，zip，combine ....


### 应用案例
```kotlin

// Interface that provides a way to make network requests with suspend functions
interface NewsApi {
    suspend fun fetchLatestNews(): List<ArticleHeadline>
}

class NewsRemoteDataSource(
    private val newsApi: NewsApi,
    private val refreshIntervalMs: Long = 5000
) {
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        while(true) {
            val latestNews = newsApi.fetchLatestNews()
            emit(latestNews) // Emits the result of the request to the flow
            delay(refreshIntervalMs) // Suspends the coroutine for some time
        }
    }
}

class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val userData: UserData,
    private val defaultDispatcher: CoroutineDispatcher
) {
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            .map { news -> // Executes on the default dispatcher
                news.filter { userData.isFavoriteTopic(it) }
            }
            .onEach { news -> // Executes on the default dispatcher
                saveInCache(news)
            }
            // flowOn affects the upstream flow ↑
            .flowOn(defaultDispatcher)
            // the downstream flow ↓ is not affected
            .catch { exception -> // Executes in the consumer's context
                emit(lastCachedNews())
            }
}

class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // Intermediate catch operator. If an exception is thrown,
                // catch and update the UI
                .catch { exception -> notifyError(exception) }
                .collect { favoriteNews ->
                    // Update View with the latest favorite news
                }
        }
    }
}
```





## Test

未完待续。。。。。



## Jetpack集成
Jetpack 与协程 官方Demo

[Sunflower](https://github.com/android/sunflower)


# 协程调用的逻辑

对于挂起函数，编译器会自动添加Continuation类型的参数，
协程的实现就是 CPS（Continuation Passing Style ）

```kotlin
// suspend () -> Unit
suspend fun foo() {

}
//等价于
fun foo(continuation: Continuation<Unit>):Any?{

}

```

![a2a349378bc35c20b8c7ed34ad8311ad](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources//D8856F83-82C4-4DBC-A98E-9C7DF50749E1.png)



挂起点的状态保持在Continuation中
```kotlin
public interface Continuation<in T> {

    public val context: CoroutineContext
    
    public fun resumeWith(result: Result<T>)
}

public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))
    
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))    
```

Continuation 类似Callback，onSuccess,onFailure

```kotlin
suspend fun main() {
    log.debug(1)
    log.debug(suspendReturn())
    log.debug(2)
    delay(1000)
    log.debug(3)
    log.debug(immediatelyReturn())
    log.debug(4)
}

suspend fun suspendReturn() = suspendCoroutineUninterceptedOrReturn<String> { continuation ->
    thread(name = "suspendThread") {
        Thread.sleep(1000)
        continuation.resume("suspend return")
    }
    COROUTINE_SUSPENDED
}

suspend fun immediatelyReturn() = suspendCoroutineUninterceptedOrReturn<String> {
    "immediately return"
}

private val executors = Executors.newScheduledThreadPool(1)
{ r: Runnable ->
    Thread(r, "My-Delay-Scheduler").apply {
        isDaemon = true
    }
}

@JvmOverloads
suspend fun delay(time: Long, unit: TimeUnit = TimeUnit.MILLISECONDS) =
    suspendCoroutine<Unit> { continuation ->
        executors.schedule({
            continuation.resume(Unit)
        }, time, unit)
    }
```

用Java实现上面的调用逻辑如下：


* RunSuspend

同标准库中RunSuspend.kt中的RunSuspend类，其作用是等待整个流程执行完毕后，再退出程序
```java
public class RunSuspend implements Continuation<Unit> {

    private Object result;
    @NotNull
    @Override
    public CoroutineContext getContext() {
        return EmptyCoroutineContext.INSTANCE;
    }

    @Override
    public void resumeWith(@NotNull Object result) {
        synchronized (this) {
            this.result = result;
            notifyAll();
        }
    }

    public void await() throws Throwable {
        synchronized (this) {
            while (true) {
                Object result = this.result;
                if (result == null) {
                    wait();
                } else if (result instanceof Throwable) {
                    throw (Throwable) result;
                } else {
                    return;
                }
            }
        }
    }
}
```


* CoutinuationImpl

用Java来模拟协程的执行流程，其实kotlin编译后的字节码执行流程与其类似
```java
public class CoutinuationImpl implements Continuation<Object> {

    private final Continuation<Unit> completion;

    public CoutinuationImpl(Continuation<Unit> completion) {
        this.completion = completion;
    }

    private int label = 0;

    @Override
    public CoroutineContext getContext() {
        return EmptyCoroutineContext.INSTANCE;
    }

    @Override
    public void resumeWith(@NotNull Object o) {
        try {
            Object result = o;
            switch (label) {
                case 0:
                    Logger.INSTANCE.debug(1);
                    result = RunCoroutineKt.suspendReturn(this);
                    label++;
                    if (isSuspended(result)) return;
                case 1:
                    Logger.INSTANCE.debug(result);
                    Logger.INSTANCE.debug(2);
                    DelayKt.delay(1000, this);
                    label++;
                    if (isSuspended(result)) return;
                case 2:
                    Logger.INSTANCE.debug(3);
                    result = RunCoroutineKt.immediatelyReturn(this);
                    label++;
                    if (isSuspended(result)) return;
                case 3:
                    Logger.INSTANCE.debug(result);
                    Logger.INSTANCE.debug(4);
            }
            completion.resumeWith(Unit.INSTANCE);
        } catch (Exception e) {
            completion.resumeWith(e);
        }
    }

    private boolean isSuspended(Object result) {
        return result == IntrinsicsKt.getCOROUTINE_SUSPENDED();
    }

    //启动
    public static void main(String[] args) throws Throwable {
        RunSuspend runSuspend = new RunSuspend();
        CoutinuationImpl table = new CoutinuationImpl(runSuspend);
        table.resumeWith(Unit.INSTANCE);
        runSuspend.await();
    }
}
```


![67b630ba6b773d119fbe2be7c517c404](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/3C1844A1-D173-40AB-90F8-59DF355AD9AB.png)


[Demo Code Click here!!!](https://github.com/vivid-cn/vivid-blog/blob/main/Kotlin%20%E5%8D%8F%E7%A8%8B.resources/CoroutineLearning.zip)

# 参考文献
[Coroutines guide](https://kotlinlang.org/docs/coroutines-guide.html)

[《深入理解Kotlin协程》](https://www.bennyhuo.com/project/kotlin-coroutines.html)
[“吹Kotlin协程的，可能吹错了！”带你真正理解一波](https://zhuanlan.zhihu.com/p/143098698)

[Kotlin Coroutine 原理解析](https://www.jianshu.com/p/06703abc56b1)

[https://www.imooc.com/article/300260](https://www.imooc.com/article/300260)

[https://developer.android.com/kotlin/coroutines/additional-resources?hl=zh-cn](https://developer.android.com/kotlin/coroutines/additional-resources?hl=zh-cn)
