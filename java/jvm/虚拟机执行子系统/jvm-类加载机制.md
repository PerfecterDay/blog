# java基础-类加载机制
{docsify-updated}

Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称作虚拟机的类加载机制。与那些在编译时需要进行连接的语言不同，在Java语言里面，类型的加载、连接和初始化过程都是在程序运行期间完成的，这种策略让Java语言进行提前编译会面临额外的困难，也会让类加载时稍微增加一些性能开销，但是却为Java应用提供了极高的扩展性和灵活性，Java天生可以动态扩展的语言特性就是依赖运行期动态加载和动态连接这个特点实现的。

**对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。**

每个 Class 对象的内部都有一个 classLoader 字段来标识自己是由哪个 ClassLoader 加载的。ClassLoader 就像一个容器，里面装了很多已经加载的 Class 对象。

```java
class Class<T> {
  ...
  private final ClassLoader classLoader;
  ...
}
```
ClassLoader 相当于类的命名空间，起到了类隔离的作用。位于同一个 ClassLoader 里面的类名是唯一的，不同的 ClassLoader 可以持有同名的类。ClassLoader 是类名称的容器，是类的沙箱。

这里所指的“相等”，包括代表类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括了使用instanceof关键字做对象所属关系判定等各种情况。

```
public class ClassLoaderTest {

    public static void main(String[] args) throws Exception {

        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1)+".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };

        Object obj = myLoader.loadClass("org.fenixsoft.classloading.ClassLoaderTest").newInstance();

        System.out.println(obj.getClass());
        System.out.println(obj instanceof org.fenixsoft.classloading.ClassLoaderTest);
    }
}
```

### Java语言系统自带的三个类加载器
<center><img src="pics/classloader.png" alt="" height=500px></center>

+ `Bootstrap ClassLoader` :引导类加载器，是虚拟机不可分割的一部分，通常由C/C++语言来实现。主要加载核心类库，`%JRE_HOME%\jre\lib`(1.8) 下的 `rt.jar` 、`resources.jar` 、`charsets.jar` 和其它类。另外需要注意的是可以通过启动 JVM 时指定`-Xbootclasspath` 和路径来改变 `Bootstrap ClassLoader` 的加载目录。比如 `java -Xbootclasspath/a:path`， `path` 被追加到 `Bootstrap ClassLoader` 默认的加载路径中。
+ `Extention ClassLoader(ExtClassLoader)`: 扩展类加载器，加载目录 `%JRE_HOME%\jre\lib\ext`(1.8) 目录下的 jar 包和 class 文件。还可以加载 `-Djava.ext.dirs` 选项指定目录下的 jar 和类。
+ `System ClassLoader(AppClassLoader)`: 系统类加载器，加载当前应用的 `classpath` 的所有类。

**每个线程都有一个对类加载器的引用，称为上下文类加载器。主线程的上下文类加载器是系统类加载器。当创建新线程是，它的上下文类加载器会被设置为创建该线程的上下文类加载器。因此，如果不做任何特殊操作，那么所有的线程都会将它的上下文类加载器设置为系统类加载器。**

### 源码分析
`Bootstrap ClassLoader` 是由 C++ 编写的本地代码，由 JDK 的 native 方法调用。
`Extention ClassLoader` 和 `System ClassLoader` 都是 `sun.misc.Launcher` 的静态内部类。

#### Bootstrap ClassLoader
`sun.misc.Launcher` 下的 `bootClassPath` 是 `Bootstrap ClassLoader`的加载路径。

    private static String bootClassPath = System.getProperty("sun.boot.class.path");

JDK 1.8 下输出的以下路径：
```
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/resources.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/rt.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/sunrsasign.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jsse.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jce.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/charsets.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jfr.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/classes
```

