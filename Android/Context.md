## 1 简介

Context 上下文对象，Android 中很多业务流程都涉及到 Context。

![](../asset/context.jpg)

Context 数量 = Activity 数量 + Service 数量 + Application 数量(1)

## 2 Context 对于Activity有什么作用

Activity 通过 Context接口 去访问Android系统的服务 & 资源，主要包括：

1. 获取应用相关信息
2. 获取系统/应用资源
3. 四大组件之间的交互
4. 文件相关
5. 数据库相关

## 3 不同类型的Context的应用场景是什么？

从上面可知，最终的Context类型主要包括：Activity、Service & Application，那么使用这三者的应用场景区别是什么呢

### case1：与UI相关的场景，都使用Activity类型Context

因为是针对当前UI界面资源进行操作 & 继承自ContextThemeWrapper（可自定义主题样式），如show a dialog、 Layout Inflation等。

### case2：生命周期较长的对象，都使用Application类型Context

因为Application Context的生命周期与应用保持一致，可避免出现Context引用的内存泄漏

其余场景，三种类型 基本可视为共用。

## 4 Application Context 的创建过程

在 Activity 启动流程源码分析的启动过程中, 最后一个重要的方法是performLaunchActivity, 源码如下：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    
    ...
    //创建 Application
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    ...
    return activity;
}

```

主要看 r.packageInfo.makeApplication -> LpadedApk

```java
@UnsupportedAppUsage
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        //不为 null 就直接返回 Application
        if (mApplication != null) {
            return mApplication;
        }
        ...
        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                ...
            }
            //创建 ContextImpl类型的 appContext
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            //创建了一个 app 实例 反射
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            ...
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;
        if (instrumentation != null) {
            try {
               //最终就调用了application.onCreate()方法
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                ...
            }
        }
        return app;
    }
```

