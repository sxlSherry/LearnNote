# JVM——堆外内存

> 分享人——沈霄莉

[TOC]

## 1. 堆外内存是什么

​	在JAVA中，JVM内存指的是堆内存，机器内存中，不属于堆内存的部分即为堆外内存。堆外内存也被称为直接内存。

## 2. 堆外内存申请

### 2.1 Unsafe类操作堆外内存

​	sun.misc.Unsafe提供了一组方法来进行堆外内存的分配，重新分配，以及释放。

- public native long allocateMemory(long size); —— 分配一块内存空间。
- public native long reallocateMemory(long address, long size); —— 重新分配一块内存，把数据从address指向的缓存中拷贝到新的内存块。
- public native void freeMemory(long address); —— 释放内存。

举例：

```csharp
Unsafe unsafe = Unsafe.getUnsafe();
```

发现该方法被拒绝了，抛出了异常：java.lang.SecurityException: Unsafe
源码如下：

```java
`Unsafe f = Unsafe.getUnsafe();`
```

于是，只能使用反射来做这件事； 

```java
`Field f = Unsafe.``class``.getDeclaredField(``"theUnsafe"``);``f.setAccessible(``true``);``Unsafe us = (Unsafe) f.get(``null``);``long` `id = us.allocateMemory(``1000``);`
```

其中，allocateMemory 返回一个指针，并且其中的数据是未初始化的。如果要释放这部分内存的话，需要调用 freeMemory 或者 reallocateMemory 方法。

从 nio 时代开始，可以使用 ByteBuffer 等类来操纵堆外内存了： 

```
`ByteBuffer buffer = ByteBuffer.allocateDirect(numBytes);`
```

### 2.2 NIO类操作堆外内存

现在也可使用 ByteBuffer 等类来操纵堆外内存了，用NIO包下的ByteBuffer分配直接内存则相对简单。

```java
ByteBuffer buffer = ByteBuffer.allocateDirect(10 * 1024 * 1024);
```

## 3.堆外内存垃圾回收

对于内存，除了关注怎么分配，还需要关注如何释放。从JAVA出发，习惯性思维是堆外内存是否有垃圾回收机制。

考虑堆外内存的垃圾回收机制，需要了解以下两个问题：

1. 堆外内存会溢出么？
2. 什么时候会触发堆外内存回收？

### 3.1 堆外内存会溢出么

通过修改JVM参数：-XX:MaxDirectMemorySize=40M，将最大堆外内存设置为40M。

既然堆外内存有限，则必然会发生内存溢出。

为模拟内存溢出，可以设置JVM参数：-XX:+DisableExplicitGC，禁止代码中显式调用System.gc()。

可以看到出现OOM。

得到的结论是，堆外内存会溢出，并且其垃圾回收依赖于代码显式调用System.gc()。 

### 3.2 什么时候会触发堆外内存回收

关于堆外内存垃圾回收的时机，首先考虑堆外内存的分配过程。JVM在堆内只保存堆外内存的引用，用DirectByteBuffer对象来表示。每个DirectByteBuffer对象在初始化时，都会创建一个对应的Cleaner对象。

这个Cleaner对象会在合适的时候执行unsafe.freeMemory(address)，从而回收这块堆外内存。

当DirectByteBuffer对象在某次YGC中被回收，只有Cleaner对象知道堆外内存的地址。当下一次FGC执行时，Cleaner对象会将自身Cleaner链表上删除，并触发clean方法清理堆外内存。此时，堆外内存将被回收，Cleaner对象也将在下次YGC时被回收。

如果JVM一直没有执行FGC的话，无法触发Cleaner对象执行clean方法，从而堆外内存也一直得不到释放。

其实，在ByteBuffer.allocateDirect方式中，会主动调用System.gc()强制执行FGC。

**JVM觉得有需要时，就会真正执行GC操作**

## 4. 什么时候用堆外内存？

堆外内存的使用场景非常巧妙。第三方堆外缓存管理包ohc(off-heap-cache)给出了详细的解释。

摘了其中一段。

