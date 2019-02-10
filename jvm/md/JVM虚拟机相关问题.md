# JVM虚拟机相关问题

由于近期在做知识储备，在做很多的复习，把之前看过的内容重温一遍真的像重新看一遍一样，真的是一入JAVA深似海。

## jvm加载以及内存相关问题

### 1.loadClass和forName的区别

首先我们需了解一下类的装在过程，如图所示: 

p1.png

#### loadClass

接下来要了解这两个的本质区别，最直接暴力的方法就是看源码。因此我们先来看一下`java.lang.ClassLoader#loadClass(java.lang.String, boolean)`的源码。

```java
/**
     * @param  name
     *         The <a href="#name">binary name</a> of the class   类的二进制名
     *
     * @param  resolve
     *         If <tt>true</tt> then resolve the class   是否解决（连接）这个类 默认是false
     *
     * @return  The resulting <tt>Class</tt> object
     *
     * @throws  ClassNotFoundException
     *          If the class could not be found
*/
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                //双亲委派模型加载机制
                if (parent != null) {
                    c = parent.loadClass(name, false);   
                } else {
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
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {  //是否进行链接  默认是false
            resolveClass(c);
        }
        return c;
    }
}
```

由标注后的方法注释，我们可以了解到，在默认调用`java.lang.ClassLoader#loadClass(java.lang.String)`方法时， `ClassLoader`会默认调用内部方法 也就是 `loadClass(name, false);` 因此我们所获取到的类是还没有链接的，也就没有初始化完成。

#### forName

我们接着来看一下 `java.lang.Class#forName(java.lang.String)`源码

```java
@CallerSensitive
public static Class<?> forName(String className)
            throws ClassNotFoundException {
    Class<?> caller = Reflection.getCallerClass();
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}
```

  我们接着看`java.lang.Class#forName0`  我们这里 

```java
/** Called after security check for system loader access checks have been made.
 * @param name       fully qualified name of the desired class  类全限定名
 * @param initialize if {@code true} the class will be initialized.   是否初始化
 *                   See Section 12.4 of <em>The Java Language Specification</em>.
 * @param loader class loader from which the class must be loaded   当前类classloader
 * @return           class object representing the desired class
*/
private static native Class<?> forName0(String name, boolean initialize,
                                        ClassLoader loader,
                                        Class<?> caller)
    throws ClassNotFoundException;
```

我们可以看到 这是一个 native方法，有兴趣的同学可以去 这里查看一下对应的源码实现 [Class.c](http://hg.openjdk.java.net/jdk8u/jdk8u60/jdk/file/935758609767/src/share/native/java/lang/Class.c)。

我们可以看到 我们调用`Class.forName(clazz)`方法时，`Class`对象内部自动调用了 `forName0(className, true, ClassLoader.getClassLoader(caller), caller);`   这里 默认的就是初始化类。

#### 总结

经过两个方法源码实现对比，我们得出结论

- loadClass: 得到的class是还没有链接的，更别提初始化了。
- forName:得到的class是已经初始化完成。



### 2.String.intern()

由于现在普遍使用的是JDK8以上的版本，这里就对JDK8以上的版本做解释。

#### 定义

intern用来返回常量池中的某字符串，如果常量池中已经存在该字符串，则直接返回常量池中该对象的引用。否则，在常量池中加入该对象，然后 返回引用。

#### 相关问题

```java
String str1 = new String("a");
str1.intern();
String str2 = "a";
System.out.println(str1  == str2);
String str3 = new String("a") + new String("a");
str3.intern();
String str4 = "aa";
System.out.println(str4 == str3);
```

上段代码的输出结果？ 并分析。

答案： false,true。

##### 图解：

p2.png



这里首先 new String("a"), 会在常量池中先生成一个a,然后再堆中在创建出一个对象 ，当str1调用intern()时，发现常量池中已经有一个a了，所以不会把引用放进去，因此当str2赋值a时，是把常量池中的引用给了str2，因此 str1不等于str2。



接下来我们看一下 当str3时，首先常量池中只有a ，因此 str3这个字符串的引用可以通过调用intern(),将aa的引用放置在常量池中，那么当str4赋值 aa 时，就可以将常量池aa的引用 直接赋给str4,所以str4 == str3。



