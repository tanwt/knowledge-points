# 经典回答

String 是Java 语言非常基础和重要的类，提供了构造和管理字符串的各种基本逻辑。他是典型的immutable 类，被声明为final Class ，所有属性也都是final 的。也由于它的不可变性，类似拼接、裁剪字符串等动作，都会产生新的String 对象。由于字符串操作的普遍性，所以相关操作的效率往往对应用有明显影响。

StringBuffer 是为了解决上面提到的拼接产生太多中间对象的问题而提供的一个类，我们可以用append 或者add 方法，把字符串添加到已有序列的末尾或者指定位置。
StringBuffer 本质是一个线程安全的可修改字符序列，它保证了线程安全，也随之带来了额外的性能开销，所以除非有线程安全的需要，不然还是推荐使用它的后继者，也就是StringBuilder。

StringBuilder 是JDK 1.5 新增的，在能力上和StringBuffer 没有本质区别，但是它去掉了线程安全的部分，有效减小了开销，是绝大部分情况下进行字符串拼接的首选。

# 拓展
###  字符串的设计与实现考量
###### （1）StringBuffer 和StringBuilder 的实现

1. 区别实现

为了实现修改字符序列的目的，StringBuffer 和 StringBuilder 底层都是利用可修改的（char，JDK 9 以后是 byte）数组，二者都继承了 AbstractStringBuilder，里面包含了基本操作，区别仅在于最终的方法是否加了 synchronized。

```
//两个类都继承至 AbstractStringBuilder

public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
{ 

public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
{
```
2. 扩容机制


另外，这个内部数组应该创建成多大的呢？如果太小，拼接的时候可能要重新创建足够大的数组；如果太大，又会浪费空间。目前的实现是，构建时初始字符串长度加 16（这意味着，如果没有构建对象时输入最初的字符串，那么初始值就是 16）。我们如果确定拼接会发生非常多次，而且大概是可预计的，那么就可以指定合适的大小，避免很多次扩容的开销。扩容会产生多重开销，因为要抛弃原有数组，创建新的（可以简单认为是倍数）数组，还要进行 arraycopy


```
// 默认16 大小的数组长度
public StringBuilder() {
        super(16);
    }

public StringBuffer() {
    super(16);
}
```
```
// 扩容函数
//minimumCapacity = 已用长度 + 添加长度
 private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }
```

###### （2）String 类为什么要设计成不可变

经过粗略统计过，把常见应用进行堆转储（Dump Heap），然后分析对象组成，会发现平均 25% 的对象是字符串，并且其中约半数是重复的。如果能避免创建重复字符串，可以有效降低内存消耗和对象创建开销。

所以JVM 把字符串常量放在字符串常量池中，**设计成不可变主要是为了并发安全性防止一个线程修改对象对另外的线程造成影响，所以每次对字符串进行更新或生成新的对象**。

我在前面介绍过，String 是 Immutable 类的典型实现，原生的保证了基础线程安全，因为你无法对它内部数据进行任何修改，这种便利甚至体现在拷贝构造函数中，由于不可变，Immutable 对象在拷贝时不需要额外复制数据。


###### （3）字符串不可变带来的问题


在JDK 1.6 之前，字符串常量池属于Perm Gen（方法区）。
方法区与Java 堆一样，在JVM 内存模型中的设计是能够被所有线程可见，它用于存储已被JVM 加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。因为这些数据基本不会被GC ，所以方法区也称“永久代”。但是这块内存区域比较小，所以字符串常量池在JDK 1.6 之后就迁移到了堆中。

###### （4）字符串常量池的实现

- 在HotSpot VM里实现的string pool功能的是一个StringTable类，它是一个Hash表，默认值大小长度是1009；这个StringTable在每个HotSpot VM的实例只有一份，被所有的类共享。字符串常量由一个一个字符组成，放在了StringTable上。
- 在JDK6.0中，StringTable的长度是固定的，长度就是1009，因此如果放入String Pool中的String非常多，就会造成hash冲突，导致链表过长，当调用String#intern()时会需要到链表上一个一个找，从而导致性能大幅度下降
- 在JDK7.0中，StringTable的长度可以通过参数指定：

```
-XX:StringTableSize=66666
```
###### （5）字符串缓存
创建字符串有两种方法：

```
String string1 = "123";
String string2 = new String("123");
```
当采用第一种方式创建字符串时先从常量池里面看有没有相同值的对象，有就返回该对象的引用，没有就新建一个对象

