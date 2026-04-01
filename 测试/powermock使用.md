# powermock
{docsify-updated}




## Powermock的特长
EasyMock、Mockito无法完成对 final类型的class和 method 的 mock 操作，不能 mock static方法，不能 mock 局部变量，PowerMock正是为了解决这些难题的。

## mock
Powermockito.mock()方法主要是根据 class 参数创建一个对应的 mock 对象， powermock 的创建方式可不像 easymock 等使用 proxy 的方式创建，他是会在你运行的过程中动态的修改 class 字节码文件的形式来创建的。

## do..when..then
我们可以看到，每次当我们想给一个 mock 的对象进行某种行为的预期时，都会使用 do...when...then...这样的语法，其实理解起来非常简单:做什么、在什么时候、然后返回什么。

## verify
当我们测试一个 void 方法的时候，根本没有办法去验证一个 mock 对象所执行后的结果，因此唯一的方法就是检查方法是否被调用， verify 就是用来验证某个方法是否被调用过的。

## @RunWith(PowerMockRunner.class)、@PrepareForTest(xxx.class)
`@RunWith(PowerMockRunner.class)`:使用 PowerMockRunner 运行测试
`@PrepareForTest(xxx.class)`: 加载构造 xxx.class。

## mock局部变量
    public class EmployeeService {
        public int getTotalEmployee() {
            EmployeeDao employeeDao = new EmployeeDao(); 
            return employeeDao.getTotal();
        }
    }
假设要测试上述方法，但是`EmployeeDao`对象并不可用且不是注入到`EmployeeService`的成员变量中的，我们无法mock一个dao然后设置到`EmployeeService`的成员变量上。

但是，使用 Powermock 可以这样做：

    EmployeeDao employeeDao = PowerMockito.mock(EmployeeDao.class);
    PowerMockito.whenNew(EmployeeDao.class).withNoArguments() .thenReturn(employeeDao);
    PowerMockito.when(employeeDao.getTotal()).thenReturn(10);
    Mockito.verify(employeeDao).addEmployee(employee);

其实相当于mock了`EmployeeDao`的构造函数。最后一行的`verify`方法可以验证`addEmployee`方法是否被调用了。

## mock static方法
需要注意的是，当你想 mock static 方法时，你必须使用
`@PrepareForTest(需要测试的类.class)` 注，这样的话，在需要测试类中使用的类的静态方法会使用你mock后的。

    PowerMockito.mockStatic(EmployeeUtils.class);
    PowerMockito.when(EmployeeUtils.getEmployeeCount()).thenReturn(10);
 
    final EmployeeService employeeService = new EmployeeService(); 
    int count = employeeService.getEmployeeCount(); 
    assertEquals(10, count);
假如想要在调用 `Thread.sleep` 方法时，抛出异常：

    // These two lines are tightly bound.
    PowerMockito.doThrow(new InterruptedException()).when(Thread.class);
    Thread.sleep(Matchers.anyLong());

## spy
正常情况下，当我们mock了一个对象时，如果调用mock对象的方法，其实什么都不会做。除非你对其做了when...thenReturn 这样的操作。

当我们采用 spy 的方式 mock 一个对象，然后再调用其中的某个方法，他就会根据真实 class 的所提供的方法来调用。
    
    FileService fileService = PowerMockito.spy(new FileService()); 
    fileService.write("liudehua");
    File file = new File(System.getProperty("user.dir")+"/wangwenjun.txt"); 
    assertTrue(file.exists());

如果是     `FileService fileService = PowerMockito.mock(FileService.class); ` 则不会生成 wangwenjun.txt 文件。

## mock private 方法
假设我们有一个类，其中有一个方法 A 是公有方法，但是他需要依赖一个私有方法 B， 但是其中的方法 B 是一个很难在单元测试中真实模拟，所以我们需要 mock 私有方法的行为。

    EmployeeService employeeService = PowerMockito.spy(new EmployeeService());
    PowerMockito.doNothing().when(employeeService, "checkExist", "wangwenjun");
    boolean result = employeeService.exist("wangwenjun");
    assertTrue(result);
    PowerMockito.verifyPrivate(employeeService).invoke("checkExist","wan gwenjun");

## doReturn/when VS thenReturn/when
1. 对于普通 mock 对象方法，两者没有什么区别，mock的方法调用都不会实际运行
2. 返回值为 void 的方法，用 doReturn
   Allows to choose a static method when stubbing in 
   doThrow()|doAnswer()|doNothing()|doReturn() style
   Example:
      doThrow(new RuntimeException()).when(StaticList.class);
      StaticList.clear();
3. 对于 spy 的对象， doReturn/when 不会执行 mock的方法，而 thenReturn/when 会执行方法。

## mock void 方法
```
doThrow(new RuntimeException()).when(mockKafka).publishMonitoredCompanyData(Mockito.anyList());

doNothing().when(mockKafka).publishMonitoredCompanyData(Mockito.anyList());
```
