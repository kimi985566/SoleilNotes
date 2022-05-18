# LiveData

## LiveData 怎么感知生命周期感知？需要取消注册吗？

1. 调用 observe 方法时，会调用 owner.getLifecycle().addObserver 已达到感知生命周期的目的。

```Java
private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
        new SafeIterableMap<>();
/**
 * 1、在给定 owner 的生命周期内，将给定的观察者添加到观察者列表中。
 * 2、事件在主线程上调度。
 * 3、如果LiveData已经有数据集，它将被交付给观察者。
 * <p>
 * 4、仅当所有者处于{@link Lifecycle.State#STARTED}或{@link Lifecycle.State#RESUME}状态
 *（活动）时，观察者才会接收事件。
 * <p>
 * 5、如果所有者移动到{@link Lifecycle.State#DETROYED}状态，观察者将自动被删除。
 * <p>
 * 6、当数据在{@code owner}未激活时发生更改时，它将不会收到任何更新。
 * 7、如果它再次激活，它将自动接收最后可用的数据。
 * <p>
 * 8、只要给定的LifecycleOwner未被销毁，LiveData就会保留对观察者和所有者的强引用。
 * 9、销毁后，LiveData将删除对所有者的引用。
 * <p>
 * 10、如果给定的所有者已经处于{@link Lifecycle.State#DESTROYED}状态，LiveData将忽略该调用。
 * <p>
 * 11、如果给定的所有者、观察者元组已经在列表中，则忽略该调用。
 * 12、如果观察者已经列表中，LiveData将抛出@link IllegalArgumentException}。
 *
 * @param owner    控制观察者的生命周期所有者
 * @param observer 将接收事件的观察者
 */
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        // owner 的状态是 DESTROYED 则不往下走。对应方法注释的第 10 点。
        return;
    }
    // 创建 owner 和 observer 的包装对象 LifecycleBoundObserver
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    // observer 对象作为 Key，包装对象 wrapper 作为 value，存进 Map 结构对象 mObservers
    // putIfAbsent 方法：若 Map 里有这个key，则返回对应的 Value 值。
    // 仅当 key 不存在时才会将 key value 存进 Map
    // put 进去的是 LifecycleBoundObserver 类型对象，返回的是 ObserverWrapper 对象，
    // 不难猜出 LifecycleBoundObserver 是 ObserverWrapper 的子类或者实现类
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    // 若 existing 不为null， 并且 existing 中的 owner 对象和 observe 方法传入的对象不是同一个对象，
    // 则抛出异常，不可以添加同一个 Observer 到两个不同的生命周期对象。对应方法注释的第 12 点。
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        // 若 existing 不为 null，则忽略该调用。对应方法注释的第 12 点。
        return;
    }
    // 将 owner 和 observer 的包装对象添加到 owner.getLifecycle()。对应方法注释的第 1 点。
    owner.getLifecycle().addObserver(wrapper);
}
```

2. onStateChanged 方法中注释如果是已销毁状态，则调用 LiveData#removeObserver 方法进行移除 mObserver (mObserver 是 LifecycleBoundObserver 的父类 ObserverWrapper 的属性，通过 ObserverWrapper 的构造函数来赋值)，对应 LiveData#observe 方法注释的第 12 点。

```Java
# LiveData 的内部类
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    // 构造函数 owner 赋值给属性 mOwner， observer 则调用了 super
    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    /**
     * 当 source (也就是 mOwner) 的生命周期改变时会回调此方法
     */
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        // 获取当前 mOwner 的生命周期状态
        Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
        if (currentState == DESTROYED) {
            // 如果是已销毁状态，则调用 LiveData#removeObserver 方法进行移除 mObserver (mObserver 是 LifecycleBoundObserver 的父类 ObserverWrapper 的属性，通过 ObserverWrapper 的构造函数来赋值)，对应 LiveData#observe 方法注释的第 12 点。
            removeObserver(mObserver);
            return;
        }
        Lifecycle.State prevState = null;
        while (prevState != currentState) {
            prevState = currentState;
            activeStateChanged(shouldBeActive());
            currentState = mOwner.getLifecycle().getCurrentState();
        }
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        // 从 mOwner 中移除此观察者
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

## setValue 和 postValue 有什么区别

1. setValue 只能在主线程使用，而 postValue 不限制线程

### setValue

> 关键：setValue 必须在主线程中调用

```Java
/**
 * Sets the value. If there are active observers, the value will be dispatched to them.
 * 设置值。如果存在活动的观察者，则会将值分派给他们。
 * <p>
 * This method must be called from the main thread. If you need set a value from a background
 * thread, you can use {@link #postValue(Object)}
 * 必须从主线程调用此方法。如果需要从后台线程设置值，可以使用{@link#postValue（Object）}
 * @param value The new value
 */
