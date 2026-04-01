# Promise
{docsify-updated}


## 回调地狱问题
连续执行两个或者多个异步操作是一个常见的需求，在上一个操作执行成功之后，开始下一个的操作，并带着上一步操作所返回的结果。在旧的回调风格中，这种操作会导致经典的回调地狱：

```js
doSomething(function (result) {
  doSomethingElse(result, function (newResult) {
    doThirdThing(newResult, function (finalResult) {
      console.log(`得到最终结果：${finalResult}`);
    }, failureCallback);
  }, failureCallback);
}, failureCallback);
```

有了 Promise，我们就可以通过一个 Promise 链来解决这个问题。这就是 Promise API 的优势，因为回调函数是附加到返回的 Promise 对象上的，而不是传入一个函数中。 `then()` 函数会返回一个和**原来不同的新的 Promise**:

```js
doSomething()
  .then((result) => doSomethingElse(result))
  .then((newResult) => doThirdThing(newResult))
  .then((finalResult) => {
    console.log(`得到最终结果：${finalResult}`);
  })
  .catch(failureCallback);
```

`doSomethingElse` 和 `doThirdThing` 可以返回任何值——如果它们返回的是 Promise，那么会**首先等待这个 Promise 的敲定，然后下一个回调函数会接收到它的兑现值，而不是 Promise 本身**。在 `then` 回调中始终返回 Promise 是非常重要的，即使 Promise 总是兑现为 undefined。如果上一个处理器启动了一个 Promise 但并没有返回它，那么就没有办法再追踪它的敲定状态了，这个 Promise 就是“漂浮”的。

## Promise 
本质上 `Promise` 是一个函数返回的对象，我们可以在它上面**绑定回调函数**，这样我们就不需要在一开始把回调函数作为参数传入这个函数了。

```js
// 成功的回调函数
function successCallback(result) {
  console.log("音频文件创建成功：" + result);
}

// 失败的回调函数
function failureCallback(error) {
  console.log("音频文件创建失败：" + error);
}

createAudioFileAsync(audioSettings, successCallback, failureCallback);
```

如果重写 createAudioFileAsync() 为返回 Promise 的形式，你可以把回调函数附加到它上面：

```js
createAudioFileAsync(audioSettings).then(successCallback, failureCallback);
```

Promise 有三种状态：
+ 待定（pending）：初始状态，既没有被兑现，也没有被拒绝。这是调用 fetch() 返回 Promise 时的状态，此时请求还在进行中。
+ 已兑现（fulfilled）：意味着操作成功完成。当 Promise 完成时，它的 then() 处理函数被调用。
+ 已拒绝（rejected）：意味着操作失败。当一个 Promise 失败时，它的 catch() 处理函数被调用。

## async 和 await
`async` 关键字为你提供了一种更简单的方法来处理基于异步 `Promise` 的代码。在一个函数的开头添加 `async` ，就可以使其成为一个**异步函数**。
```
async function myFunction() {
  // 这是一个异步函数
}
```
在异步函数中，可以**在调用一个返回 Promise 的函数之前使用 `await` 关键字。这使得代码在该点上等待，直到 Promise 被完成，这时 Promise 的响应被当作返回值，或者被拒绝的响应被作为错误抛出。** 这使你能够编写像同步代码一样的异步函数。
```
async function fetchProducts() {
  try {
    // 在这一行之后，我们的函数将等待 `fetch()` 调用完成
    // 调用 `fetch()` 将返回一个“响应”或抛出一个错误
    const response = await fetch(
      "https://mdn.github.io/learning-area/javascript/apis/fetching-data/can-store/products.json",
    );
    if (!response.ok) {
      throw new Error(`HTTP 请求错误：${response.status}`);
    }
    // 在这一行之后，我们的函数将等待 `response.json()` 的调用完成
    // `response.json()` 调用将返回 JSON 对象或抛出一个错误
    const json = await response.json();
    console.log(json[0].name);
  } catch (error) {
    console.error(`无法获取产品列表：${error}`);
  }
}

fetchProducts();
```
这里我们调用 `await fetch()` ，我们的调用者得到的并不是 `Promise` ，而是一个完整的 `Response` 对象，就好像 `fetch()` 是一个同步函数一样。

**但请注意，这个写法只在异步函数中起作用。异步函数总是返回一个 `Promise`**,  所以你不能做这样的事情：
```
const promise = fetchProducts();
console.log(promise[0].name); // “promise”是一个 Promise 对象，因此这句代码无法正常工作
```

相反，你需要这样做：
```
const promise = fetchProducts();
promise.then((data) => console.log(data[0].name));
```

注意 `await` 只能用在以下两种场景：
+ 在 **`async` 函数**中使用 `await` 
+ ESM 的顶层 await (Top-level await, TLA)。

在 ESM 模块 (ES Module) 中，可以直接写 await，不需要额外包裹在 `async function` 里。
```
// file.mjs
const data = await fetch('https://jsonplaceholder.typicode.com/todos/1')
  .then(r => r.json());
console.log(data);

export {data}
```