# Kotlin委托

## 什么是委托

讲Kotlin的委托之前，要先讲一下委托模式。委托模式又称代理模式，是常见的设计模式。

委托模式类似于我们生活中的代理、代购、中介。有些东西我们很难直接去买或者不知道怎么去买，但是我们能通过代理、代购、中介等方式间接去购买，这样我们也有具有了购买该东西的能力。

那代码怎么实现呢？首先我们定义一个接口，声明一个购买方法：

```kotlin
interface Subject {
  fun buy()
}
```

然后创建一个代理类实现该购买功能：

```kotlin
class Delegate : Subject {
  override fun buy() {
    print("I can buy it because I live abroad.")
  }
}
```

在某个类需要该功能但是没法直接实现的时候，就能通过代理间接地实现功能。

```kotlin
class RealSubject : Subject {

  private val delegate: Subject = Delegate()

  override fun buy() {
    delegate.buy()
  }
} 

```

> 总结一下，委托（代理）模式其实就是将接口功能交给另一个接口实例对象去实现。所以委托模式是有模板代码的，每个接口方法调用了对应的代理对象方法。

虽然存在模板代码，但是Java没有什么好的办法能生成模板代码， 而Kotlin可以零模板代码地原生支持它。通过by关键字进行委托，我们就能把上面的代码改成：

```kotlin
class RealSubject : Subject by Delegate()
```

这个委托的代码最终会生成上面的代码，简而言之就是编译器帮我们生成了模板代码。另外by关键字后面的表达式可能会有很多写法：

```kotlin
class RealSubject(delegate: Subject) : Subject by delegate

class RealSubject : Subject by globalDelegate

class RealSubject : Subject by GlobalDelegate

class RealSubject : Subject by delegate {...}
```

虽然写法有很多种，但是记住一点，by后面的表达式一定是得到一个接口的实例对象。因为接口功能是要委托给一个具体的实例对象，而这个对象可能通过构造函数、顶级属性、单例、方法等方式得到。

对此不了解的话，看到by后面各式各样的写法会很懵。其实不是什么Kotlin语法，只是为了得到个对象而已。

## 什么是属性委托

接口是把接口方法委托出去，那属性要委托什么呢？其实也很容易想到，属性能把get、set方法委托出去。

```kotlin
val delegate = Delegate()
var message: String
  get() = delegate.getValue()
  set(value) = delegate.setValue(value)
```

Kotlin支持用by关键字将属性委托给一个实例对象，这就是Kotlin的属性委托：

```kotlin
var message: String by Delegate()
```

当然属性的委托类不能随便写，有一套具体的规则。先来看一个委托类的示例：

```kotlin
class Delegate {

  operator fun getValue(thisRef: Any?, property: KProperty<*>): String =
    "$thisRef, thank you for delegating '${property.name}' to me!"

  operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) =
    println("$value has been assigned to '${property.name}' in $thisRef.")
}
```

可能有些人看了就懵了，怎么这么多东西，为什么要这么写？

其实是有一套固定的模板，不过不用特地去背，因为官方提供了接口类给我们快速实现。

```kotlin
public interface ReadWriteProperty<in T, V> : ReadOnlyProperty<T, V> {

  public override operator fun getValue(thisRef: T, property: KProperty<*>): V

  public operator fun setValue(thisRef: T, property: KProperty<*>, value: V)
}
```

- 首先来看一下方法的第一个参数thisRef，顾名思义这是this的引用。因为这个属性可能是在一个类里面，可能需要调用该类的方法，这时如果连外层的this引用都拿不到，那还谈何委托。不过我们可能用不到外层的类，这就可以像上面的示例那样定义为Any?类型。
- 然后是第二个参数property，必须是KProperty<*>类型或其超类型。这个参数可以拿到属性的信息，比如属性的名字。为什么要这个参数呢？因为可能要根据不同的属性实现不同的业务。举个例子，假如我们是做房地产中介，我们不可能什么人都推荐一样的房子，有钱人可能要买别墅的。我们会根据不同的客户信息反馈不同的结果，这就需要拿到客户的资料。同理属性的委托类需要能拿到属性的信息。
- 还有最重要的一点，需要有operator关键字修饰的getValue()和setValue()方法。这就涉及到另一个Kotlin的进阶用法——重载操作符。先讲个场景让大家来了解重载操作符，比如我们写了一个类，我们并不能像Int类型那样使用a + b的方式把两个对象相加。但是重载操作符能帮我们做到这事，我们增加 operator 关键字修饰的 plus() 方法之后就能a + b了。还能重载很多操作符，如a++、a > b等，大家有兴趣自行去了解。能重载的方法名是固定的，比如重载plus()方法就对应加法操作。而重载getValue()和setValue()方法是对应该类的get、set方法。属性委托就是把属性的get、set方法委托给代理类的get、set方法。

