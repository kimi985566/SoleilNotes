# React Native

## 解释React Native

React Native是Facebook引入的开源JavaScript框架。它用于为iOS和Android平台开发真实的本机移动应用程序。它仅使用JavaScript来构建移动应用程序。就像React, 它使用本机组件而不是使用Web组件作为构建块。它是跨平台的, 允许你编写一次代码, 并且可以在任何平台上运行。

React Native应用程序基于React(一种由Facebook开发的JavaScript库和XML-Esque标记(JSX))来创建用户界面。它针对移动平台而不是浏览器。它可以节省开发时间, 因为它允许你使用针对Android和iOS平台的单一语言JavaScript来构建应用程序。

## React Native有什么优势

- 跨平台：它提供了”编写一次并随处运行”的功能。它用于为Android, iOS和Windows平台创建应用。
- 性能：用React Native编写的代码被编译成本机代码, 从而使所有操作系统都可以在所有平台上以相同的方式提供更接近的本机外观和功能。
- 社区：React Native提供了一个由热情的开发人员组成的庞大社区, 他们随时准备帮助我们修复错误, 问题随时出现。
- 热重载：在开发过程中立即可以看到应用程序代码中的一些更改。如果更改业务逻辑, 则会将其反映实时重新加载到屏幕上。
- 加快开发速度：React Native有助于快速开发应用程序。它使用通用语言来构建适用于Android, iOS和Windows平台的应用程序, 从而加快了应用程序的部署, 交付和上市时间。
- JavaScript：JavaScript知识用于构建本机移动应用程序。

## React Native的缺点是什么？

- React Native仍然是新的且不成熟：React Native是Windows, Android和iOS编程语言中的新框架。它仍处于改进阶段, 可能会对应用程序产生负面影响。
- 学习困难：React Native无法快速学习, 尤其是对于应用程序开发领域的新手而言。
- 它缺乏安全性鲁棒性：React Native是一个开放源代码的JavaScript框架, 它很脆弱, 并且在安全性鲁棒性方面造成了差距。在创建数据高度机密的银行和金融应用程序时, 专家建议不要选择React Native。
- 初始化需要花费更多时间：即使使用高科技的小工具和设备, React Native也会花费大量时间来初始化运行时。
- 存在是不确定的：随着Facebook开发此框架, 它的存在是不确定的, 因为它保留了随时终止该项目的所有权利。随着React Native的流行, 这种情况不太可能发生。

## 列出React Native的基本组件

- 视图：它是用于构建移动应用程序UI的基本内置组件。该视图类似于HTML中的div。这是一个内容区域, 你可以在其中显示内容。
- 状态：用于控制组件。变量数据可以存储在状态中。这是可变的, 表示状态可以随时更改该值。
- 道具：道具用于将数据传递到不同的组件。这是不可变的, 表示道具无法更改值。它提供了容器组件和表示组件之间的连接。
- 样式：它是Web或移动设备中必不可少的组件, 使应用程序更具吸引力。 React Native不需要任何特殊的语言或语法来进行样式设置。它可以使用JavaScript对象为应用程序设置样式。
- 文字：此组件在应用程序中显示文字。它使用基本组件textInput从用户那里获取文本输入。
- ScrollView：这是一个用于容纳多个视图的滚动容器。它可用于通过滚动条在视图中呈现大型列表或内容。

## 在React Native中运行多少个线程？

- React Native UI线程(主线程)：此线程用于移动应用程序的布局。
- React Native JavaScript线程：这是运行我们的业务逻辑的线程。这意味着JS线程是执行我们的JavaScript代码的地方。
- React Native Modules Thread：有时候, 我们的应用需要访问平台API, 这是Native Module线程的一部分。
- React Native Render Thread：此线程用于生成用于绘制应用程序UI的实际OpenGL命令。

## 什么是React Native Apps

React Native Apps不是Web应用程序。这些类型的应用程序正在移动设备上运行, 并且无法通过浏览器加载。而且, 它们不是可在WebView组件上运行的基于Ionic, Phonegap等构建的混合应用程序。它们是使用单一语言JavaScript构建的真正的本机应用程序, 并具有可在移动设备上运行的本机组件。

## 列出创建和启动React Native App的步骤

1. 安装Node.js
2. 使用以下命令安装本地React环境
    > npm install -g create-react-native-app
3. 使用以下命令创建一个项目
    > create-react-native-app MyProject
