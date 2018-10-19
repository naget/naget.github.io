---
layout: post
title: "ThreadLocal详解"
date: 2018-10-19
excerpt: "大家一起用不安全，那就一人一个呗"
tags: [JDK源码]
comments: true
---
保证线程安全一是可以同步对共享资源的操作和访问，二是不共享。就像ThreadLocal这样，给每个线程分一个对象，每个线程也只能访问到自己的这个对象，从而保证线程安全。比如SimpleDateFormat这个类，咋也没想到它是线程不安全的，既然线程不安全我们就给每一个线程都实例化一个SimpleDateFormat，自己用自己的就安全了，ThreadLocal就给我们实现了分配线程私有对象这么个功能。
### 使用
```java
public class ThreadlocalDemo {
    static  ThreadLocal<SimpleDateFormat> local = new ThreadLocal<>();
    public static void main(String[] args) throws Exception {
        local.set(new SimpleDateFormat());
        SimpleDateFormat sdf = local.get();
        Thread thread = new Thread(){
            @Override
            public void run() {
                local.set(new SimpleDateFormat());
                SimpleDateFormat sdf = local.get();
            }
        };
        thread.start();
        thread.join();
    }
}
```
我们声明了一个全局的ThreadLocal类型变量local，然后在主线程和thread线程中通过local.set设置自己的SimpleDateFormat，然后通过local.get来得到自己的SimpleDateFormat。两个线程各用各的，不争不抢。也许你觉得local.set每个线程都需要做一个遍，有点繁琐，那么可以重写ThreadLocal的initialValue方法，如下

```java
     static ThreadLocal<SimpleDateFormat> local = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue() {
            System.out.println("Thread:"+Thread.currentThread().getName());
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };
```
那么每个线程通过local.get就可以直接得到属于自己线程的专属SimpleDateFormat对象。
### get方法源码解析
以上过程总会给人一个错觉，仿佛是threadlocal维护了线程和线程特有对象之间的关系，每当我们调用local.get的时候，threadlocal就检测当前线程，并取出当前线程对应的对象。其实不是这样的，线程特有对象是存在线程对象中而不是threadlocal中，Thread中有这么个字段threadlocals

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
这个map中就保存了此线程的特有对象。了解了这一点我们来看一下local.get的过程

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//得到线程t中的threadlocals
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
第三行代码取出了当前线程的threadloacls，这是一个map，也就是说一个线程可以有许多线程特有变量，这些特有变量都被存到了map中，当我们去取特有变量的时候，需要告诉线程要取哪个特有变量，如何分辨这些特有变量呢？第五行代码map.getEntry(this)，这个map中，特有对象做为值被存入，键是谁呢？键就是对应的threadlocal对象。于是，我们传入this，也就是此刻调用get的threadlocal对象，就可以取出这个threadlocal所对应的特有对象，就如上面第一段代码中local引用的threadlocal对象是键，对应的值就是SimpleDateFormat对象。
如果此线程还没有线程特有对象，或者线程特有对象中没有我们查找的这个ThreadLocal对象，那么我们就需要执行初始化方法，就是是代码的最后一句

```java
setInitialValue()
```
我们进入这个方法

```java
private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
哎？第一句调用的这个方法InitialValue有没有很熟悉，这就是我们文章开始重写的那个方法（如果不重写，这个方法直接返回null）。还记得当时我们直接返回了一个SimpleDateFormat对象，也就是这里的value，并且把它加入到当前线程的threadlocals引用的map当中，最后返回这个value，get方法圆满结束。

### 浅析内存泄漏
线程中维护的那个threadlocals就是ThreadLocalMap类型的，这个map内部维护了一个Entry数组，每个Entry就是一个键值对，键就是一个ThreadLocal对象，值就是一个线程特有对象（如文章开头例子中的SimpleDateFormat对象）。当我们调用ThreadLocal的set方法时，最终调用的是ThreadLocalMap的set方法。
#### ThreadLocalMap的set方法

```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];//位置i上已经有元素
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {//元素相等，则覆盖原值
                    e.value = value;
                    return;
                }

                if (k == null) {//位置i的元素已陈旧,替换
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)//清理陈旧元素，如果没有元素被清理，考虑是否扩容
                rehash();
        }
