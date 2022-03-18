# 集锦

## Dart

Dart 综合了动态语言和静态语言的特性, 这里主要提供一些不一样，或者有意思的概念。

1. Dart 属于是强类型语言 ，但可以用 var  来声明变量，Dart 会自推导出数据类型，var 实际上是编译期的“语法糖”。dynamic 表示动态类型， 被编译后，实际是一个 object 类型，在编译期间不进行任何的类型检查，而是在运行期进行类型检查。
2. Dart 中 if 等语句只支持 bool 类型，switch 支持 String 类型。
3. Dart 中数组和 List 是一样的。
4. Dart 中，Runes 代表符号文字 ,  是 UTF-32 编码的字符串, 用于如  Runes input = new Runes('\u{1f596}  \u{1f44d}');
5. Dart 支持闭包。
6. Dart 中 number 类型分为 int 和 double ，没有 float 类型。
7. Dart 中 级联操作符 可以方便配置逻辑，如下代码:

```Dart
    event
    ..id = 1
    ..type = ""
    ..actor = "";
```

8. 赋值操作符

```Dart
AA ?? "999"  ///表示如果 AA 为空，返回999
AA ??= "999" ///表示如果 AA 为空，给 AA 设置成 999
AA ~/999 ///AA 对于 999 整除
```

9. 可选方法参数: Dart 方法可以设置 参数默认值 和 指定名称 。
10. 作用域: Dart 没有关键词 public 、private 等修饰符，_ 下横向直接代表 private ，但是有 @protected 注解 。
11. 构造方法: Dart 中的多构造方法，可以通过命名方法实现。默认构造方法只能有一个，而通过 Model.empty() 方法可以创建一个空参数的类，其实方法名称随你喜欢，而变量初始化值时，只需要通过 this.name 在构造方法中指定即可：

```Dart
class ModelA {
  String name;
  String tag;
  
  //默认构造方法，赋值给name和tag
  ModelA(this.name, this.tag);

  //返回一个空的ModelA
  ModelA.empty();
  
  //返回一个设置了name的ModelA
  ModelA.forName(this.name);
}
```

12. getter setter 重写

Dart 中所有的基础类型、类等都继承 Object ，默认值是 NULL， 自带 getter 和 setter ，而如果是 final 或者 const 的话，那么它只有一个 getter 方法，Object  都支持 getter、setter 重写：

```Dart
@override
  Size get preferredSize {
    return Size.fromHeight(kTabHeight + indicatorWeight);
}
```

13. Assert(断言)

assert 只在检查模式有效，在开发过程中， assert(unicorn == null); 只有条件为真才正常，否则直接抛出异常，一般用在开发过程中，某些地方不应该出现什么状态的判断。

14. 重写运算符，如下所示重载 operator 后对类进行 +/- 操作。

```Dart
class Vector {
  final int x, y;

  Vector(this.x, this.y);

  Vector operator +(Vector v) => Vector(x + v.x, y + v.y);
  Vector operator -(Vector v) => Vector(x - v.x, y - v.y);
  
  ···
}

void main() {
  final v = Vector(2, 3);
  final w = Vector(2, 2);

  assert(v + w == Vector(4, 5));
  assert(v - w == Vector(0, 1));
}
```

支持重载的操作符 ：

![](../../asset/dart%E9%87%8D%E8%BD%BD%E7%AC%A6.webp)

类、接口、继承

- Dart 中没有接口，类都可以作为接口，把某个类当做接口实现时，只需要使用 implements ，然后复写父类方法即可。
- Dart 中支持 mixins ，按照出现顺序应该为 extends 、 mixins 、implements 。

## Flutter

1. Flutter 和 React Native 不同主要在于 Flutter UI是直接通过 skia 渲染的 ，而 React Native 是将 js 中的控件转化为原生控件，通过原生去渲染的