# Optional
{docsify-updated}
> https://www.baeldung.com/java-optional

- [Optional](#optional)
    - [Optional 创建](#optional-创建)
    - [常用方法](#常用方法)
      - [检查值是否存在：`isPresent()` 和 `isEmpty()`](#检查值是否存在ispresent-和-isempty)
      - [`ifPresent()`](#ifpresent)
      - [`orElse()` 返回默认值](#orelse-返回默认值)
      - [`orElseGet()` 返回默认值](#orelseget-返回默认值)
      - [`orElse()` 和 `orElseGet()` 的区别](#orelse-和-orelseget-的区别)
      - [检索封装值的最终方法是 `get()` 方法。](#检索封装值的最终方法是-get-方法)
      - [用map()转换值](#用map转换值)
      - [用flatMap()转换值](#用flatmap转换值)
      - [map 与 flatmap 区别](#map-与-flatmap-区别)
    - [序列化问题](#序列化问题)
    - [判空示例](#判空示例)


`Optional` 类的目的是提供一个类型级别的解决方案，用于表示可以为`null`的可选的值 。  
为了更深入地了解我们为什么要关心 `Optional` 类，请看一下[Oracle的官方文章](https://www.oracle.com/technical-resources/articles/java/java8-optional.html)。

首先，直观的看一下 `Optional` 的作用：
```
String version = "UNKNOWN";
if(computer != null){
  Soundcard soundcard = computer.getSoundcard();
  if(soundcard != null){
    USB usb = soundcard.getUSB();
    if(usb != null){
      version = usb.getVersion();
    }
  }
}


String version = computer.flatMap(Computer::getSoundcard)
                          .flatMap(Soundcard::getUSB)
                          .map(USB::getVersion)
                          .orElse("UNKNOWN");
```

### Optional 创建
1. 要创建一个空的Optional对象，只需要使用其 `empty()` 静态方法。
```
@Test
public void whenCreatesEmptyOptional_thenCorrect() {
    Optional<String> empty = Optional.empty();
    assertFalse(empty.isPresent());
}
```
注意，我们使用 `isPresent()` 方法来检查 `Optional` 对象中是否有一个值。只有当我们用一个非 `null` 值创建 `Optional` 时，`isPresent()` 才会返回 true。

2. 也可以用静态方法`of(Object o)`创建一个Optional对象。
```
@Test
public void givenNonNull_whenCreatesNonNullable_thenCorrect() {
    String name = "baeldung";
    Optional<String> opt = Optional.of(name);
    assertTrue(opt.isPresent());
}
```
然而，传递给of()方法的参数**不能是null**。否则，我们会得到一个 `NullPointerException` 。

3. 如果我们期望允许从一些空值创建，我们可以使用 `ofNullable(bject o)` 方法。
```
@Test
public void givenNonNull_whenCreatesNullable_thenCorrect() {
    String name = "baeldung";
    Optional<String> opt = Optional.ofNullable(name);
    assertTrue(opt.isPresent());
}
```
通过这样做，如果我们传入一个空引用，它不会抛出一个异常，而是返回一个 `Optional.EMPTY` 对象。

### 常用方法

#### 检查值是否存在：`isPresent()` 和 `isEmpty()`
当我们有一个从方法返回或由我们创建的Optional对象时，我们可以用isPresent()方法检查其中是否有一个值。如果包裹的值不是 null ，该方法返回true。
另外，从Java 11开始，我们可以用 `isEmpty()` 方法做相同的事情。如果 `Optional` 是 `EMPTY` 对象时返回 true。


#### `ifPresent()`
`ifPresent()` 方法使我们能够在发现包裹的值为非空时对其运行一些代码。在 `Optional` 之前，我们会这样做。
```
if(name != null) {
    System.out.println(name.length());
}
```
这段代码先检查name变量是否为空，然后再继续对其执行一些代码。这种方法很冗长，而且这还不是唯一的问题--它还容易出错。  
如果一个 `null` 值进入该一段，这可能会在运行时导致`NullPointerException`。当一个程序由于输入问题而失败时，这往往是不良的编程实践的结果。  
可选的使我们明确地处理可空值，作为执行良好编程实践的一种方式。  
现在我们来看看在Java 8中如何重构上述代码。
```
@Test
public void givenOptional_whenIfPresentWorks_thenCorrect() {
    Optional<String> opt = Optional.of("baeldung");
    opt.ifPresent(name -> System.out.println(name.length()));
}
```

#### `orElse()` 返回默认值
`orElse()` 方法用于检索被包裹在一个Optional实例中的值。它需要一个参数，作为一个默认值。如果存在非 null 的值， orElse()方法返回被包装的值，否则返回其参数。
```
@Test
public void whenOrElseWorks_thenCorrect() {
    String nullName = null;
    String name = Optional.ofNullable(nullName).orElse("john");
    assertEquals("john", name);
}
```

#### `orElseGet()` 返回默认值
orElseGet()方法与orElse()类似。然而，如果Optional值不存在，它不是取一个值来返回，而是取一个 `supplier` 功能接口，该接口被调用并返回调用的返回值。
```
@Test
public void whenOrElseGetWorks_thenCorrect() {
    String nullName = null;
    String name = Optional.ofNullable(nullName).orElseGet(() -> "john");
    assertEquals("john", name);
}
```

#### `orElse()` 和 `orElseGet()` 的区别
```
@Test
public void whenOrElseGetAndOrElseDiffer_thenCorrect() {
    String text = "Text present";

    System.out.println("Using orElseGet:");
    String defaultText 
      = Optional.ofNullable(text).orElseGet(this::getMyDefault);
    assertEquals("Text present", defaultText);

    System.out.println("Using orElse:");
    defaultText = Optional.ofNullable(text).orElse(getMyDefault());
    assertEquals("Text present", defaultText);
}
```

请注意，当使用orElseGet()检索被包装的值时，如果包含的值已经存在，`getMyDefault()` 方法不会被调用。
然而，当使用`orElse()`时，无论包裹的值是否存在，默认对象都被创建。所以在这种情况下，我们只是创建了一个多余的对象，而这个对象永远不会被使用。

在这个简单的例子中，创建一个默认对象并没有什么大的代价，因为JVM知道如何处理这种情况。然而，当像`getMyDefault()`这样的方法需要调用网络服务或甚至查询数据库时，成本就变得非常明显了。


#### 检索封装值的最终方法是 `get()` 方法。
然而，与前三种方法不同的是，get()只能在被包装的对象不是空的情况下返回一个值；否则，它会抛出一个无此元素的异常。  
这就是get()方法的主要缺陷。理想情况下，Optional应该帮助我们避免这种不可预见的异常。因此，这种方法违背了Optional的目标，可能会在未来的版本中被废弃。


#### 用map()转换值
我们可以用map()方法转换Optional值:
```
@Test
public void givenOptional_whenMapWorks_thenCorrect() {
    List<String> companyNames = Arrays.asList(
      "paypal", "oracle", "", "microsoft", "", "apple");
    Optional<List<String>> listOptional = Optional.of(companyNames);

    int size = listOptional
      .map(List::size)
      .orElse(0);
    assertEquals(6, size);
}
```
在这个例子中，我们将一个字符串列表包裹在一个Optional对象中，并使用它的map方法对所包含的列表执行一个动作。我们执行的操作是检索列表的大小。
map方法返回包裹在Optional中的计算结果。然后我们必须在返回的Optional上调用一个适当的方法来检索其值。

请注意，过滤器方法只是对值进行检查，只有当值符合给定的谓词时，才返回一个描述该值的可选项。否则返回`Optional.EMPTY`。而 map 方法会获取现有的值，使用该值执行计算，并将计算结果封装在一个 Optional 对象中返回。

```
@Test
public void givenOptional_whenMapWorksWithFilter_thenCorrect() {
    String password = " password ";
    Optional<String> passOpt = Optional.of(password);
    boolean correctPassword = passOpt.filter(
      pass -> pass.equals("password")).isPresent();
    assertFalse(correctPassword);

    correctPassword = passOpt
      .map(String::trim)
      .filter(pass -> pass.equals("password"))
      .isPresent();
    assertTrue(correctPassword);
}
```

#### 用flatMap()转换值
就像`map()`方法一样，我们也有`flatMap()`方法作为转换值的替代品。**不同的是，`map()`返回的还是一个 Optional 对象，而 `flatMap()`会直接返回特定的类型。**
之前，我们创建了简单的String和Integer对象来包装在一个Optional实例中。然而，我们经常会从一个复杂对象的访问器中接收这些对象。
为了更清楚地了解其中的区别，让我们看看一个Person对象，它接收一个人的详细信息，如姓名、年龄和密码:
```
public class Person {
    private String name;
    private int age;
    private String password;

    public Optional<String> getName() {
        return Optional.ofNullable(name);
    }

    public Optional<Integer> getAge() {
        return Optional.ofNullable(age);
    }

    public Optional<String> getPassword() {
        return Optional.ofNullable(password);
    }

    // normal constructors and setters
}
```

```
@Test
public void givenOptional_whenFlatMapWorks_thenCorrect2() {
    Person person = new Person("john", 26);
    Optional<Person> personOptional = Optional.of(person);

    Optional<Optional<String>> nameOptionalWrapper  
      = personOptional.map(Person::getName);
    Optional<String> nameOptional  
      = nameOptionalWrapper.orElseThrow(IllegalArgumentException::new);
    String name1 = nameOptional.orElse("");
    assertEquals("john", name1);

    String name = personOptional
      .flatMap(Person::getName)
      .orElse("");
    assertEquals("john", name);
}
```
在这里，我们试图检索Person对象的name属性来执行断言。

注意我们是如何在第三条语句中用`map()`方法实现这一目标的，然后注意我们之后是如何用`flatMap()`方法做同样的事情的。

`Person::getName`方法的引用类似于我们在上一节中为清理密码而进行的`String::trim`调用。

唯一的区别是，getName()返回一个Optional，而不是像trim()操作那样返回一个String。这一点，再加上地图转换将结果包裹在一个Optional对象中的事实，导致了一个嵌套的Optional。

因此，在使用map()方法时，我们需要在使用转换后的值之前增加一个额外的调用来检索该值。这样一来，Optional的包装就会被移除。在使用flatMap时，这个操作是隐式执行的。

#### map 与 flatmap 区别
```
public <U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent()) {
        return empty();
    } else {
        return Optional.ofNullable(mapper.apply(value));
    }
}

public <U> Optional<U> flatMap(Function<? super T, ? extends Optional<? extends U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent()) {
        return empty();
    } else {
        @SuppressWarnings("unchecked")
        Optional<U> r = (Optional<U>) mapper.apply(value);
        return Objects.requireNonNull(r);
    }
}
```
可以看出来， `map` 的参数 `Function` 可以返回任意类型对象， `map` 方法内部会使用 `Optional.ofNullable(xxx)` 来包装返回值。  
而 `flatMap` 要求参数 `Function` 返回 `Optional` 对象，自己本身不会再包装返回的对象。

所以如果我们使用 `Function<? super T, ? extends Optional<? extends U>>` 的参数分别调用两个方法时， `map` 就会返回类似 `Optional<Optional<String>>` 的值，而 `flatmap` 会直接返回 `Optional<String>` 类型值。

### 序列化问题
如上所述，Optional是作为一个返回类型使用的。不建议将其作为一个字段类型使用。
此外，在一个可序列化的类中使用Optional会导致NotSerializableException。我们的文章[Java Optional作为返回类型](https://www.baeldung.com/java-optional-return)进一步解决了序列化的问题。
在[Using Optional With Jackson](https://www.baeldung.com/jackson-optional)中，我们解释了当Optional字段被序列化时会发生什么，以及一些变通方法以达到预期效果。


### 判空示例
```
public String getCity(User user)  throws Exception{
        if(user!=null){
            if(user.getAddress()!=null){
                Address address = user.getAddress();
                if(address.getCity()!=null){
                    return address.getCity();
                }
            }
        }
        throw new Excpetion("取值错误"); 
    }

public String getCity(User user) throws Exception{
    return Optional.ofNullable(user)
                   .map(u-> u.getAddress())
                   .flatMap(a->a.getCity())
                   .orElseThrow(()->new Exception("取值错误"));
}
```

