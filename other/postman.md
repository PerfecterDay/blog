# postman
{docsify-updated}

## 脚本编写
1. 断言响应状态码
```
pm.test("Status code is 200", function () {
  pm.response.to.have.status(200);
});
```

2. 校验返回 JSON 字段
```
const jsonData = pm.response.json();

pm.test("Code is 0", function () {
  pm.expect(jsonData.code).to.eql(0);
});

pm.test("Message is success", function () {
  pm.expect(jsonData.message).to.eql("success");
});
```

3. 提取响应值存为环境变量
```
const jsonData = pm.response.json();
pm.environment.set("newId", jsonData.data.id);
```

4. 根据响应动态判断
```
const jsonData = pm.response.json();

if (jsonData.code !== 0) {
  console.warn("接口报错:", jsonData.message);
} else {
  console.log("成功创建:", jsonData.data.id);
}
```

## Mock Server
> https://learning.postman.com/latest-v-12/docs/design-apis/mock-apis/overview

1. 建真实接口请求，在 Collection 里建一个 Request。
2. 给这个 Request 加多个 Example，右键request -> add example
3. 创建 Mock Server， collection 右键（或者点击...） -> more -> mock
4. 用 Header 控制返回哪个 Example， `x-mock-response-name: success-example`

注意点：
1. 当你创建的是 private mock server 时，需要创建一个API key，点击头像-> settings -> API keys
2. 请求时添加请求头 `x-api-key: PMAK-xxxxx..`
3. 当有多个 example 时，需要添加请求头 `x-mock-response-name: success-example` 指定返回的 example 

postman 最左侧有个 Mock Servers 菜单栏，单击菜单栏能看到已经创建的 Mock Servers 服务器。

注意，如果请求模糊， postman mock server 返回的 example 不一定符合预期。 mock server 具体的匹配算法请参考以下链接：
> https://learning.postman.com/docs/design-apis/mock-apis/matching-algorithm/