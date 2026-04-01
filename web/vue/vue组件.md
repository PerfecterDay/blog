
# Vue组件
{docsify-updated}

vue组件主要是封装dom、dom所需要展示的数据及其的展现方式的封装。组件可以扩展 HTML 元素，封装可重用的代码。
 
## veu组件全局注册
组件在注册之后，便可以作为自定义元素在一个实例的模板中使用。注意确保在初始化根实例之前注册组件：
```
<div id="example">
    <my-component></my-component>
</div>

// 注册,在所有其它组件中可用
Vue.component('my-component', {
    template: '<div>A custom component!</div>'
})
// 创建根实例
new Vue({
    el: '#example'
})
```
会被渲染为       
```
<div id="example">
    <div>A custom component!</div>
</div>
```

## 局部注册
你不必把每个组件都注册到全局。你可以通过某个 Vue 实例/组件的实例选项 components 注册仅在其作用域中可用的组件：
```
var Child = {
    template: '<div>A custom component!</div>',
    data: function () {
    return {
        count: 0
    }
    }
}
new Vue({
    components: {
    // <my-component> 将只在父组件模板中可用
    'my-component': Child
    }
})
```
这种封装也适用于其它可注册的 Vue 功能，比如指令。

***一个组件的 data 选项必须是一个函数，因此每个实例可以维护一份被返回对象的独立的拷贝。如果 Vue 没有这条规则，多处用到的组件实例之间会相互影响。***


## 基础组件的自动化全局注册
可能你的许多组件只是包裹了一个输入框或按钮之类的元素，是相对通用的。我们有时候会把它们称为基础组件，它们会在各个组件中被频繁的用到。

所以会导致很多组件里都会有一个包含基础组件的长列表：
```
import BaseButton from './BaseButton.vue'
import BaseIcon from './BaseIcon.vue'
import BaseInput from './BaseInput.vue'

export default {
components: {
    BaseButton,
    BaseIcon,
    BaseInput
}
}
```

可以在一个文件中注册所有的基础组件。

## 单文件组件
将一个组件封装到一个文件中，其中可以包含 html , css , JS 代码。 使用 `vue-loader` .

## 父子组件之间的数据交互
开发组件的目的一是为了缩小问题规模，减小复杂度；更多的情况是复用代码，组件一般是可重用的代码。因此，在开发组件时就应该考虑到可重用性问题。

例如开发了一个展示博客的组件，每条博文都有标题、作者、日期、浏览量等信息。组件展示这些信息的时候，数据必须由父组件传递过来，组件内对数据的展示及处理方式是可重用的。因此，就需要一种父组件向子组件传递数据的一种方式。

有时候我们在子组件中做了某些操作之后，又需要去操作父组件中的某个值。

父子组件之间通过什么方式来传递数据呢？

### 通过 Prop 向子组件传递数据
Prop 使你可以在组件上注册的一些自定义特性。当一个值传递给一个 prop 特性的时候，它就变成了那个组件实例的一个属性。为了给博文组件传递一个标题，我们可以用一个 props 选项将其包含在该组件可接受的 prop 列表中：
```
Vue.component('blog-post', {
    props: ['title'],
    template: '<h3>{{ title }}</h3>'
})
```
一个组件默认可以拥有任意数量的 prop，任何值都可以传递给任何 prop。在上述模板中，你会发现我们能够在组件实例中访问这个值，就像访问 data 中的值一样。

一个 prop 被注册之后，你就可以像这样把数据作为一个自定义特性传递进来：
```
<blog-post title="My journey with Vue"></blog-post>
<blog-post title="Blogging with Vue"></blog-post>
<blog-post title="Why Vue is so fun"></blog-post>
```
然而在一个典型的应用中，你可能在 data 里有一个博文的数组：
```
new Vue({
el: '#blog-post-demo',
data: {
    posts: [
    { id: 1, title: 'My journey with Vue' },
    { id: 2, title: 'Blogging with Vue' },
    { id: 3, title: 'Why Vue is so fun' }
    ]
}
})
```
并想要为每篇博文渲染一个组件：

```
<blog-post
v-for="post in posts"
v-bind:key="post.id"
v-bind:title="post.title"
></blog-post>
```

