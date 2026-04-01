# JNI 和 
{docsify-updated}

## 示例Hello World
1. 编写一个 java 类，里边写上 native 方法
```
public class HelloJNI {

    static {
        // 加载本地库 libhello.so / libhello.dylib
        System.loadLibrary("hello");
    }

    // 声明 native 方法
    public static native void sayHello();

    public static void main(String[] args) {
        sayHello();
    }
}
```

2. 编译头文件
```
javac HelloJNI.java
javac -h . HelloJNI.java
```

3. 使用C/C++ 实现
```
#include <jni.h>
#include <stdio.h>
#include "HelloJNI.h"

JNIEXPORT void JNICALL
Java_HelloJNI_sayHello(JNIEnv *env, jclass cls) {
    printf("Hello World from JNI!\n");
    fflush(stdout);
}
```
函数名规则： `Java_包名_类名_方法名`，比如 `Java_com_gtja_gjyw_jni_HelloJNI_sayHello`

4. 编译
```
gcc -shared -fPIC \
  -I"$JAVA_HOME/include" \
  -I"$JAVA_HOME/include/darwin" \
  hello.c \
  -o libhello.dylib
```

5. 执行
```
java -Djava.library.path=. HelloJNI

输出：
➜  Downloads java HelloJNI                      
Hello World from JNI!
```

## 区块链项目中遇到的问题
chainMaker 的demo中 `ChainmakerX509CryptoSuite` 中会加载
```
Class clazz = Class.forName("io.netty.handler.ssl.OpenSsl");
```

OpenSsl 会依赖原生的叫 tcnative 的东西，实例化 OpenSsl 时会尝试加载这些本地库，这些库的名字和运行时（JDK版本）是相关的，比如17是下边这些名字：
```
netty_tcnative_osx_aarch_64
netty_tcnative_aarch_64
netty_tcnative
```

1.8 是：
```
netty_tcnative_osx_x86_64
netty_tcnative_x86_64
netty_tcnative
```

如果强行通过参数修改运行时的配置：
```
-Dos.name=osx
-Dos.arch=x86_64
```

可以达到让类加载器加载到路径下的 `libnetty_tcnative_osx_x86_64.jnilib`，但是这个jnilib 文件是和运行时相关的，所以 17 会报不兼容错误：
```
'/private/var/folders/jn/2_tytdt13111lvxr205tjg840000gn/T/libnetty_tcnative_osx_x86_64223919015082559059.dylib' (mach-o file, but is an incompatible architecture (have 'x86_64', need 'arm64e' or 'arm64')), '/System/Volumes/Preboot/Cryptexes/OS/private/var/folders/jn/2_tytdt13111lvxr205tjg840000gn/T/libnetty_tcnative_osx_x86_64223919015082559059.dylib' (no such file)
```
 
如果不加VM参数， JDK17正常加载的是 ：`libnetty_tcnative_osx_aarch_64.jnilib` 名字的文件。

解决方案，找到与平台（mac M1）/JDK版本/ netty 版本对应的依赖：
```<dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-tcnative-boringssl-static</artifactId>
        <version>2.0.53.Final</version>
        <classifier>osx-aarch_64</classifier>
    </dependency>
```

如果版本不对，会报 `java.lang.LinkageError: Possible multiple incompatible native libraries on the classpath` 错误，提示方法缺失或不匹配


## panama
> https://openjdk.org/projects/panama/

1. 新建 `hello.c` 文件，内容如下：
```
#include <stdio.h>

void hello() {
    printf("Hello World from Panama!");
}
```

2. 编译C文件
```
gcc -shared -fPIC hello.c -o libhello.dylib  //macos
gcc -shared -fPIC hello.c -o libhello.so // linux
```

3. 编写java 文件
```
import java.lang.foreign.*;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.nio.file.Path;

import static java.lang.foreign.ValueLayout.*;

public class HelloPanama {

    public static void main(String[] args) throws Throwable {

        // 1. 加载 native library
        System.load(Path.of("./libhello.dylib").toAbsolutePath().toString());
        // Linux 用 libhello.so

        // 2. 获取链接器
        Linker linker = Linker.nativeLinker();

        // 3. 查找 native 符号
        SymbolLookup lookup = SymbolLookup.loaderLookup();
        MemorySegment helloFunc = lookup.find("hello").orElseThrow();

        // 4. 创建函数描述（void hello()）
        FunctionDescriptor fd = FunctionDescriptor.ofVoid();

        // 5. 生成 MethodHandle
        MethodHandle hello =
                linker.downcallHandle(
                        helloFunc,
                        fd
                );

        // 6. 调用
        hello.invokeExact();
    }
}
```

4. 直接编译运行
```
javac HelloPanama.java
java HelloPanama

输出：
Hello World from Panama!
```

## 两者的对比
JNI 的底层逻辑：
```
Java compiled code
↓
call jni_stub
↓
jni_stub 机器码
  - 保存寄存器
  - 建立 JNI frame
  - 修改线程状态
  - Safepoint 检查
↓
call native_function
↓
native_function
↓
返回 jni_stub
  - 恢复线程状态
  - 异常检查
↓
ret 回 Java
```

Panama 的底层逻辑：
```
Java compiled code
↓
直接 call native_function
↓
native_function
↓
ret
```

JNI是运行时动态加载动态库，调用 stub/ABI、传递本地方法参数等都是运行时做的，所以会有很大的开销。  
但是Panama 能在（JIT）编译期就生成这些调用代码。注意这里的编译期是 JIT 编译器而不是 javac 编译器，所以 Panama 在 javac 编译时也不需要准备好 `.so/.dylib` 文件。

为什么 Panama 能做到“直接 call”？
JIT 在**编译**期就知道：
+ 参数类型
+ 返回类型
+ ABI 规则
+ 内存布局

JIT 能直接生成符合 ABI 的机器码