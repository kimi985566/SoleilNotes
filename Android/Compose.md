# Compose

## Compose 管理状态

### 状态和组合

由于 Compose 是声明式工具集，因此更新它的唯一方法是通过新参数调用同一可组合项。这些参数是界面状态的表现形式。每当状态更新时，都会发生重组。

### 可组合项中的状态

可组合函数可以使用 remember 可组合项记住单个对象。系统会在初始组合期间将由 remember 计算的值存储在组合中，并在重组期间返回存储的值。remember 既可用于存储可变对象，又可用于存储不可变对象。

在可组合项中声明 MutableState 对象的方法有三种：

- val mutableState = remember { mutableStateOf(default) }
- var value by remember { mutableStateOf(default) }
- val (value, setValue) = remember { mutableStateOf(default) }

### 其他受支持的状态类型

在 Jetpack Compose 中读取其他可观察类型之前，您必须将其转换为 State<T>，以便 Jetpack Compose 可以在状态发生变化时自动重组界面。

Compose 附带一些可以根据 Android 应用中使用的常见可观察类型创建 State<T> 的函数：

- LiveData
- Flow
- RxJava2

任何允许 Jetpack Compose 订阅每项更改的对象都可以转换为 State<T> 并由可组合项读取。使用 remember 存储对象的可组合项会创建内部状态，使该可组合项有状态。

无状态可组合项是指不保持任何状态的可组合项。

在开发可重复使用的可组合项时，您通常想要同时提供同一可组合项的有状态和无状态版本。有状态版本对于不关心状态的调用方来说很方便，而无状态版本对于需要控制或提升状态的调用方来说是必要的。

### 状态提升

Compose 中的状态提升是一种将状态移至可组合项的调用方以使可组合项无状态的模式。

常规状态提升模式是将状态变量替换为两个参数：

- value: T：要显示的当前值
- onValueChange: (T) -> Unit：请求更改值的事件，其中 T 是建议的新值

具有一些重要的属性：

- 单一可信来源
- 封装
- 可共享
- 可拦截
- 解耦

状态下降、事件上升的这种模式称为“单向数据流”。

提升状态时，有三条规则可帮助您弄清楚状态应去向何处：

- 状态应至少提升到使用该状态（读取）的所有可组合项的最低共同父项。
- 状态应至少提升到它可以发生变化（写入）的最高级别。
- 如果两种状态发生变化以响应相同的事件，它们应一起提升。

### 在 Compose 中恢复状态

在重新创建 activity 或进程后，您可以使用 rememberSaveable 恢复界面状态。rememberSaveable 可以在重组后保持状态。此外，rememberSaveable 也可以在重新创建 activity 和进程后保持状态。