When using a very huge number of objects in a very large heap, Virtual machines will suffer from increased GC pressure since it basically has to inspect each and every object whether it can be collected and has to access all memory pages. A cache shall keep a hot set of objects accessible for fast access (e.g. omit disk or network roundtrips). The only solution is to use native memory - and there you will end up with the choice either to use some native code (C/C++) via JNI or use direct memory access.

大概的意思如下：

考虑使用缓存时，本地缓存是最快速的，但会给虚拟机带来GC压力。使用硬盘或者分布式缓存的响应时间会比较长，这时候「堆外缓存」会是一个比较好的选择。

## 5. 如何用堆外内存？ 

​	在第一章中介绍了两种分配堆外内存的方法，Unsafe和NIO。对于两种方法只是停留在分配和回收的阶段，距离真正使用的目标还很遥远。在第三章中提到堆外内存的使用场景之一是缓存。那是否有一个包，支持分配堆外内存，又支持KV操作，还无需关心GC。

​	答案当然是有的。有一个很知名的包，**Ehcache**。Ehcache被广泛用于Spring，Hibernate缓存，并且支持堆内缓存，堆外缓存，磁盘缓存，分布式缓存。此外，Ehcache还支持多种缓存策略。

其仓库坐标如下：

```xml
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>3.4.0</version>
</dependency>
```

接下来就是写代码进行验证：

```java
public class HelloHeapServiceImpl implements HelloHeapService {

    private static Map<String, InHeapClass> inHeapCache = Maps.newHashMap();

    private static Cache<String, OffHeapClass> offHeapCache;

    static {
        ResourcePools resourcePools = ResourcePoolsBuilder.newResourcePoolsBuilder()
                .offheap(1, MemoryUnit.MB)
                .build();

        CacheConfiguration<String, OffHeapClass> configuration = CacheConfigurationBuilder
                .newCacheConfigurationBuilder(String.class, OffHeapClass.class, resourcePools)
                .build();

        offHeapCache = CacheManagerBuilder.newCacheManagerBuilder()
                .withCache("cacher", configuration)
                .build(true)
                .getCache("cacher", String.class, OffHeapClass.class);


        for (int i = 1; i < 10001; i++) {
            inHeapCache.put("InHeapKey" + i, new InHeapClass("InHeapKey" + i, "InHeapValue" + i));
            offHeapCache.put("OffHeapKey" + i, new OffHeapClass("OffHeapKey" + i, "OffHeapValue" + i));
        }
    }

    @Data
    @AllArgsConstructor
    private static class InHeapClass implements Serializable {
        private String key;
        private String value;
    }

    @Data
    @AllArgsConstructor
    private static class OffHeapClass implements Serializable {
        private String key;
        private String value;
    }

    @Override
    public void helloHeap() {
        System.out.println(JSON.toJSONString(inHeapCache.get("InHeapKey1")));
        System.out.println(JSON.toJSONString(offHeapCache.get("OffHeapKey1")));
        Iterator iterator = offHeapCache.iterator();
        int sum = 0;
        while (iterator.hasNext()) {
            System.out.println(JSON.toJSONString(iterator.next()));
            sum++;
        }
        System.out.println(sum);
    }
}
```

其中`.offheap(1, MemoryUnit.MB)`表示分配的是堆外缓存。

Demo很简单，主要做了以下几步操作：

1. 新建了一个Map，作为堆内缓存。
2. 用Ehcache新建了一个堆外缓存，缓存大小为1MB。
3. 在两种缓存中，都放入10000个对象。
4. helloHeap方法做get测试，并统计堆外内存数量，验证先插入的对象是否被淘汰。

使用Java VisualVM工具Dump一个内存镜像。

打开镜像，堆里有10000个InHeapClass，却没有OffHeapClass，表示堆外缓存中的对象的确没有占用JVM内存。 

接着测试helloHeap方法。

输出：

> {"key":"InHeapKey1","value":"InHeapValue1"}
>
> null
>
> ……(此处有大量输出)
>
> 5887

输出表示堆外内存启用了淘汰机制，插入10000个对象，最后只剩下5887个对象。

如果堆外缓存总量不超过最大限制，则可以顺利get到缓存内容。

**总体而言，使用堆外内存可以减少GC的压力，从而减少GC对业务的影响。**