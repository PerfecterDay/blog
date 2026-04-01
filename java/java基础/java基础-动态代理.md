# 动态代理
{docsify-updated}

## Proxy
    public class Proxy implements java.io.Serializable{
        public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h){
                                            ......
                                          }
    }

`Proxy` 类是java反射工具包，主要功能是生成动态代理,通常使用`Proxy.newProxyInstance(...)`方法创建 `Proxy` 对象。
+ 第一个参数是类加载器对象
+ 第二个参数是一个 `Class[]` 数组，表示代理对象可以代理**多个接口**中的多个方法，代理对象实际上在内部实现了接口数组中的所有方法，可以看成是接口数组中所有接口的子类型，所以代理对象可以被转换成接口数组中的每个接口类型，并调用接口中的方法。
+ 第三个参数是 `InvocationHandler` 对象，当代理对象被当成某个被代理接口的实现对象，并调用被代理接口中的方法时，将会调用 `InvocationHandler` 中的 `invoke` 方法。并把自身对象的引用作为第一个参数传递给它，把被调用的方法对象作为第二个参数，参数以对象数组的形式作为第三个参数。因此在 `invoke` 方法中可以完全地控制是否及何时何处调用原来的方法。

`Proxy` 的底层工作原理是：
jdk为我们的生成了一个叫 `$Proxy0` （这个名字后面的0是编号，有多个代理类会依次递增）的代理类，这个类文件放在内存中的，我们在创建代理对象时，就是通过反射获得这个类的构造方法，然后创建的代理实例。

## InvocationHandler接口
```
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```
`InvocationHandler` 只有一个 `invoke` 方法，有三个参数，proxy是通过 `Proxy` 对象生成的动态代理对象， `method` 是代理接口的方法对象， `args` 数组代表参数。
用动态代理的时候一般会实现 `InvocationHandler` 接口，在 `invoke` 方法中实现通用的代理功能。所有使用了这个 `InvocationHandler` 对象的代理对象（上面生成的Proxy对象），在调用代理接口的方法时，都会执行 `invoke` 中的代理逻辑。

所以即使没有代理接口的真正实现类，也可以通过代理对象和 `InvocationHandler` 实现代理逻辑，此时，没有实现被代理对象的逻辑功能。所以，即使我们只声明一个接口，也可以通过动态代理，为其实现很高端的功能。 Mybatis 就是如此，只要声明数据库访问接口，通过动态代理功能就能实现数据库操作。基本思想是通过接口名和方法名，在 xml 文件中找到对应id的sql语句，并执行。

如果有真正的被代理对象，就是代理接口的实现类对象，则一般会在 `InvocationHandler` 中，保存一个代理接口实现类对象的引用，并在 `invoke` 方法中，通过调用 `Method` 类的 `invoke` 方法，第一个参数设为被代理对象，表示调用被代理对象（真实对象）的实现方法。

## 动态代理原理
JDK的 `sun.misc.ProxyGenerator` 可以生成动态的代理对象，这个代理类在内部，拥有所代理的接口的所有方法对象(通过反射获得),并且这个代理类会实现所有代理接口的所有方法，并在各个实现方法内部，调用 `InvocationHandler` 的 `invoke` 方法，只不过不同实现方法中传递的方法对象和参数不同。通过代理对象调用接口方法，就是调用其内部实现的各个方法。

流程如下：

    代理类对象.接口方法()---->代理类对象内部实现的接口方法()----->InvocationHandler.invoke()方法。
    
    interface A{
      void say();
    }
    
    interface B{
      void run();
    }
    
    Object o = Proxy.newProxyInstance(A.class.getClassLoader(),new Class[]{A.class,B.class},new Handler());
    
    (A) o.say();
    (B) o.run();

实际上动态代理生成的类类似于下面这个类：
    
    class Proxy0 implements A,B{
        InvocationHandler h;
        Method say; //通过反射会初始化为A的say方法对象
        Method run; //通过反射会初始化为B的run方法对象
    
       public void say(){
          h.invoke(this,m1,null);
       }
    
       public void run(){
          h.invoke(this,m1,null);
       }
    }
可以通过 `ProxyGenerator` 生成二进制文件，然后反编译出来验证结果，大体思想原理是这样的。