@MainThread
protected void setValue(T value) {
    // 检查当前线程是否是主线程，若非主线程则会抛异常
    assertMainThread("setValue");
    // 很关键的 mVersion ，在这里进行 + 1 操作
    mVersion++;
    // 将新值赋值给属性 mData
    mData = value;
    // 分发值 (后面再看它具体实现)
    dispatchingValue(null);
}
```

### postValue

LiveData#postValue 最终使用主线程将新 value 分发给观察者。意味着我们可以在任何线程调用 postValue，而不用担心线程问题。因为 LiveData 内部做好了线程转换。

```Java
/**
 * Posts a task to a main thread to set the given value. So if you have a following code
 * executed in the main thread:
 * 将任务发布到主线程以设置给定值。因此，如果在主线程中执行以下代码：
 * <pre class="prettyprint">
 * liveData.postValue("a");
 * liveData.setValue("b");
 * </pre>
 * The value "b" would be set at first and later the main thread would override it with
 * the value "a".
 * 首先设置值“b”，然后主线程将用值“a”覆盖它。(等我们看完了 postValue 具体怎么做的，这个官方小示例就能明白了)
 * <p>
 * If you called this method multiple times before a main thread executed a posted task, only
 * the last value would be dispatched.
 * 如果在主线程执行已发布任务之前多次调用此方法，则只会调度最后一个值。
 * @param value The new value
 */
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        // 新值赋值给属性 mPendingData
        mPendingData = value;
    }
    // 当 mPendingData == NOT_SET 时，才会往下走
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```

```Java
private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            // 将刚赋值新值的 mPendingData 赋值给对象 newValue
            newValue = mPendingData;
            // 将 mPendingData 重置为 NOT_SET
            // 回头看看 postValue 会明白为啥要这么做
            mPendingData = NOT_SET;
        }
        // 嘶 ~ 在这儿调用了 setValue，
        setValue((T) newValue);
    }
};
```

这时候看出来 postValue 最终还是调用了 setValue 再看下 postToMainThread

```Java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class ArchTaskExecutor extends TaskExecutor {

    @NonNull
    private TaskExecutor mDelegate;

    private ArchTaskExecutor() {
        mDefaultTaskExecutor = new DefaultTaskExecutor();
        mDelegate = mDefaultTaskExecutor;
    }
    
    @Override
    public void postToMainThread(Runnable runnable) {
        mDelegate.postToMainThread(runnable);
    }
}
```

由 DefaultTaskExecutor 对象调用的 postToMainThread

```Java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class DefaultTaskExecutor extends TaskExecutor {

    @Nullable
    private volatile Handler mMainHandler;

    @Override
    public void postToMainThread(Runnable runnable) {
        if (mMainHandler == null) {
            synchronized (mLock) {
                if (mMainHandler == null) {
                    // MainLooper 主线程的 Looper 呢
                    mMainHandler = createAsync(Looper.getMainLooper());
                }
            }
        }
        //noinspection ConstantConditions
        mMainHandler.post(runnable);
    }
    
    private static Handler createAsync(@NonNull Looper looper) {
        if (Build.VERSION.SDK_INT >= 28) {
            return Handler.createAsync(looper);
        }
        if (Build.VERSION.SDK_INT >= 16) {
            try {
                return Handler.class.getDeclaredConstructor(Looper.class, Handler.Callback.class,
                        boolean.class)
                        .newInstance(looper, null, true);
            } catch (IllegalAccessException ignored) {
            } catch (InstantiationException ignored) {
            } catch (NoSuchMethodException ignored) {
            } catch (InvocationTargetException e) {
                return new Handler(looper);
            }
        }
        return new Handler(looper);
    }
}
```

最终是通过 Handler 的 post ~~ 太熟悉了。
看代码得知 Handler 的 Looper 是 MainLooper
So LiveData#postValue 最终使用主线程将新 value 分发给观察者。意味着我们可以在任何线程调用 postValue，而不用担心线程问题。因为 LiveData 内部做好了线程转换。

## 设置相同的值，订阅的观察者们会收到同样的值吗

结论：在分发值的过程中没有判断过新值是否等于老值的代码出现，只出现了一个判断 version，通过 demo 尝试也会发现，会收到同样的值的

setValue 调用了 dispatchingValue， postValue 调用了 setValue 所以最终也是调用 dispatchingValue

```Java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    // mDispatchingValue 是一个标记，为 true 表示正在分发 value，
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            // 1.
            considerNotify(initiator);
            initiator = null;
        } else {
            // 2.
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

```

从 setValue 中看到调用 dispatchingValue 时传入的参数是 null 所以我们先看注释第 2 点的情况，使用迭代器获取 ObserverWrapper 对象传到 considerNotify 方法。

```Java
private void considerNotify(ObserverWrapper observer) {
    // 观察者处于不活跃状态时，不往下走了
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    // 分发的时候判断当前观察者是否不活跃了
    if (!observer.shouldBeActive()) {
        // 更改 ObserverWrapper 的 mActive 为 false 非活跃状态
        observer.activeStateChanged(false);
        return;
    }
    // 版本号来了~~！
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    // 同步版本号，
    observer.mLastVersion = mVersion;
    // 这里终于调用了我们 LiveData.observe 方法传入的 Observer 对象的 onChanged 方法。
    // 终于形成了闭环
    observer.mObserver.onChanged((T) mData);
}

```

来看看版本号 LiveData#mVersion

```Java
static final int START_VERSION = -1;
private int mVersion;

// 当创建 LiveData 时传入了初始值，则 mVersion 的版本为 -1 + 1 也就是 0
public LiveData(T value) {
    mData = value;
    mVersion = START_VERSION + 1;
}

// 大多数我们会使用无参构造，这时候 mVersion 是 START_VERSION -1
/**
 * Creates a LiveData with no value assigned to it.
 */
public LiveData() {
    mData = NOT_SET;
    mVersion = START_VERSION;
}

```

那 mVersion 的值是什么时候更改的呢？
其实在刚刚分析 setValue 时就 ++ 了，回顾一下

```Java
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

```

而且我们也知道 postValue 最后会调用 setValue，所以 mVersion 值更改的只是就到这儿了。
这时候是不是可以分析一下第三个问题 3. 设置相同的值，订阅的观察者们会收到同样的值吗 在分发值的过程中没有判断过新值是否等于老值的代码出现，只出现了一个判断 version，通过 demo 尝试也会发现，会收到同样的值的哟。

## 粘性事件原理，怎么防止数据倒灌

粘性事件，说白了是被观察者原本有值，当新注册观察者时，被观察者会将旧值发送给观察者。

 LiveData 会分发值几种情况：

1. 调用 setValue 和 postValue 并且 LifecycleOwner 处于活跃状态时
2. LiveData 有值，并且处于活跃状态时，调用 LiveData#observe 订阅观察者
3. LiveData 有新值，也就是 ObserverWrapper 的 mLastVersion 小于 LiveData 的 mVersion，LifecycleOwner 从不活跃状态转为活跃状态时

在 LiveData 有值时，调用 LiveData#observe 订阅观察者，会收到旧值。

可以通过设置AtomicBoolean的方式解决

## observeForever怎么用

LiveData#observeForever 方法其实用的比较少，但是如果将 LiveData 作为事件总线机制或者配置之类时，它就会派上用场。

AlwaysActiveObserver 不依赖生命周期了，所以不会像 LifecycleBoundObserver 在生命周期变为 DESTROYED 时调用 LiveData#removeObserver 从 LiveData#mObservers Map 中移除自身，所以我们在使用 LiveData#observeForever 时应在不需要的时候调用 LiveData#removeObserver ，否则可能会发生内存泄露。

## 总结

### LiveData 如何感知生命周期的变化？

- Jetpack 引入了 Lifecycle，让任何组件都能方便地感知界面生命周期的变化。只需实现 `LifecycleEventObserver` 接口并注册给生命周期对象即可。
- LiveData 的数据观察者在内部被包装成另一个对象（实现了 `LifecycleEventObserver` 接口），它同时具备了数据观察能力和生命周期观察能力。

### LiveData 是如何避免内存泄漏的？

- LiveData 的数据观察者通常是匿名内部类，它持有界面的引用，可能造成内存泄漏。
- LiveData 内部会将数据观察者进行封装，使其具备生命周期感知能力。当生命周期状态为 DESTROYED 时，自动移除观察者。

### LiveData 是粘性的吗？若是，它是怎么做到的？

- LiveData 的值被存储在内部的字段中，直到有更新的值覆盖，所以值是持久的。
- 两种场景下 LiveData 会将存储的值分发给观察者。
  - 一是值被更新，此时会遍历所有观察者并分发之。
  - 二是新增观察者或观察者生命周期发生变化（至少为 STARTED），此时只会给单个观察者分发值。
- LiveData 的观察者会维护一个“值的版本号”，用于判断上次分发的值是否是最新值。该值的初始值是-1，每次更新 LiveData 值都会让版本号自增。
- LiveData 并不会无条件地将值分发给观察者，在分发之前会经历三道坎：
  - 1. 数据观察者是否活跃。
  - 2. 数据观察者绑定的生命周期组件是否活跃。
  - 3. 数据观察者的版本号是否是最新的。
- “新观察者”被“老值”通知的现象叫“粘性”。因为新观察者的版本号总是小于最新版号，且添加观察者时会触发一次老值的分发。

### 什么情况下 LiveData 会丢失数据？

- 在高频数据更新的场景下使用 LiveData.postValue() 时，会造成数据丢失。因为“设值”和“分发值”是分开执行的，之间存在延迟。值先被缓存在变量中，再向主线程抛一个分发值的任务。若在这延迟之间再一次调用 postValue()，则变量中缓存的值被更新，之前的值在没有被分发之前就被擦除了。

### 在 Fragment 中使用 LiveData 需注意些什么？

- 在 Fragment 中观察 LiveData 时使用viewLifecycleOwner而不是this。因为 Fragment 和 其中的 View 生命周期不完全一致。LiveData 内部判定生命周期为 DESTROYED 时，才会移除数据观察者。存在一种情况，当 Fragment 之间切换时，被替换的 Fragment 不执行 onDestroy()，当它再次展示时会再次订阅 LiveData，于是乎就多出一个订阅者。