#### ExtClassLoader
<details>
    <summary>ExtClassLoader</summary>

        static class ExtClassLoader extends URLClassLoader {
            private static volatile Launcher.ExtClassLoader instance;

            public static Launcher.ExtClassLoader getExtClassLoader() throws IOException {
                if (instance == null) {
                    Class var0 = Launcher.ExtClassLoader.class;
                    synchronized(Launcher.ExtClassLoader.class) {
                        if (instance == null) {
                            instance = createExtClassLoader();
                        }
                    }
                }
        
                return instance;
            }
        
            private static Launcher.ExtClassLoader createExtClassLoader() throws IOException {
                try {
                    return (Launcher.ExtClassLoader)AccessController.doPrivileged(new PrivilegedExceptionAction<Launcher.ExtClassLoader>() {
                        public Launcher.ExtClassLoader run() throws IOException {
                            File[] var1 = Launcher.ExtClassLoader.getExtDirs();
                            int var2 = var1.length;
        
                            for(int var3 = 0; var3 < var2; ++var3) {
                                MetaIndex.registerDirectory(var1[var3]);
                            }
        
                            return new Launcher.ExtClassLoader(var1);
                        }
                    });
                } catch (PrivilegedActionException var1) {
                    throw (IOException)var1.getException();
                }
            }
        
            private static File[] getExtDirs() {
                String var0 = System.getProperty("java.ext.dirs");
                File[] var1;
                if (var0 != null) {
                    StringTokenizer var2 = new StringTokenizer(var0, File.pathSeparator);
                    int var3 = var2.countTokens();
                    var1 = new File[var3];
        
                    for(int var4 = 0; var4 < var3; ++var4) {
                        var1[var4] = new File(var2.nextToken());
                    }
                } else {
                    var1 = new File[0];
                }
        
                return var1;
            }
        
            private static URL[] getExtURLs(File[] var0) throws IOException {
                Vector var1 = new Vector();
        
                for(int var2 = 0; var2 < var0.length; ++var2) {
                    String[] var3 = var0[var2].list();
                    if (var3 != null) {
                        for(int var4 = 0; var4 < var3.length; ++var4) {
                            if (!var3[var4].equals("meta-index")) {
                                File var5 = new File(var0[var2], var3[var4]);
                                var1.add(Launcher.getFileURL(var5));
                            }
                        }
                    }
                }
        
                URL[] var6 = new URL[var1.size()];
                var1.copyInto(var6);
                return var6;
            }
        
            public String findLibrary(String var1) {
                var1 = System.mapLibraryName(var1);
                URL[] var2 = super.getURLs();
                File var3 = null;
        
                for(int var4 = 0; var4 < var2.length; ++var4) {
                    URI var5;
                    try {
                        var5 = var2[var4].toURI();
                    } catch (URISyntaxException var9) {
                        continue;
                    }
        
                    File var6 = Paths.get(var5).toFile().getParentFile();
                    if (var6 != null && !var6.equals(var3)) {
                        String var7 = VM.getSavedProperty("os.arch");
                        File var8;
                        if (var7 != null) {
                            var8 = new File(new File(var6, var7), var1);
                            if (var8.exists()) {
                                return var8.getAbsolutePath();
                            }
                        }
        
                        var8 = new File(var6, var1);
                        if (var8.exists()) {
                            return var8.getAbsolutePath();
                        }
                    }
        
                    var3 = var6;
                }
        
                return null;
            }
    }
</details>

`ExtClassLoader` 加载的是 `System.getProperty("java.ext.dirs")` 路径下的类。

```
/Users/zhongzwang/Library/Java/Extensions
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext
/Library/Java/Extensions
/Network/Library/Java/Extensions
/System/Library/Java/Extensions
/usr/lib/java
```