```
set方法很简单，就是在对应的位置i放入元素。如果位置i没有元素的话，直接新建entry，并且检查是否需要扩容。如果位置i已经有元素，则判断这个entry的key与要插入的是否相等，相等则覆盖；如果为Null，则进行替换。也许你会觉得这两种情况结果不是一样嘛，一个覆盖，一个替换，都是更新了这个Entry嘛，其实不然，替换的话，会调用replaceStaleEntry，在这个方法中还会进行陈旧entry的清理。前面的代码注释中也看到了这个词“陈旧”，什么样的entry是陈旧呢？首先这个entry不为null，但他的key为null，那么这个entry就是陈旧的。一个Entrty为什么会陈旧呢？也就是说它的值不为Null，键为什么会为Null呢？这就是一个比较巧妙的设计了，用到了java中的弱引用。entry的key是一个指向Threadlocal对象的弱引用，也就是说当没有其他强引用指向这个ThreadLocal对象时，这个entry的key就可能被GC回收，从而key指向了null。当我们调用get和set方法时都可能触发一个检查机制，来处理这些陈旧的entry，从而避免内存泄漏。但这些机制的触发都不是一定的，还是有内存泄漏的可能，为了避免内存泄漏，我们在使用Threadlocal时一定要记得自己手动调用remove，进行陈旧entry的处理。总之，threadlocal为了解决内存泄漏使用了弱引用，但任然存在内存泄漏的可能，所以用完最好还是remove一下。

### ThreadLocal的Hash碰撞解决方案
每个线程中的ThreadLocalMap也是一个map，装入键值对的时候通过threadlocal的hash值来决定对应的位置。在上边的set代码中可以看到

```java
int i = key.threadLocalHashCode & (len-1);
```
i就是对应的位置，并且之后是一个for循环，如果这个位置有元素就会选择i+1（如果i+1数组越界，就置为0，这也是nextIndex()方法中实现的逻辑）位置进行检测。也就是说这里使用了线性探测的方式来解决hash冲突，为什么使用线性探测呢？毕竟总有人觉得跟链表法比起来，尤其是跟使用了红黑树进行优化的Hashmap中的链表法比起来，线性探测很可能出现连续多次的冲突。其实，Threadlocal通过在hash值上做了个“小手脚”，使得求得的Hash值不会出现太多冲突，大致分布很均匀。这个小手脚是什么呢？我们看一下这里hash值的求法，threadlocal.threadLocalHashCode就得到了这个hash值，threadLocalHashCode做了什么呢？

```java
 
 private final int threadLocalHashCode = nextHashCode();
 private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```
threadLocalHashCode就是nextHashCode()的返回值，nextHashCode()就是将nextHashCode这个值加HASH_INCREMENT。nextHashCode指向一个AtomicInteger对象

```java
private static AtomicInteger nextHashCode =
        new AtomicInteger();
```
HASH_INCREMENT是一个常量

```java
private static final int HASH_INCREMENT = 0x61c88647;
```
总之呢，第一个hash值求出来就是0，然后第二个是0+HASH_INCREMENT，第三个0+HASH_INCREMENT+HASH_INCREMENT...
这个数很有魔力，这个数可以使得每次求得的hash值放入entry数组中分布均匀。有多均匀，请看实验结果
![在这里插入图片描述](https://img-blog.csdn.net/20181019142117670?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjQwOTQ2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
实验代码如下

```java
public class Hash_increment {
    private static int hash = 0;
    private static final int HASH_INCREMENT = 0x61c88647;
    public static void main(String[] args) {
        int len = 16;
        int time = len/4*3;
        while (len<200){
            System.out.print("len等于"+len+"时：");
            for (int i=0;i<time;i++){
                System.out.print(String.valueOf(increment()&len-1)+" ");
            }
            System.out.println();
            len=len<<1;
            time = len/4*3;
        }

    }
    public static int increment(){
        return hash+=HASH_INCREMENT;
    }
}
```
至于这个数为啥这么厉害，更多的和数学相关，叫做斐波那契散列。数学渣渣就不在这多说了，再见。