以上都是一个属性委托的必要条件，你可能不用，但是你不能没有。只要委托实例不满足条件，编译器就会报错。

Kotlin标准库还提供了几种常用的标准委托，方便我们在一些常用的场景快速实现属性委托。

## 延时委托

用于延时初始化的场景。因为委托的逻辑是固定的，官方已经帮我们写好了委托类代码，提供了一个lazy()方法快速创建委托类的实例。

```kotlin
val loadingDialog by lazy { LoadingDialog(this) }
```

首次获取属性的值会执行返回lambda表达式的结果，后续获取属性都是直接拿缓存。其实就是执行了以下的逻辑。

```kotlin
private var _loadingDialog: LoadingDialog? = null
val loadingDialog: LoadingDialog
  get() {
    if (_loadingDialog == null) {
      _loadingDialog = LoadingDialog(this)
    }
    return _loadingDialog!!
  }
```

lazy()方法返回的是Lazy类的对象，那么编译器生成的委托类会是Lazy而不是前面的ReadWriteProperty。

```kotlin
val delegate: Lazy = SynchronizedLazyImpl<LoadingDialog>(...)
val loadingDialog: LoadingDialog
  get() = delegate.value
```

## 可观察的委托

能够很方便地实现观察者模式。委托类的代码也是固定的，所以官方提供了Delegates.observable()方法创建委托的实例。每次设置属性的值都能在lambda表达式接收到回调。

```kotlin
var name: String by Delegates.observable("<no name>") { prop, old, new ->
  println("$old -> $new")
}
```

该方法返回一个ObservableProperty对象，继承自ReadWriteProperty。内部实现很简单，就是在setValue()的时候执行回调方法。

## 委托映射的属性值

简单来说就是把一个属性委托给map。

```kotlin
class User(val map: Map<String, Any?>) {
  val name: String by map
}
```

获取上面的属性其实是用map获取键名为name的值，编译器会生成以下逻辑。

```kotlin
class User(val map: Map<String, Any?>) {
  val name: String 
    get() = map["name"] as String
}
```

## 属性委托给属性

听起来有点绕，讲一个我遇过的问题来让大家更好地理解这个特性。Jetpack MVVM 架构是让数据写在Repository层，Activity需要通过ViewModel去操作Repository的数据。所以我之前写了以下逻辑：

```kotlin
object DataRepository {
  var isFirstLaunch: Boolean by MMKVDelegate()
}

class GuideViewModel : ViewModel() {
  var isFirstLaunch = DataRepository.isFirstLaunch
}

// In GuideActivity.kt
viewModel.isFirstLaunch = false
```

简单来说就是ViewModel转发了Repository属性委托的属性，但代码执行下来并没有调用属性委托的set()方法，导致数据没保存到。这段代码看似没什么问题，实则是有陷阱的。

问题出在ViewModel，等号的写法是有问题的，那只是个赋值操作，并不是持有引用，所以修改ViewModel的属性不会影响到Repository的属性。我当时发现问题后改为了下面的写法：

```kotlin
class GuideViewModel : ViewModel() {
  var isFirstLaunch: Boolean
    get() = DataRepository.isFirstLaunch
    set(value) {
      DataRepository.isFirstLaunch = value
    }
}
```

逻辑上没什么问题了，但是代码看起来很丑陋。后来发现Kotlin在1.4版本新增了特性来应对这种情况，能将一个属性委托给另一个属性，这样就完美解决了。

```kotlin
class GuideViewModel : ViewModel() {
  var isFirstLaunch by DataRepository::isFirstLaunch
}
```

## 小结

Kotlin委托其实就是使用by语法后，编译器会帮我们生成了委托类的代码。如果是接口的委托：

```kotlin
class RealSubject : Subject by Delegate()
```

编译器就会帮我们生成委托模式的代码：

```kotlin
class RealSubject : Subject {

  private val delegate: Subject = Delegate()

  override fun buy() {
    delegate.buy()
  }
}
```

如果是属性的委托：

```kotlin
var name: String by PropertyDelegate()
```

编译器就会帮我们把属性的get、set方法委托出去：

```kotlin
val delegate = PropertyDelegate()
var name: String
  get() = delegate.getValue()
  set(value) = delegate.setValue(value)
```

by关键字后面的表达式可能有各种各样的写法，但一定是返回一个委托的实例。