#### AppClassLoader
```
static class AppClassLoader extends URLClassLoader {
        final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);

        public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
            final String var1 = System.getProperty("java.class.path");
            final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
            return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<Launcher.AppClassLoader>() {
                public Launcher.AppClassLoader run() {
                    URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
                    return new Launcher.AppClassLoader(var1x, var0);
                }
            });
        }

        AppClassLoader(URL[] var1, ClassLoader var2) {
            super(var1, var2, Launcher.factory);
            this.ucp.initLookupCache(this);
        }

        public Class<?> loadClass(String var1, boolean var2) throws ClassNotFoundException {
            int var3 = var1.lastIndexOf(46);
            if (var3 != -1) {
                SecurityManager var4 = System.getSecurityManager();
                if (var4 != null) {
                    var4.checkPackageAccess(var1.substring(0, var3));
                }
            }

            if (this.ucp.knownToNotExist(var1)) {
                Class var5 = this.findLoadedClass(var1);
                if (var5 != null) {
                    if (var2) {
                        this.resolveClass(var5);
                    }

                    return var5;
                } else {
                    throw new ClassNotFoundException(var1);
                }
            } else {
                return super.loadClass(var1, var2);
            }
        }
```
从源码中可以看出来： 应用类加载器加载的是 `System.getProperty("java.class.path")` 路径下的类和jar。其实就是 classpath 路径。

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/charsets.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/deploy.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/cldrdata.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/dnsns.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/jaccess.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/jfxrt.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/localedata.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/nashorn.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/sunec.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/ext/zipfs.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/javaws.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jce.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jfr.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jfxswt.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/jsse.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/management-agent.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/plugin.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/resources.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/jre/lib/rt.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/ant-javafx.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/dt.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/javafx-mx.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/jconsole.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/packager.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/sa-jdi.jar
/Library/Java/JavaVirtualMachines/jdk1.8.0_171.jdk/Contents/Home/lib/tools.jar
/Users/zhongzwang/worksapce/My/JavaBase/sort/out/production/MasetrcardScript
/Applications/IntelliJ IDEA CE.app/Contents/lib/idea_rt.jar
```

### 类加载顺序（双亲委派模型）
类加载器有一种父子关系，除了引导类加载器外，每个类加载器都有一个父类加载器:
`SystemClassLoader` 的父加载器是 `ExtClassLoader` , `ExtClassLoader` 的父加载器是 `Bootstrap ClassLoader`。 自定义的父加载器一般是 `SystemClassLoader`.

加载类的步骤一般是：
1. 一个 `SystemClassLoader` 加载类时，它会委托给父类加载器 `ExtClassLoader`。
2. `ExtClassLoader` 则首先会委托 `Bootstrap ClassLoader` 加载类，它首先查找缓存，如果没有找到的话，就去找自己的规定的路径下，也就是 `sun.mic.boot.class` 下面的路径。找到就返回，没有找到，让子加载器自己去找。
3. `Bootstrap ClassLoader` 如果没有查找成功，则 `ExtClassLoader` 自己在 `java.ext.dirs` 路径中去查找，查找成功就返回，查找不成功，再向下让子加载器找。
4. `ExtClassLoader` 查找不成功， `SystemClassLoader` 就自己查找，在 `java.class.path` 路径下查找。找到就返回。如果没有找到就让子类找，如果没有子类会怎么样？抛出各种异常。

### 重要方法
#### loadClass()
上面是方法原型，一般实现这个方法的步骤是

1. 执行findLoadedClass(String)去检测这个class是不是已经加载过了。
2. 执行父加载器的loadClass方法。如果父加载器为null，则jvm内置的加载器去替代，也就是Bootstrap ClassLoader。这也解释了ExtClassLoader的parent为null,但仍然说Bootstrap ClassLoader是它的父加载器。
3. 如果向上委托父加载器没有加载成功，则通过findClass(String)查找。
4. 如果class在上面的步骤中找到了，参数resolve又是true的话，那么loadClass()又会调用resolveClass(Class)这个方法来生成最终的Class对象。 我们可以从源代码看出这个步骤。

```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检测是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                    	//父加载器不为空则调用父加载器的loadClass
                        c = parent.loadClass(name, false);
                    } else {
                    	//父加载器为空则调用Bootstrap Classloader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    //父加载器没有找到，则调用findclass
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
            	//调用resolveClass()
                resolveClass(c);
            }
            return c;
        }
    }