如上所示，你会发现我们可以使用 `v-bind` 来动态传递 prop。这在你一开始不清楚要渲染的具体内容，比如从一个 API 获取博文列表的时候，是非常有用的。

### 单根元素和对象传递
***单个组件中只能包含一个跟元素***当组件需要渲染多元素时，必须用一个父元素将它们包裹起来。
假如向子元素传递太多属性，则要在子元素的 prop 中全部声明，且父组件在传递的时候也要为每个 prop 传递数据。例如，上面的博文组件，我们要位组件传递标题、作者、日期、浏览量等信息，代码量多且杂。

这时候可以考虑将要传递的数据封装成一个对象传递给子组件，然后在子组件内部使用对象取值的方式获取数据。

```
<blog-post
v-for="post in posts"
v-bind:key="post.id"
v-bind:post="post"
></blog-post>

Vue.component('blog-post', {
props: ['post'],
template: `
    <div class="blog-post">
    <h3>{{ post.title }}</h3>
    <div v-html="post.content"></div>
    </div>
`
})
```

### 通过事件向父级组件发送消息
Vue 实例提供了一个自定义事件的系统来解决这个问题。我们可以调用内建的 `$emit` 方法并传入事件的名字，来向父级组件触发一个事件，例如在子组件中有个 button ，当用户点击 button 时，更新父组件中的一个值：
```
<button v-on:click="$emit('enlarge-text', 0.1)">
Enlarge text
</button>
```

然后当在父级组件监听这个事件的时候，我们可以通过 `v-on` 监听这个事件:
```    
<blog-post
...
v-on:enlarge-text="postFontSize += 0.1"
></blog-post>
```

#### 使用事件抛出一个值
有的时候用一个事件来抛出一个特定的值是非常有用的。例如我们可能想让 <blog-post> 组件决定它的文本要放大多少。这时可以使用 $emit 的第二个参数来提供这个值：
```
<button v-on:click="$emit('enlarge-text', 0.1)">
    Enlarge text
</button>
```
然后当在父级组件监听这个事件的时候，我们可以通过 `$event` 访问到被抛出的这个值：
```
<blog-post
...
v-on:enlarge-text="postFontSize += $event"
></blog-post>
```
或者，如果这个事件处理函数是一个方法：
```
<blog-post
...
v-on:enlarge-text="onEnlargeText"
></blog-post>
```
那么这个值将会作为第一个参数传入这个方法：
```
methods: {
    onEnlargeText: function (enlargeAmount) {
        this.postFontSize += enlargeAmount
    }
}
```
#### 自定义输入组件上使用 `v-model`
自定义事件也可以用于创建支持 v-model 的自定义输入组件。记住：
```
<input v-model="searchText">
```
等价于：
```
<input
v-bind:value="searchText"
v-on:input="searchText = $event.target.value"
>
```
当用在组件上时，v-model 则会这样：
```
<custom-input
v-bind:value="searchText"
v-on:input="searchText = $event"
></custom-input>
```

为了让它正常工作，这个组件内的 `<input>` 必须：

+ 将其 value 特性绑定到一个名叫 value 的 prop 上
+ 在其 input 事件被触发时，将新的值通过自定义的 input 事件抛出
写成代码之后是这样的：
```
Vue.component('custom-input', {
props: ['value'],
template: `
    <input
    v-bind:value="value"
    v-on:input="$emit('input', $event.target.value)"
    >
`
})
```
现在 v-model 就应该可以在这个组件上完美地工作起来了：
```
<custom-input v-model="searchText"></custom-input>
```
#### slot 插槽分发内容
和 HTML 元素一样，我们经常需要向一个组件传递内容，像这样：
```
<alert-box>
Something bad happened.
</alert-box>
```
幸好，Vue 自定义的 `<slot>` 元素让这变得非常简单：
```
Vue.component('alert-box', {
    template: '
        <div class="demo-alert-box">
        <strong>Error!</strong>
        <slot></slot>
        </div>'
})
```
'Something bad happened.' 会直接插入到组件的 `slot` 标签位置.

#### 动态组件
```
<component v-bind:is="currentTabComponent"></component>
```
在上述示例中， `currentTabComponent` 可以包括
+ 已注册组件的名字，或
+ 一个组件的选项对象