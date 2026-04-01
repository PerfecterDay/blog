# Flutter 入门
{docsify-updated}

### 安装 Flutter
1. 下载安装 flutter sdk: `scoop install flutter`
2. vscode 安装 flutter 插件

### 构建第一个 Flutter 应用
1. vscode 打开命令面板 ——> `Flutter: New Application Project`
2. 删除 lib/main.dart 中的所有代码，然后替换为下面的代码，它将在屏幕的中心显示”Hello World”。
   ```
    import 'package:flutter/material.dart';
    void main() => runApp(MyApp());

    class MyApp extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
        return MaterialApp(
        title: 'Welcome to Flutter',
        home: Scaffold(
            appBar: AppBar(
            title: Text('Welcome to Flutter'),
            ),
            body: Center(
            child: Text('Hello World'),
            ),
        ),
        );
    }
    }
   ```
3. 按 F5 可以运行程序

+ 本示例创建了一个具有 Material Design 风格的应用， Material 是一种移动端和网页端通用的视觉设计语言， Flutter 提供了丰富的 Material 风格的 widgets。在 pubspec.yaml 文件的 flutter 部分选择加入 uses-material-design: true 会是一个明智之举，通过这个可以让您使用更多 Material 的特性，比如其预定义好的 图标 集。
+ 主函数（main）使用了 (=>) 符号，这是 Dart 中单行函数或方法的简写。
+ 该应用程序继承了 StatelessWidget，这将会使应用本身也成为一个 widget。在 Flutter 中，几乎所有都是 widget，包括对齐 (alignment)、填充 (padding) 和布局 (layout)。
+ Scaffold 是 Material 库中提供的一个 widget，它提供了默认的导航栏、标题和包含主屏幕 widget 树的 body 属性。 widget 树可以很复杂。
+ 一个 widget 的主要工作是提供一个 build() 方法来描述如何根据其他较低级别的 widgets 来显示自己。
+ 本示例中的 body 的 widget 树中包含了一个 Center widget， Center widget 又包含一个 Text 子 widget， Center widget 可以将其子 widget 树对齐到屏幕中心。

### 路由管理
路由(Route)在移动开发中通常指页面（Page），这跟web开发中单页应用的Route概念意义是相同的，Route在Android中通常指一个Activity，在iOS中指一个ViewController。所谓路由管理，**就是管理页面之间如何跳转，通常也可被称为导航管理**。Flutter中的路由管理和原生开发类似，无论是Android还是iOS，**导航管理都会维护一个路由栈，路由入栈(push)操作对应打开一个新页面，路由出栈(pop)操作对应页面关闭操作，而路由管理主要是指如何来管理路由栈。**

1. Navigator
   `Navigator` 是一个路由管理的组件，它提供了打开和退出路由页方法。Navigator通过一个栈来管理活动路由集合。通常当前屏幕显示的页面就是栈顶的路由。Navigator提供了一系列方法来管理路由栈，在此我们只介绍其最常用的两个方法：
   1. `Future push(BuildContext context, Route route)`
      将给定的路由入栈（即打开新的页面），返回值是一个Future对象，用以接收新路由出栈（即关闭）时的返回数据。
   2. `bool pop(BuildContext context, [ result ])`
      将栈顶路由出栈，result为页面关闭时返回给上一个页面的数据。

2. MaterialPageRoute
   `Navigator` 方法的参数都是 `Route` 类型的。
   `MaterialPageRoute` 继承自 `PageRoute` 类， `PageRoute` `类是一个抽象类，表示占有整个屏幕空间的一个模态路由页面，它还定义了路由构建及切换时过渡动画的相关接口及属性。MaterialPageRoute` 是Material组件库提供的组件，它可以针对不同平台，实现与平台页面切换动画风格一致的路由切换动画。
   `MaterialPageRoute` 构造函数:
   ```
    MaterialPageRoute({
        WidgetBuilder builder,
        RouteSettings settings,
        bool maintainState = true,
        bool fullscreenDialog = false,
    })
   ```
    + builder 是一个WidgetBuilder类型的回调函数，它的作用是构建路由页面的具体内容，返回值是一个widget。我们通常要实现此回调，返回新路由的实例。
    + settings 包含路由的配置信息，如路由名称、是否初始路由（首页）。
    + maintainState：默认情况下，当入栈一个新路由时，原来的路由仍然会被保存在内存中，如果想在路由没用的时候释放其所占用的所有资源，可以设置maintainState为false。
    + fullscreenDialog表示新的路由页面是否是一个全屏的模态对话框，在iOS中，如果fullscreenDialog为true，新页面将会从屏幕底部滑入（而不是水平方向）。
  
### 资源管理