```
另外，要注意的是如果要编写一个classLoader的子类，也就是自定义一个classloader，建议覆盖findClass()方法，而不要直接改写loadClass()方法。
另外
```
if (parent != null) {
	//父加载器不为空则调用父加载器的loadClass
    c = parent.loadClass(name, false);
} else {
	//父加载器为空则调用Bootstrap Classloader
    c = findBootstrapClassOrNull(name);
}
```

### 自定义ClassLoader
不知道大家有没有发现，不管 `Bootstrap ClassLoader` 还是 `ExtClassLoader` 等，这些类加载器都只是加载指定的目录下的jar包或者资源。如果在某种情况下，我们需要动态加载一些东西呢？比如从D盘某个文件夹加载一个class文件，或者从网络上下载class主内容然后再进行加载，这样可以吗？

如果要这样做的话，需要我们自定义一个classloader。

#### 自定义步骤
1. 编写一个类继承自ClassLoader抽象类。
2. 复写它的findClass()方法,为来自本地文件系统或者其他来源的类加载其字节码，实际上就是通过各种方式处理获得一个 byte[] 数组。
3. 在findClass()方法中调用defineClass()。

##### defineClass()
这个方法在编写自定义classloader的时候非常重要，它能将class二进制内容转换成Class对象，如果不符合要求的会抛出各种异常。

**注意点：一个ClassLoader创建时如果没有指定parent，那么它的parent默认就是AppClassLoader。**

上面说的是，如果自定义一个ClassLoader，默认的parent父加载器是AppClassLoader，因为这样就能够保证它能访问系统内置加载器加载成功的class文件。

### 字节码校验
当类加载器将新家在的字节码传递给虚拟机时，这些字节码首先要接收校验器的校验。校验器负责检查那些指令无法执行的明显有破坏性的操作。除了系统类外，所有的类都要被校验。下面是校验器执行的一些检查：
+ 变量在使用之前是否初始化
+ 方法调用与对象引用类型之间要匹配
+ 访问私有数据和方法的规则没有被破坏
+ 对本地变量的访问都落在运行时堆栈内
+ 运行时堆栈没有溢出

上述任何一条检查没有通过，类就不会被加载。
为什么要有一个专门的校验器来检查这些呢？毕竟编译器自身也会进行这些检查？实际上，如果正常用 java 编译器编译出来的字节码总是可以通过这些检查的。但是，除了编译器之外，也可以使用二进制编辑的方式按照字节码规范生成字节码，恶意程序很可能这样做，对于这种字节码文件，校验器就能很好的防范，阻止这些类的加载运行。

## 线程与加载器
```java
class Thread {
  ...
  private ClassLoader contextClassLoader;

  public ClassLoader getContextClassLoader() {
    return contextClassLoader;
  }

  public void setContextClassLoader(ClassLoader cl) {
    this.contextClassLoader = cl;
  }
  ...
}
```
其次线程的 contextClassLoader 是从父线程那里继承过来的，所谓父线程就是创建了当前线程的线程。程序启动时的 main 线程的 contextClassLoader 就是 AppClassLoader。这意味着如果没有人工去设置，那么所有的线程的 contextClassLoader 都是 AppClassLoader。

每个类都将使用自己的类加载器加载其他类。因此，如果 ClassA.class 引用 ClassB.class，那么 ClassB 必须位于 ClassA 或其父类的类加载器的类路径上。

线程上下文类加载器是当前线程的当前类加载器。对象可以从 ClassLoaderC 中的类创建，然后传递给 ClassLoaderD 所拥有的线程。在这种情况下，如果对象想加载自身类加载器中没有的资源，就需要直接使用 `Thread.currentThread().getContextClassLoader()` 来加载。

## 加载器问题实例

使用了 `CompletableFuture` 来执行一个异步任务，异步任务中使用了 jaxb xml 相关的包来解析 xml：
```java

CompletableFuture.supplyAsync(() -> {doSomeThing()});
doSomeThing(){
    JAXBContext context = JAXBContext.newInstance(t.getClass());
}
```
线上报错异常栈如下：
```
javax.xml.bind.JAXBException: Implementation of JAXB-API has not been found on module path or classpath.
	at javax.xml.bind.ContextFinder.newInstance(ContextFinder.java:278)
	at javax.xml.bind.ContextFinder.find(ContextFinder.java:421)
	at javax.xml.bind.JAXBContext.newInstance(JAXBContext.java:721)
	at javax.xml.bind.JAXBContext.newInstance(JAXBContext.java:662)
	at com.demo.demo.utils.XmlUtils.beanToXml(XmlUtils.java:39)
	at com.demo.demo.service.impl.CmsSendBoServiceImpl.sendBoCms(CmsSendBoServiceImpl.java:30)
	at com.demo.demo.business.login.TradeLoginServiceV2.getCmsAccountInfo(TradeLoginServiceV2.java:144)
	at com.demo.demo.business.login.TradeLoginServiceV2.findUserTradeAcctType(TradeLoginServiceV2.java:179)
	at com.demo.demo.business.UserService.lambda$getUserInfoV2$1(UserService.java:117)
	at java.base/java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java:1700)
	at java.base/java.util.concurrent.CompletableFuture$AsyncSupply.exec(CompletableFuture.java:1692)
	at java.base/java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:290)
	at java.base/java.util.concurrent.ForkJoinPool$WorkQueue.topLevelExec(ForkJoinPool.java:1020)
	at java.base/java.util.concurrent.ForkJoinPool.scan(ForkJoinPool.java:1656)
	at java.base/java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1594)
	at java.base/java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:177)
