# Validation, Data Binding, and Type Conversion
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation.html

将验证视为业务逻辑存在利弊两面，而Spring提供的验证与数据绑定设计并未排斥任何一方。具体而言，验证机制不应与Web层绑定，应便于本地化处理，且需支持任意可用验证器的插拔式集成。基于这些考量，Spring提供了一种基础且高度实用的Validator契约，适用于应用程序的每个层级。

数据绑定能将用户输入动态绑定到应用程序的领域模型（或用于处理用户输入的任何对象），具有重要作用。Spring 提供了恰如其名的 `DataBinder` 组件来实现这一功能。 `Validator` 与 `DataBinder` 共同构成了验证包，该包主要用于 Web 层但不限于此。

`BeanWrapper` 是 Spring 框架中的基础概念，在许多场景中都有应用。不过，您通常无需直接使用 `BeanWrapper` 。但鉴于这是参考文档，我们认为有必要进行说明。本章将阐述 `BeanWrapper` 的原理，因为若您需要使用它，最可能是在尝试将数据绑定到对象时。
 
Spring的 `DataBinder` 和底层的 `BeanWrapper` 都使用 `PropertyEditorSupport` 实现来解析和格式化属性值。 `PropertyEditor` 和 `PropertyEditorSupport` 类型属于 JavaBeans 规范的一部分，本章也将对此进行说明。Spring 的 `core.convert` 包提供了通用类型转换功能，以及用于格式化 UI 字段值的高级格式化包。这些包可作为 `PropertyEditorSupport` 实现的简化替代方案，本章也将对此进行讨论。

Spring通过基础架构设置和适配器支持Java Bean验证，该适配器对接Spring自有的 `Validator` 接口。应用程序可参照《Java Bean验证》所述方法全局启用Bean验证功能，并将其作为所有验证需求的唯一解决方案。在Web层，应用程序还可参照《配置DataBinder》所述方法，为每个 `DataBinder` 注册控制器级别的Spring `Validator` 实例，这有助于集成自定义验证逻辑。