```
String string = "123";
String string1 = "123";
System.out.println(string == string1);
//运行结果为true
```

采用第二种的时候，直接创建对象
如果用new 的方式也想判断有没缓存，可以采用字符串的intern() 方法

```
public String intern()返回字符串对象的规范表示。 
最初为空的字符串池由String类String 。 

当调用intern方法时，如果池已经包含与equals(Object)方法确定的相当于此String对象的字符串，则返回来自池的字符串。 
否则，此String对象将添加到池中，并返回对此String对象的引用。 

由此可见，对于任何两个字符串s和t ， s.intern() == t.intern()是true当且仅当s.equals(t)是true 。 

所有文字字符串和字符串值常量表达式都被实体化。 字符串文字在The Java™ Language Specification的 3.10.5节中定义。 

结果 
一个字符串与该字符串具有相同的内容，但保证来自一个唯一的字符串池。
```
######  （6）字符串的拼接原理
前面我们提到StringBuilder 和StringBuffer 的产生是因为String 拼接产生大量中间无用对象

但是字符串拼接真的这么不堪吗？

```
String userName = "Andy";
String age = "24";
String job = "Developer";
String info = userName + age + job;
```

要得到上面的info，就会userName和age拼接生成临时一个String对象t1,内容为Andy24，然后有t1和job拼接生成最终我们需要的info对象，这其中，产生了一个中间的t1，而且t1创建之后，没有主动回收，势必会占一定的空间。如果是一个很多(假设上百个，多见于对对象的toString的调用)字符串的拼接，那么代价就更大了，性能一下会降低很多。

**来自编译器的优化**


一个Java程序如果想运行起来，需要经过两个时期，编译时和运行时。在编译时，Java 编译器(Compiler)将java文件转换成字节码。在运行时，Java虚拟机（JVM）运行编译时生成的字节码。通过这样两个时期，Java做到了所谓的一处编译，处处运行。

我们反编译一下上面代码：

```
public class Concatenation {
  public static void main(String[] args) {
      String userName = "Andy";
      String age = "24";
      String job = "Developer";
      String info = userName + age + job;
      System.out.println(info);
  }
}
```

```
Compiled from "Concatenation.java"
public class Concatenation {
  public Concatenation();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return       
  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String Andy
       2: astore_1
       3: ldc           #3                  // String 24
       5: astore_2
       6: ldc           #4                  // String Developer
       8: astore_3
       9: new           #5                  // class java/lang/StringBuilder
      12: dup
      13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
      16: aload_1
      17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      20: aload_2
      21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      24: aload_3
      25: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      28: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      31: astore        4
      33: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
      36: aload         4
      38: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      41: return
}

```
其中，ldc，astore等为java字节码的指令，类似汇编指令。后面的注释使用了Java相关的内容进行了说明。 我们可以看到上面有很多StringBuilder,但是我们在Java代码里并没有显示地调用，这就是Java编译器做的优化，当Java编译器遇到字符串拼接的时候，会创建一个StringBuilder对象，后面的拼接，实际上是调用StringBuilder对象的append方法。这样就不会有我们上面担心的问题了。

**但是仅仅靠优化不能解决问题**

但遇到这种操作的时候也无济于事

```
public void  implicitUseStringBuilder(String[] values) {
  String result = "";
  for (int i = 0 ; i < values.length; i ++) {
      result += values[i];
  }
  System.out.println(result);
}
```
反编译：

```
   8: if_icmpge     38
      11: new           #5                  // class java/lang/StringBuilder
      14: dup
      15: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
```
这里会循环创建StringBuilder。我们需要人工优化一下。

```
public void explicitUseStringBuider(String[] values) {
  StringBuilder result = new StringBuilder();
  for (int i = 0; i < values.length; i ++) {
      result.append(values[i]);
  }
}
```

```
public void explicitUseStringBuider(java.lang.String[]);
    Code:
       0: new           #5                  // class java/lang/StringBuilder
       3: dup
       4: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
       7: astore_2
       8: iconst_0
       9: istore_3
      10: iload_3
      11: aload_1
      12: arraylength
      13: if_icmpge     30
      16: aload_2
      17: aload_1
      18: iload_3
      19: aaload
      20: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      23: pop
      24: iinc          3, 1
      27: goto          10
      30: return
```
从上面可以看出，13: if_icmpge 30和27: goto 10构成了一个loop循环，而0: new #5位于循环之外，所以不会多次创建StringBuilder.

所以为什么要实现StringBuilder 和实现StringBuffer的原因