Caused by: java.lang.ClassNotFoundException: com.sun.xml.internal.bind.v2.ContextFactory
	at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:582)
	at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178)
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:521)
	at javax.xml.bind.ServiceLoaderUtil.nullSafeLoadClass(ServiceLoaderUtil.java:122)
	at javax.xml.bind.ServiceLoaderUtil.safeLoadClass(ServiceLoaderUtil.java:155)
	at javax.xml.bind.ContextFinder.newInstance(ContextFinder.java:276)
	... 15 common frames omitted"
```
从中可以看出来是类加载不到错误： `Caused by: java.lang.ClassNotFoundException: com.sun.xml.internal.bind.v2.ContextFactory`

跟踪了一些代码发现：jaxb-xml 在内部使用了很多 SPI 机制：
```
@Deprecated
static String firstByServiceLoaderDeprecated(Class spiClass,
                                                ClassLoader classLoader) throws JAXBException {

    final String jaxbContextFQCN = spiClass.getName();

    logger.fine("Searching META-INF/services");

    // search META-INF services next
    BufferedReader r = null;
    final String resource = "META-INF/services/" + jaxbContextFQCN;
    final InputStream resourceStream =
                (classLoader == null) ?
                        ClassLoader.getSystemResourceAsStream(resource) :
                        classLoader.getResourceAsStream(resource);
    if (resourceStream != null) {
        r = new BufferedReader(new InputStreamReader(resourceStream, "UTF-8"));
        String factoryClassName = r.readLine();
        if (factoryClassName != null) {
            factoryClassName = factoryClassName.trim();
        }
        r.close();
        logger.log(Level.FINE, "Configured factorty class:{0}", factoryClassName);
        return factoryClassName;
    } 
}
```
尽管在 pom 中依赖了 jaxb-runtime:2.3.6 包，也没有加载到。正常情况下应该是SPI去加载 `META-INF/services/javax.xml.bind.JAXBContext` 下的服务。
<center><img src="pics/classloader-issue1.png" width="40%"></center>

加了日志，打印请求处理线程和 `CompletableFuture` 线程中的 Classloader 信息如下：

```text
10.4.152.170
iZj6c5i921jg0zpxwdm52oZ
/home/logs/user-cent...e.log
content: 2024-05-15 11:51:56.543 [http-nio-8913-exec-9] INFO  com.demo.demo.business.UserService  b761d28c-fb26-4e8b-a9cd-aab4c3b56fc6 - Classloader:TomcatEmbeddedWebappClassLoader
  context: user
  delegate: true
----------> Parent Classloader:
org.springframework.boot.loader.LaunchedURLClassLoader@1a86f2f1


2024-05-15 11:51:56.543 [ForkJoinPool.commonPool-worker-19] INFO  com.demo.demo.business.UserService   - Classloader:jdk.internal.loader.ClassLoaders$AppClassLoader@1affbebc
```

确实是使用了不同的 Classloader ，因为我们使用了 `CompletableFuture` ，并且没有使用自定义的线程池，所以 `CompletableFuture` 会使用默认的 fork-join pool 去执行任务，并且 jaxb 会使用线程的 Classloader 去加载类：`Thread.currentThread().getContextClassLoader();`

网上有相似的问题：
+ https://juejin.cn/post/6909445190642040846  
+ https://blog.csdn.net/qq_40836488/article/details/126367111
+ https://blog.51cto.com/u_16608712/10069927