4. 接下来, 使用以下命令浏览项目
5. 现在, 运行以下命令以启动项目
    > npm start

## React Native中的状态是什么

用于控制组件。变量数据可以存储在状态中。这是可变的, 表示状态可以随时更改该值。

在这里, 我们将使用状态数据创建一个Text组件。每当我们单击文本组件时, 其内容都会更新。事件onPress调用setState函数, 该函数使用” myState”文本更新状态。

```js
import React, {Component} from 'react';  
import { Text, View } from 'react-native';  
  
export default class App extends Component {  
    state = {  
        myState: 'This is a text component, created using state data. It will change or updated on clicking it.'  
    }  
    updateState = () => this.setState({myState: 'The state is updated'})  
    render() {  
        return (  
            <View>  
                <Text onPress={this.updateState}> {this.state.myState} </Text>  
            </View>  
        );  
    }  
}
```

## 所有React组件都可以在React Native中使用吗？

React Web组件使用DOM元素(例如div, h1, 表等)在UI上显示。但是, React Native不支持这些组件。你将需要找到专门针对React Native的库或组件。很难找到支持这两种功能的组件。但是, 应该很容易弄清楚给定的组件是否为React Native所用。因此, 很清楚所有组件都不能在React Native中使用。

## 虚拟DOM如何在React Native中工作？

虚拟DOM是一个轻量级的JavaScript对象, 它是真实DOM的内存表示形式。这是调用渲染函数和在屏幕上显示元素之间的中间步骤。它类似于节点树, 它列出了元素, 它们的属性以及作为对象及其属性的内容。渲染功能创建React组件的节点树, 然后响应于由用户或系统执行的各种操作导致的数据模型中的突变来更新此节点树。

虚拟DOM分为三个步骤：

- 每当React App中的任何数据发生更改时, 整个UI都会以虚拟DOM表示形式重新呈现。
- 现在, 将计算先前的DOM表示和新的DOM之间的差异。
- 一旦计算完成, 实际DOM仅使用那些已更改的内容进行更新。

## Android & iOS

### 我们可以在React Native中结合本地iOS或Android代码吗？

是的, 我们可以将本地iOS或Android代码与React Native结合在一起。它可以合并用Objective-C, Java和Shift编写的组件。

### 我们是否为Android和iOS使用相同的代码库？

是的, 我们可以为Android和iOS使用相同的代码库, React负责所有本机组件的翻译。例如, React Native ScrollView在Android上使用ScrollView, 在iOS上使用UiScrollView。

## React和React Native有什么区别？

React和React Native之间的本质区别是：

- React是一个JavaScript库, 而React Native是基于React的JavaScript框架。
- 标签在两个平台中的使用方式可能不同。
- React用于开发UI和Web应用程序, 而React Native可用于创建跨平台的移动应用程序。

## React Native和Native(Android和iOS)之间有什么区别？

React Native允许你编写一次并在任何地方运行。这意味着我们可以在Android和iOS平台上重用React Native代码。因为我们可以在两个平台之间重用大多数React Native代码, 但是Android和iOS是不同的系统。在这里, 我们将看到这些差异。

### 操作系统

你可以使用React Native来为Android和iOS构建应用程序, 但是如果你在Windows系统上工作, 要检查该应用程序是否在两个系统上都工作并不容易。 Windows不允许运行XCode及其模拟器, 后者是macOS应用。还有其他可用工具, 但它们不是官方的。

### 本机元素

这些元素对React Native和Native应用程序执行不同的操作。 React Native应用程序使用React Native库中的元素, 而Native应用程序不使用React Native库中的元素。

### 特定样式阴影

在跨平台应用程序上工作时, Shadows样式是iOS和Android之间差异的重要术语。 Android不支持阴影；取而代之的是, 它使用了height属性。

### 链接库

有时我们想在我们的应用程序中使用第三方库。在大多数情况下, 我们将其添加为依赖项, 但有时需要手动链接才能添加库。对于Web或本机应用程序的开发人员而言, 手动链接库并非易事。由于React Native处于改进阶段, 因此库文档不会根据最新框架进行更新。

## React Native中使用了什么XH​​R模块？

在React Native中, XHR模块用于实现XMLHttpRequest。它是与远程服务进行交互的对象。该对象由前端和后端两部分组成, 其中前端允许在JavaScript中进行交互。它将请求发送到XHR后端, 后者负责处理网络请求。后端部分称为网络。

