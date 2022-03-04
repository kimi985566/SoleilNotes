# 监测应用的 FPS

## 什么是 FPS ？

即使你不知道 FPS，但你一定听说过这么一句话，在 Android 中，每一帧的绘制时间不要超过 16.67ms。那么，这个 16.67ms 是怎么来的呢？就是由 FPS 决定的。

FPS，Frame Per Second，每秒显示的帧数，也叫 帧率。

Android 设备的 FPS 一般是 60，也即每秒要刷新 60 帧，所以留给每一帧的绘制时间最多只有 1000/60 =  16.67ms 。一旦某一帧的绘制时间超过了限制，就会发生 掉帧，用户在连续两帧会看到同样的画面。

监测 FPS 在一定程度上可以反应应用的卡顿情况，原理也很简单，但前提是你对屏幕刷新机制和绘制流程很熟悉。所以我不会直接进入主题，让我们先从 View.invalidate() 说起。

## 如何监测应用的 FPS？

监测当前应用的 FPS 很简单。每次 vsync 信号回调中，都会执行四种类型的 mCallbackQueues 队列中的回调任务。而 Choreographer 又对外提供了提交回调任务的方法，这个方法就是 Choreographer.getInstance().postFrameCallback() 。简单跟进去看一下。

```java
> Choreographer.java

public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    ......
    // 这里的类型是 CALLBACK_ANIMATION
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```

和 View.invalite() 流程中调用的 Choreographer.postCallback() 基本一致，仅仅只是 callback 类型不一致，这里是 CALLBACK_ANIMATION 。

```Kotlin
object FpsMonitor {

    private const val FPS_INTERVAL_TIME = 1000L
    private var count = 0
    private var isFpsOpen = false
    private val fpsRunnable by lazy { FpsRunnable() }
    private val mainHandler by lazy { Handler(Looper.getMainLooper()) }
    private val listeners = arrayListOf<(Int) -> Unit>()

    fun startMonitor(listener: (Int) -> Unit) {
        // 防止重复开启
        if (!isFpsOpen) {
            isFpsOpen = true
            listeners.add(listener)
            mainHandler.postDelayed(fpsRunnable, FPS_INTERVAL_TIME)
            Choreographer.getInstance().postFrameCallback(fpsRunnable)
        }
    }

    fun stopMonitor() {
        count = 0
        mainHandler.removeCallbacks(fpsRunnable)
        Choreographer.getInstance().removeFrameCallback(fpsRunnable)
        isFpsOpen = false
    }

    class FpsRunnable : Choreographer.FrameCallback, Runnable {
        override fun doFrame(frameTimeNanos: Long) {
            count++
            Choreographer.getInstance().postFrameCallback(this)
        }

        override fun run() {
            listeners.forEach { it.invoke(count) }
            count = 0
            mainHandler.postDelayed(this, FPS_INTERVAL_TIME)
        }
    }
}
```

大致逻辑是这样的 ：

- 声明变量 count 用于统计回调次数
- 通过 Choreographer.getInstance().postFrameCallback(fpsRunnable) 注册监听下一次 vsync信号，提交任务，任务回调只做两件事，一是 count++，二是继续注册监听下一次 vsync 信号 。
- 通过 Handler 做个定时任务，每隔一秒统计 count 值并清空。

目前也有很多开源项目实现了 FPS 监测或者流畅度检测的功能。

- 滴滴开源的 DoraemonKit
- 腾讯开源的 Matrix: 虽然也是在 Choreographer 上动手脚，但做的更加彻底，它可以监听到 CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_TRAVERSAL  三种事件各自精确的耗时，重点关注 LooperMonitor 和 UIThreadMonitor 这两个类。
