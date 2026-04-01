# Widget
{docsify-updated}

Flutter 的核心思想是用 widget 来描述构建你的 UI 界面 。Widget 描述了在当前的配置和状态下视图所应该呈现的样子。当 widget 的状态改变时，它会重新构建其描述（展示的 UI），框架则会对比前后变化的不同，以确定底层渲染树从一个状态转换到下一个状态所需的最小更改。

在Flutter中，Widget的功能是“描述一个UI元素的配置数据”，它就是说，Widget其实并不是表示最终绘制在设备屏幕上的显示元素，而它只是描述显示元素的一个配置数据。Widget只是UI元素的一个配置数据，并且一个Widget可以对应多个Element 。
+ Widget实际上就是Element的配置数据，Widget树实际上是一个配置树，而真正的UI渲染树是由Element构成；不过，由于Element是通过Widget生成的，所以它们之间有对应关系，在大多数场景，我们可以宽泛地认为Widget树就是指UI控件树或UI渲染树。
+ 一个Widget对象可以对应多个Element对象。这很好理解，根据同一份配置（Widget），可以创建多个实例（Element）。

创建一个最小的 Flutter 应用简单到仅需调用 runApp() 方法并传入一个 widget 即可。runApp() 函数会持有传入的 Widget，并且使它成为 widget 树中的根节点，框架会强制让根 widget 铺满整个屏幕。Widget 的主要工作是实现 build 方法，该方法根据其它较低级别的子 widget 来描述这个 widget 。框架会逐一构建这些 widget，直到最底层的描述 widget 几何形状的 RenderObject。

`StatelessWidget` 是不可变的，这意味着它们的属性不能改变 —— 所有的值都是 final 。  
`Statefulwidgets` 持有的状态可能在 widget 生命周期中发生变化，实现一个 stateful widget 至少需要两个类： 
1. 一个 StatefulWidget 类；
2. 一个 State 类， StatefulWidget 类本身是不变的，但是 State 类在 widget 生命周期中始终存在