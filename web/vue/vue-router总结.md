
# vue-router总结
{docsify-updated}




用 Vue.js + Vue Router 创建单页应用，是非常简单的。使用 Vue.js ，我们已经可以通过组合组件来组成应用程序，当你要把 Vue Router 添加进来，我们需要做的是，
+ 将组件 (components) 映射到路由 (routes)，
+ 然后告诉 Vue Router 在哪里渲染它们。

### 组件路由映射
通常使用一个数组来定义组件与路由之间的映射关系，将数组作为选项传递给 VueRouter 构造函数：

    const router = new VueRouter({
        routes: [
            // 动态路径参数 以冒号开头
            { path: '/user/', component: User ,children:[],name:'user'}
        ]
    })
数组中的每一项描述了一条或多条路由规则，
#### 动态路由匹配
我们经常需要把某种模式匹配到的所有路由，全都映射到同个组件。例如，我们有一个 User 组件，对于所有 ID 各不相同的用户，都要使用这个组件来渲染。那么，我们可以在 vue-router 的路由路径中使用“动态路径参数”(dynamic segment) 来达到这个效果：
    
    const router = new VueRouter({
        routes: [
            // 动态路径参数 以冒号开头
            { path: '/user/:id', component: User }
        ]
    })
一个“路径参数”使用冒号 : 标记。当匹配到一个路由时，参数值会被设置到 `this.$route.params` ，可以在每个组件内使用。

可以在一个路由中设置多段“路径参数”，对应的值都会设置到 $route.params 中:

| 模式|匹配路径|$route.params|
|:---:|:--:|:--:|
|/user/:username|	 /user/evan|	{ username: 'evan' }|
|/user/:username/post/:post_id|	 /user/evan/post/123|	{ username: 'evan', post_id: 123 }|

#### 高级匹配模式
`vue-router` 使用 `path-to-regexp` 作为路径匹配引擎，所以支持很多高级的匹配模式，例如：可选的动态路径参数、匹配零个或多个、一个或多个，甚至是自定义正则匹配。

### 匹配优先级
有时候，同一个路径可以匹配多个路由，此时，匹配的优先级就按照路由的定义顺序：谁先定义的，谁的优先级就最高。




