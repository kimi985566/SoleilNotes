# ViewModel

相关问题：

1. 为什么Activity旋转屏幕后ViewModel可以恢复数据。
2. ViewModel 的实例缓存到哪儿了。
3. 什么时候 ViewModel#onCleared() 会被调用。

## 分析

ViewModel出现之前，Activity 可以使用 onSaveInstanceState() 方法保存，然后从 onCreate() 中的 Bundle 恢复数据，但此方法仅适合可以序列化再反序列化的少量数据（IPC 对 Bundle 有 1M 的限制），而不适合数量可能较大的数据，如用户信息列表或位图。

ViewModel 的出现完美解决这个问题。

## ViewModel 的实例缓存到哪儿了

我们先看看 ViewModel 怎么创建的：

最终 ViewModel 的创建方法是：

```Kotlin
val mainViewModel = ViewModelProvider(this).get(MainViewModel::class.java)
```

> 创建`ViewModelProvider`对象并传入了`this`参数，然后通过`ViewModelProvider#get`方法，传入`MainViewModel`的`class`类型，然后拿到了`mainViewModel`实例。

`ViewModelProvider`的构造方法：

```Kotlin
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {

// 获取 owner 对象的 ViewModelStore 对象
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());

}
```

看看`ComponentActivity`发现:

```Java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements

        ...

        // 实现了 ViewModelStoreOwner 接口

        ViewModelStoreOwner,

        ...{

    private ViewModelStore mViewModelStore;

// 重写了 ViewModelStoreOwner 接口的唯一的方法 getViewModelStore()
    @NonNull
    @Override

    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");

        }

        ensureViewModelStore();
        return mViewModelStore;

    }
```

> `ComponentActivity`类实现了`ViewModelStoreOwner`接口。

再看看刚刚的`ViewModelProvider`构造方法里调用了`this(ViewModelStore, Factory)`，将`ComponentActivity#getViewModelStore`返回的`ViewModelStore`实例传了进去，并缓存到`ViewModelProvider`中:

```Java
public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    // 缓存 ViewModelStore 对象
    mViewModelStore = store;

}
```

接着看`ViewModelProvider#get`方法做了什么:

```Java
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}
```

> 获取 ViewModel 的 CanonicalName , 调用了另一个 get 方法。

```Java
@MainThread

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {

// 从 mViewModelStore 缓存中尝试获取
    ViewModel viewModel = mViewModelStore.get(key);
// 命中缓存
    if (modelClass.isInstance(viewModel)) {

        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        // 返回缓存的 ViewModel 对象
        return (T) viewModel;

    } else {

        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }

    }

    // 使用工厂模式创建 ViewModel 实例

    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
    } else {
        viewModel = mFactory.create(modelClass);
    }

    // 将创建的 ViewModel 实例放进 mViewModelStore 缓存中
    mViewModelStore.put(key, viewModel);
    // 返回新创建的 ViewModel 实例

    return (T) viewModel;

}
```

> mViewModelStore 是啥？通过 ViewModelProvider 的构造方法知道 mViewModelStore 其实是我们 Activity 里的 mViewModelStore 对象，它在 ComponentActivity 中被声明。

看到了 put 方法，不难猜它内部用了 Map 结构。

### ViewModel 的实例缓存到哪儿了总结

ViewModel 对象存在在 ComponentActivity 的 mViewModelStore 对象中。

## 为什么Activity旋转屏幕后ViewModel可以恢复数据

转换思路`mViewModelStore`出现频率这么高，何不看看它是什么时候被创建的呢？

记不记得刚才看`ViewModelProvider`的构造方法时 ，获取`ViewModelStore`对象时，

实际调用了`MainActivity#getViewModelStore()`，

而`getViewModelStore()`实现在`ComponentActivity`中。

```Java
// ComponentActivity#getViewModelStore()

@Override

public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }

    ensureViewModelStore();
    return mViewModelStore;

}
```

在返回`mViewModelStore`对象之前调用了`ensureViewModelStore()`。

```Java
void ensureViewModelStore() {

    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();

        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }

        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }

    }

}
```

> 当`mViewModelStore == null`调用了`getLastNonConfigurationInstance()`获取`NonConfigurationInstances`对象`nc`，当`nc != null`时将`mViewModelStore`赋值为`nc`.`viewModelStore`，最终`viewModelStore == null`时，才会创建`ViewModelStore`实例。

不难发现，之前创建的 viewModelStore 对象被缓存在 NonConfigurationInstances 中。

```Java
/ 它是 ComponentActivity 的静态内部类

static final class NonConfigurationInstances {

    Object custom;
    // 果然在这儿
    ViewModelStore viewModelStore;

}
```

`NonConfigurationInstances`对象通过`getLastNonConfigurationInstance()`来获取的。

