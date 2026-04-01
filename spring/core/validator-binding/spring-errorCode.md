# 解析错误码为错误消息
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/error-code-resolution.html

前面已讲解了数据绑定与验证机制。本节将介绍如何将验证错误码解析为错误消息。若需通过 `MessageSource` 将错误码转化为错误消息，可利用调用 `reject()` 方法时提供的错误码实现。当调用 `Errors` 接口中的 `rejectValue` 或其他 `reject` 方法（无论是直接调用，还是通过 `ValidationUtils` 类等间接调用）时，底层实现不仅会注册传递的错误码参数，还会注册若干额外的错误代码。 `MessageCodesResolver` 决定了 `Errors` 接口注册哪些错误代码。默认使用 `DefaultMessageCodesResolver` ，它不仅会注册 `reject` 时提供的错误代码，还会注册包含传递给拒绝方法的字段名称的消息。因此，调用 `rejectValue("age", "too.darn.old")` 方法，除 `too.darn.old` 代码外，Spring 还会注册 `too.darn.old.age` 和 `too.darn.old.age.int`（前者包含字段名，后者包含字段类型）。此设计旨在方便开发者定位错误消息。