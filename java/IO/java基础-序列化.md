# Java基础-序列化
{docsify-updated}

- [Java基础-序列化](#java基础-序列化)
	- [序列化方式](#序列化方式)
	- [Serializable 接口](#serializable-接口)
		- [引用成员的序列化](#引用成员的序列化)
		- [自定义序列化](#自定义序列化)
	- [版本](#版本)
	- [Externalizable 接口](#externalizable-接口)
	- [总结](#总结)


### 序列化方式
实现 Serializable 或者 Externailizable 接口。

### Serializable 接口
一旦某个类实现了 Serializable 接口，就可以使用如下方式序列化：
1. 创建 ObjectOutputStream 对象.
2. 调用 ObjectOutputStream 的 writeObject(object)输出可序列化对象

反序列化：
1. 创建 ObjectInputStream 对象.
2. 调用 ObjectOutputStream 的 readObject()反序列化.

反序列化时读取的只是对象的数据，而不包含类的信息，其实，序列化时也仅仅是保存了对象的信息。因此，反序列化时需要提供类信息，让JVM知道还原成什么类型对象及如何还原对象。

**反序列化时，并没有调用类的构造器来构造对象。**

当一个可序列化类有一个或多个父类时（直接父类和间接父类），这些父类要么有无参构造函数，要么也是可序列化的。如果是不可序列化的，只有无参构造函数，则父类成员变量中的值不会被序列化到二进制流中。

#### 引用成员的序列化
递归序列化：当实例化某个对象时，系统会递归地序列化对象的实例变量，如果实例变量是引用类型，则被引用的对象也会被实例化。

如果一个类中含有引用类型的成员变量，那么引用类型的类必须也是可序列化的，这个类才是可序列化的，否则即使实现了 Serializable 接口，也是不可序列化的。

假设同一个对象被多个其它对象引用，如果同时序列化这些对象，那么这个对象可能会被序列化多次，一来浪费空间，二来在反序列化时，可能会生成多个对象，这与实际情况不符。所以 Java 使用了下面特殊的序列化算法：
1. 所有保存到二进制流中的对象都有一个序列化编号
2. 当程序试图序列化一个对象时，程序将检查对象是否已经被序列化过，只有对象从未被序列化过，系统才会将对象转换成字节序列并输出。
3. 如果某个对象已经被序列化过，程序将只会输出一个序列化编号，而不会再次序列化该对象
<center><img src="pics/serialize.jpg" alt="" width="40%"></center>

上述机制会引入一个问题：假设有一个可变对象，当第一次序列化后，改变了对象的状态（某些成员变量值），再次序列化时，因为已经被序列化过，所以系统只是输出一个编号，所以序列化的值并不是修改后的值。

#### 自定义序列化
1. 通过在实例变量前添加 transient 关键字，可以让 JAVA 序列化机制在序列化时完全忽略该实例变量。
2. 还可以通过下述签名方法来完全控制某个实例变量的序列化：
    + private void writeObject(ObjectOutputStream out) throws IOException
    + private void readObject(ObjectInputStream in) throws IOException,ClassNotFoundException
    + private void readObjectNoData(ObjectOutputStream out) throws ObjectStreamException

例如：
```
private void writeObject(ObjectOutputStream out) throws IOException{
    out.writeObject(new String(name).reverse());
    out.writeInt(age);
}

private void readObject(ObjectInputStream in) throws IOException,ClassNotFoundException{
    this.name = (String)in.readObjetc().reverse();
    this.age = in.readInt();
}
```

### 版本
Java序列化机制允许为序列化类提供一个 `private static final long serialVersionUID` 的值，用于标识 Java 类序列化的版本，也就是说，如果一个类升级后，只要它的 serialVersionUID 类变量值保持不变，序列化机制也会把它们当成同一个序列化版本。如果不显示定义 serialVersionUID 类变量的值，该类变量值将由 JVM 根据类的相关信息计算，而修改后的类计算结果与修改前的类的计算结果往往不同，从而造成对象的反序列化因为类版本不兼容而失败。

可以通过 JDK 安装路径 bin 下的 serialver.exe 工具来获得该类的 serialVersionUID 类变量的值： `serialver Person`

### Externalizable 接口
```
void writeExternal(ObjectOutput out) throws IOException;
void readExternal(ObjectInput in) throws IOException, ClassNotFoundException;
```
实际上，使用实现 Externalizable 接口方式的序列化方式与前面介绍的自定义序列化方式非常相似，只是 Externalizable 接口强制自定义序列化。
<center><img src="pics/serialize-compare.jpg" alt="" width="60%"></center>

### 总结
1. 对象的类名、实例变量（包括基本类型、数组、其它对象的引用）都会被序列化；方法、类变量、 transient 实例变量都不会被序列化
2. 实现 Serializable 接口的类如果需要让某个实例变量不被序列化，则在该实例变量前加 transient 修饰.
3. 保证类是可序列化的必须保证引用的实例变量类型也是可序列化的，否则需要时用 transient 修饰，否则该类也是不可序列化的
4. 反序列化时必须要有序列化对象的 class 文件
5. 当通过文件、网络来反序列化对象时，必须按实际写入的顺序来读取。