> `onRetainNonConfigurationInstance`方法和`getLastNonConfigurationInstance`是成对出现的，跟`onSaveInstanceState（Bundle`机制类似，只不过它是仅用作处理配置更改的优化。

看看 onRetainNonConfigurationInstance 方法：

```Java
**
* 保留所有适当的非配置状态
*/

@Override
@Nullable
@SuppressWarnings("deprecation")

public final Object onRetainNonConfigurationInstance() {

    // Maintain backward compatibility.
    Object custom = onRetainCustomNonConfigurationInstance();

    ViewModelStore viewModelStore = mViewModelStore;

    // 若 viewModelStore 为空，则尝试从 getLastNonConfigurationInstance() 中获取
    if (viewModelStore == null) {
        // No one called getViewModelStore(), so see if there was an existing
        // ViewModelStore from our last NonConfigurationInstance
        NonConfigurationInstances nc =
               (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }

    }

// 依然为空，说明没有需要缓存的，则返回 null
    if (viewModelStore == null && custom == null) {
        return null;
    }

// 创建 NonConfigurationInstances 对象，并赋值 viewModelStore
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;

    return nci;

}
```

### 为什么Activity旋转屏幕后ViewModel可以恢复数据总结

1. Activity 在因配置更改而销毁重建过程中会先调用 onRetainNonConfigurationInstance 保存 viewModelStore 实例
2. 在重建后可以通过 getLastNonConfigurationInstance 方法获取之前的 viewModelStore 实例。

## 什么时候 ViewModel#onCleared() 会被调用

```Java
public abstract class ViewModel {

    protected void onCleared() {

    }

    @MainThread
    final void clear() {

        mCleared = true;
        // Since clear() is final, this method is still called on mock objects
        // and in those cases, mBagOfTags is null. It'll always be empty though
        // because setTagIfAbsent and getTag are not final so we can skip
        // clearing it

        if (mBagOfTags != null) {

            synchronized (mBagOfTags) {
                for (Object value : mBagOfTags.values()) {
                    // see comment for the similar call in setTagIfAbsent
                    closeWithRuntimeException(value);
                }

            }

        }

        onCleared();

    }

}
```

> onCleared() 方法被 clear() 调用了

刚才看`ViewModelStore`源码时好像是调用了`clear()`，回顾一下：

```Java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {

        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }

    }


    final ViewModel get(String key) {
        return mMap.get(key);
    }


    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }


    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {

        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }

        mMap.clear();

    }

}
```

在`ViewModelStore`的`clear()`中，遍历`mMap`并调用`ViewModel`对象的`clear()`，再看`ViewModelStore`的 clear() 什么时候被调用的：

```Java
// ComponentActivity 的构造方法

public ComponentActivity() {

    ... 

    getLifecycle().addObserver(new LifecycleEventObserver() {

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {

            if (event == Lifecycle.Event.ON_DESTROY) {
                // Clear out the available context
                mContextAwareHelper.clearAvailableContext();
                // And clear the ViewModelStore
                if (!isChangingConfigurations()) {
                    getViewModelStore().clear();

                }
            }
        }

    });

    ...

}
```

> 观察当前 activity 生命周期，当 Lifecycle.Event == Lifecycle.Event.ON_DESTROY，并且 isChangingConfigurations() 返回 false 时才会调用 ViewModelStore#clear 。

```Java
// Activity#isChangingConfigurations()

    /**
     * Check to see whether this activity is in the process of being destroyed in order to be
     * recreated with a new configuration. This is often used in
     * {@link #onStop} to determine whether the state needs to be cleaned up or will be passed
     * on to the next instance of the activity via {@link #onRetainNonConfigurationInstance()}.
     *
     * @return If the activity is being torn down in order to be recreated with a new configuration,
     * returns true; else returns false.
     */

    public boolean isChangingConfigurations() {

        return mChangingConfigurations;

    }
```

isChangingConfigurations 用来检测当前的 Activity 是否因为 Configuration 的改变被销毁了, 配置改变返回 true，非配置改变返回 false。

### 什么时候 ViewModel#onCleared() 会被调用 解决总结

在 activity 销毁时，判断如果是非配置改变导致的销毁， getViewModelStore().clear() 才会被调用。

## 总结

1. 什么时候 ViewModel#onCleared() 会被调用：在 activity 销毁时，判断如果是非配置改变导致的销毁， getViewModelStore().clear() 才会被调用。
2. ViewModel 的实例缓存到哪儿了：ViewModel 对象存在在 ComponentActivity 的 mViewModelStore 对象中。
3. 为什么Activity旋转屏幕后ViewModel可以恢复数据
    1. Activity 在因配置更改而销毁重建过程中会先调用 onRetainNonConfigurationInstance 保存 viewModelStore 实例
    2. 在重建后可以通过 getLastNonConfigurationInstance 方法获取之前的 viewModelStore 实例。
