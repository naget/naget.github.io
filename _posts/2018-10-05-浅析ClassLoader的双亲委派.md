---
layout: post
title: "浅析ClassLoader的双亲委派"
date: 2018-10-05
excerpt: "双亲委派只是汇报一声，重活还得自己干哇"
tags: [JDK源码]
comments: true
---
### 小引

```java
public class Demo {
    public static void main(String[] args) {
        System.out.println(Demo.class.getClassLoader().toString());
    }
}
```
输出

```java
sun.misc.Launcher$AppClassLoader@18b4aac2
```
**小n**：都说java中的类加载器是双亲委派原则，为啥加载我这个类的是AppClassLoader呢？不应该是类加载器的大佬BootStrapClassLoader吗？
**大N**：看来你还没有完全理解双亲委派原则呀，今天我们就来谈一谈！

### 正文
我们平常敲的代码xx.java都是源文件，给程序员看的，机器可是看不懂的。于是我们需要先将java源文件编译成为xx.class文件，然后将这个class文件交给Java虚拟机。虚拟机再将这些class文件根据不同的平台转换成为不同的机器码在不同的机器上进行执行，这也就是Java所谓的跨平台。今天我们主要说这个步骤——将class文件交给虚拟机。
原来总会这样说：类加载就是双亲委派机制，当类加载器收到类加载的请求的时候自己不会去加载，而是交给父类加载器进行加载。
这句话不能说错误，但确实不怎么严谨。让人首先感觉所有类都是顶层类加载器进行加载的，就像文章开头小n的问题，他认为双亲委派就是把加载任务都给了bootStrap类加载器。

我们看一下类加载器实际到底是如何进行加载的

```java
// First, check if the class has already been loaded,首先检查这个类是否已经加载（最终的检查方法是个本地方法）
            Class<?> c = findLoadedClass(name);
            if (c == null) {//如果这个类还没有被加载
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//有父加载器就交给父加载器去加载
                        c = parent.loadClass(name, false);
                    } else {//父加载器为空则调用BootstrapClassLoader
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
                    //在指定的路径下去寻找需要加载的class文件
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
```
1、查看这个类是否已经被加载，如果是，返回这个类；如果没有，交给父类加载器。
2、如果父类加载器为null，那么传给bootstrap类加载器。
*前两步就是我们对双亲委派的一个感性认识，感觉最终传到bootstrap之后，bootstrap直接加载就完事了，其实不然
3、bootstrap首先查看这个class文件是否已经加载，如果还没有加载，就去**指定路径**中寻找这个class文件，这个查找的过程就是以上代码中对应的findClass()方法。如果找不到，返回null。注意这个指定路径，对与bootstarp类加载器来说这个路径就是JDK\jre\lib，对与ExtClassloader这个指定路径就是JDK\jre\lib\ext。bootstrap在这个路径下找不到我们这个Demo的class文件，于是就直接返回null。
4、bootstrap返回之后，extClassLoader进行一样的寻找，也没找到，于是再次返回，就到了AppClassLoader中。AppClassLoader在的对应路径是classpath，在这个路径下找到了Demo.class，并将它加载进虚拟机。

### 总结和附加几个知识点
- 父类加载器和子类加载器不是继承的关系，而是组合。子类加载器中有一个字段指向父类加载器。注意读法（父 （停顿） 类加载器）
- 双亲委派是为了避免重复加载同一个类，保证类的唯一性
- 双亲委派有一个向上传递的过程和向下查找的过程
