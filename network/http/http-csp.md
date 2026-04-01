# Content-Security-Policy
{docsify-updated}

> https://csp-evaluator.withgoogle.com/


配置内容安全策略涉及到添加 `Content-Security-Policy` HTTP 标头到一个页面，并配置相应的值，以控制用户代理（浏览器等）可以为该页面获取哪些资源。比如一个可以上传文件和显示图片页面，应该允许图片来自任何地方，但限制表单的 action 属性只可以赋值为指定的端点。一个经过恰当设计的内容安全策略应该可以有效的保护页面免受跨站脚本攻击。

<center><img src="pics/csp.png" alt=""></center>