# 注解
{docsify-updated}

## 元注解
1. `@Documented` —— 指明拥有这个注解的元素可以被javadoc此类的工具文档化。
2. `@Target` —— 指明该类型的注解可以注解的程序元素的范围。该元注解的取值可以为 `TYPE,METHOD,CONSTRUCTOR,FIELD` 等。如果 `Target` 元注解没有出现，那么定义的注解可以应用于程序的任何元素。
3. `@Inherited` —— 指明该注解类型被自动继承。如果用户在当前类中查询这个元注解类型并且当前类的声明中不包含这个元注解类型，那么也将自动查询当前类的父类是否存在 `Inherited` 元注解，这个动作将被重复执行知道这个标注类型被找到，或者是查询到顶层的父类。
4. `@Retention` ——指明了该 `Annotation` 被保留的时间长短。 `RetentionPolicy` 取值为 `SOURCE,CLASS,RUNTIME` 。

## 自定义注解语法
创建自定义注解和创建一个接口相似，但是注解需要以 `@interface` 关键字符号开头。我们可以为注解声明方法。
```
语法：
modifiers @interface AnnotationName{
	type elementName1(); 
	type elementName2() default value; 
	...
}

示例：
@Documented
@Target(ElementType.METHOD)
@Inherited
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodInfo{
	String author() default 'Pankaj';
	String date();
	int revision() default 1;
	String comments();
}
```
+ 注解方法不能带有参数；
+ 注解方法返回值类型限定为： 基本类型 、 `String` 、 `Enums` 、 `Class` 、 `Annotation` 或者是这些类型的数组（多维数组-数组组成的数组不行）；
+ 注解方法可以有默认值；
+ 注解本身能够包含元注解，元注解被用来注解其它注解。

### 注解使用
```
@AnnotationName(elementName1=value1,elementName2=value2)
```

注解可以用在：包、类、接口、方法、构造器、实例域、局部变量、参数变量、类型参数。  
**对包/局部变量的注解只能在源码级别处理。编译后的class文件并不描述局部变量，因此所有的局部变量注解在编译完成后都会被遗弃掉。**

## Java运行时注解解析

使用反射技术来解析java类的注解:
<center><img src="pics/annotation-process.jpg" width="50%"></center>

使用反射处理注解的RetentionPolicy应该设置为RUNTIME，否则java类的注解信息在执行过程中将不可用，那么我们也不能从中得到任何和注解有关的数据。

    public class AnnotationParsing {
     
    public static void main(String[] args) {
        try {
        for (Method method : AnnotationParsing.class
            .getClassLoader()
            .loadClass(('com.journaldev.annotations.AnnotationExample'))
            .getMethods()) {
            // checks if MethodInfo annotation is present for the method
            if (method.isAnnotationPresent(com.journaldev.annotations.MethodInfo.class)) {
                try {
            // iterates all the annotations available in the method
                    for (Annotation anno : method.getDeclaredAnnotations()) {
                        System.out.println('Annotation in Method ''+ method + '' : ' + anno);
                        }
                    MethodInfo methodAnno = method.getAnnotation(MethodInfo.class);
                    if (methodAnno.revision() == 1) {
                        System.out.println('Method with revision no 1 = '+ method);
                        }
     
                } catch (Throwable ex) {
                        ex.printStackTrace();
                        }
            }
        }
        } catch (SecurityException | ClassNotFoundException e) {
                e.printStackTrace();
             }
        }
     
    }

## 编译时源码级注解处理
APT(Annotation Processing Tool)是一种处理注解的工具，它对源代码文件进行检测，并找出源文件所包含的注解信息，然后针对注解信息进行额外的处理。  
Java 提供的 javac.exe 编译工具有一个 `-processor` 选项，该选项可以指定一个注解处理器，如果编译时，指定了注解处理器，那么这个注解处理器将会起作用。
```
javac -prcessor ProcessorName1,ProcessorName2,.... sourceFiles
```  
每个注解处理器都需要实现 `javax.annotation.processing` 包下的 `Processor` 接口。不过实现该接口必须实现它里边的所有方法，因此通常会采用继承 `AbstractProcessor` 的方式来实现注解处理器。  
一个注解处理器可以处理一个或多个注解。需要指定将要处理的注解，处理器可以声明具体的注解类型或诸如 `com.horstmann＊` 这样的通配符（com.horstmann包及其所有子包中的注解），甚至是 `＊`（所有注解）。

## 字节码工程-ASM，修改编译后的字节码
我们已经看到了我们是怎样在运行期或者在源码级别上对注解进行处理的。还有第3种可能：在字节码级别上进行处理。除非将注解在源码级别上删除，否则它们会一直存在于类文件中。类文件格式是归过档的，这种格式相当复杂，并且在没有特殊类库的情况下，处理类文件具有很大的挑战性。但是ASM库就是这样的特殊类库之一。

```java
// 举例做了如下注解
@LogEntry(logger="global")
public int hashCode()
// 那么,每次调用该方法都会打印一条信息
/**
1. 加载类文件的字节码
2.定位所有方法
3.对于每个方法,检查它是否有一个LogEntry注解
4.如果有,再方法开始部分添加字码指令(用于打印日志的字节码指令)
**/
```

## 加载时修改字节码
将字节码工程延迟到载入时,即类加载器加载类的时候.  
设备（ Instrumentation ）API提供了一个安装字节码转换器的挂钩,不过,必须在程序的main方法调用之前安装转换器
1. 实现一个具有下面这个方法的类
   ```java
   public static void premain(String arg,Instrumentation  instr)
   ```
   当加载代理的时候,此方法会被调用, `instr` 可以用来安装各种挂钩
2. 制作一个清单文件EntryLoggingAgent.mf来设置Premain-Class属性
3. 将代理打包,并生成jar文件
4. 为了运行一个具有改代理的Java程序,需要(在idea的破解文件看过)

   ```shell
   java -javaagent:EntryLoggingAgent.jar=set.Item -classpath .:asm/lib/\* set.SetTest
   ```
	<center><img src="pics/instruction.png" width="50%"></center> 