## 类和功能组件之间有什么区别？

```js
//Functional Component
function WelcomeMessage(props) {  
  return <h1>Welcome to the , {props.name}</h1>;  
} 

//Class Component
class MyComponent extends React.Component {  
  render() {  
    return (  
      <div>This is main component.</div>  
    );  
  }  
}
```

- 语法：两个组件的声明不同。**功能组件**接受道具, 并返回React元素, 而**类组件**则需要从React扩展
- 状态：**类组件**具有状态, 而**功能组件**是无状态的。
- 生命周期：**类组件**具有生命周期, 而**功能组件**没有生命周期。

## React Native如何处理不同的屏幕尺寸？

1. Flexbox：用于在不同屏幕尺寸上提供一致的布局。它具有三个主要属性：
    - flexDirection
    - 证明内容
    - alignItems
2. 像素比率：用于使用PixelRatio类访问设备像素密度。如果我们在高像素密度的设备上, 我们将获得更高分辨率的图像。
3. 尺寸：用于处理不同的屏幕尺寸并精确设置页面样式。它仅需编写一次代码即可在任何设备上工作。
4. AspectRatio：用于设置高度, 反之亦然。 AspectRatio仅存在于React-Native中, 而不存在于CSS标准中。
5. ScrollView：这是一个滚动容器, 其中包含多个组件和视图。可滚动项目可以垂直和水平滚动。

## React和React Native之间有什么相似之处？

React和React Native之间最常见的相似之处是：

- React生命周期方法
- React组件
- React状态和道具
- Redux库

## React Native中的动画是什么？

动画是一种方法, 其中图像被操纵为显示为运动对象。

React Native动画允许你添加额外的效果, 从而在应用程序中提供出色的用户体验。

我们可以将其与React Native API, Animated.parallel, Animated.decay和Animated.stagger一起使用。

React Native有两种类型的动画

- 动画的：此API用于控制特定值。它具有用于控制基于时间的动画执行的启动和停止方法。
- LayoutAnimated：此API用于对全局布局事务进行动画处理。

## 为什么React Native使用Redux？

Redux是JavaScript应用程序的状态容器。它是一种状态管理工具, 可帮助你编写行为一致, 可以在不同环境中运行且易于测试的应用程序。

React Native使用Redux, 因为它允许开发人员将一个应用程序状态用作全局状态, 并可以轻松地与任何React组件中的状态进行交互。它可以与任何框架或库结合使用。

## 在页面上隐藏元素的方法有哪些？

> 首先我们要从多个纬度来回答这个问题，首先要根据你所面试的技术栈来回答。我们知道现在主流的2个前端框架Vue和React，其实都是基于虚拟节点来实现的。在回答这个问题的时候，先回答在不同的技术栈如何去隐藏元素。然后，阐述不同的隐藏方式他们都是怎么操作虚拟节点的。

1. css可以使用visibility属性

    - visible: 元素是可见的。
    - hidden: 元素是不可见的。
    - collapse: 当在表格元素中使用时，此值可删除一行或一列，但是它不会影响表格的布局。被行或列占据的空间会留给其他内容使用。如果此值被用在其他的元素上，会呈现为 "hidden"。
    - inherit: 规定应该从父元素继承 visibility 属性的值。

2. 在vue中

    - 我们可以使用：v-if、v-show。
    - 差异：v-if是操作元素的DOM节点创建元素和删除元素；v-show是操作元素的display属性。
    - 使用的场景：如果是单纯的元素显示隐藏不会涉及到权限、安全、页面展示的情况下一般使用v-show如果涉及到权限、安全、页面展示的情况下用v-if

3. react & react native

    - 用三元运算符

## 如何使用flex实现三栏布局，两边固定，中间自适应

```js
<View style={{ flex:1,flexDirection:'row',justifyContent:'space-between', }}>
    <View style={{backgroundColor:"red",flex:2}} />
    <View style={{backgroundColor:'black',flex:4}} />
    <View style={{backgroundColor:'yellow',flex:2}} />
</View>
```

## 何时在javascript中使用reduce()、map()、foreach()和filter()

- reduce()：方法接收一个函数作为累加器，数组中的每个值（从左到右）开始缩减，最终计算为一个值。
- map()、foreach()都是用于遍历List、Array
- filter()方法创建一个新的数组，新数组中的元素是通过检查指定数组中符合条件的所有元素。