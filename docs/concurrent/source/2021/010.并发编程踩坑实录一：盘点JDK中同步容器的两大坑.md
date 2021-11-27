## 并发编程踩坑实录一：盘点JDK中同步容器的两大坑！！

**大家好，我是冰河~~**

关注冰河的CSDN博客，学习最牛逼的【[精通高并发系列](https://blog.csdn.net/l1028386804/category_9719933.html)】专栏。

说实话，在实际的工作过程中，我在使用JDK中的并发容器时，确实踩过不少坑。为了让小伙伴们更好的消化这些知识，今天，首先和小伙伴们分享下使用同步容器时需要注意哪些问题，后续再为大家分享使用并发容器时需要注意哪些问题，以便大家在实际工作过程中尽量少走弯路。

相信很多小伙伴都知道，并发编程一直都是一个难点，不仅仅是Java语言，其他编程语言也是如此。说它难，不只是并发编程需要我们掌握的知识点比较繁杂，包括：操作系统知识、系统调度知识、CPU、时间片、内存、同步、异步、锁、重排序、多线程、线程池、悲观锁、乐观锁等等一系列知识。

而且并发编程还可能会导致线上生产环境出现一系列的诡异问题，并且这些问题重现几率小，排查困难，需要深入理解并发编程的相关知识才能很好的解决遇到的问题。

如何更好的学习高并发编程，不用着急，关注冰河的CSDN博客，订阅【[精通高并发系列](https://blog.csdn.net/l1028386804/category_9719933.html)】就好啦，我们一起进阶，一起吃透高并发编程。

**如果文章对你有点帮助，小伙伴们点赞、收藏、关注、评论、分享，走一波哇~~**

啰嗦了这么多，接下来，我们开始今天的主题，为大家分享下在使用JDK中的同步容器时，应该尽量避免哪些坑。

## 同步容器与并发容器

在JDK中，总体上可以将容器分为同步容器和并发容器。

![001](images\003\001.png)

**同步容器**一般指的是JDK1.5版本之前的线程安全的容器，同步容器有个最大的问题，就是性能差，容器中的所有方法都是用synchronized保证互斥，串行度太高。在JDK1.5之后提供了性能更高的线程安全的容器，我们称之为**并发容器**。

无论是同步容器还是并发容器，都可以将其分为四个大类，分别为：**List、Set、Map和Queue**，如下所示。

![002](images\003\002.png)

接下来，我们就简单聊聊使用JDK中的同步容器时，究竟要注意避免哪些坑。

## 同步容器的坑

在Java中，容器可以分为四大类：List、Set、Map和Queue，但是在这些容器中，有些容器并不是线程安全的，例如我们经常使用的ArrayList、HashSet、HashMap等等就不是线程安全的容器。

那么，根据我们在【[精通高并发系列](https://blog.csdn.net/l1028386804/category_9719933.html)】专栏学习的并发编程知识，**如何将一个不是线程安全的容器变成线程安全的呢？** 相信有很多小伙伴都能够想到一个办法，那就是把非线程安全的容器的方法都加上synchronized锁，使这些方法的访问都变成同步的。

没错，这确实是一种解决方案，例如，我们自定义一个 `CustomSafeHashMap`类，内部维护着一个`HashMap`，外界对`HashMap`的访问都加上了synchronized锁，以此来保证方法的原子性，例如下面的伪代码所示。

```java
public class CustomSafeHashMap<K, V>{
    private Map<K, V> innerMap = new HashMap<K, V>();
    public synchronized void put(K k, V v){
        innerMap.put(k, v);
    }
    
    public synchronized V get(K k){
        return innerMap.get(k);
    }
}
```

看到这里，一些小伙伴可能会想：**是不是所有的非线程安全的容器类都可以通过为方法添加synchronized锁来保证方法的原子性，从而使容器变得安全呢？**

是的，我们可以通过为非线程安全的容器方法添加synchronized锁来解决容器的线程安全问题。其实，在JDK中也是这么做的。例如，在JDK中提供了线程安全的List、Set和Map，它们都是通过synchronized锁保证线程安全的。

例如，我们可以通过如下方式创建线程安全的List、Set和Map。

```java
List list = Collections.synchronizedList(new ArrayList());
Set set = Collections.synchronizedSet(new HashSet());
Map map = Collections.synchronizedMap(new HashMap());
```

**那么，说了这么多，同步容器有哪些坑呢？**

### 坑一：竞态条件问题

在使用同步容器时需要注意的是，**在并发编程中，组合操作要时刻注意竞态条件**，例如下面的代码。

```java
public class CustomSafeHashMap<K, V>{
    private Map<K, V> innerMap = new HashMap<K, V>();
    public synchronized void put(K k, V v){
        innerMap.put(k, v);
    }
    
    public synchronized V get(K k){
        return innerMap.get(k);
    }
    
    public synchronized void putIfNotExists(K k, V v){
        if(!innerMap.containsKey(k)){
             innerMap.put(k, v);
        }
    }
}
```

其中，`putIfNotExists()`方法就包含组合操作。在高并发环境中，存在组合操作的方法可能就会存在竞态条件。

**也就是说，在并发编程中，即使每个操作都能保证原子性，也不能保证组合操作的原子性。**

### 坑二：使用迭代器遍历容器

**一个容易被人忽略的坑就是使用迭代器遍历容器，对容器中的每个元素调用一个方法，这就存在了并发问题，这些组合操作不具备原子性。**

例如下面的代码，通过迭代器遍历同步List，对List集合中的每个元素调用format()方法。

```java
List list = Collections.synchronizedList(new ArrayList());
Iterator iterator = list.iterator(); 
while (iterator.hasNext()){
    format(iterator.next());
}
```

此时，会存在并发问题，这些组合操作并不具备原子性。

如何解决这个问题呢？一个很简单的方式就是锁住list集合，如下所示。

```java
List list = Collections.synchronizedList(new ArrayList());
synchronized(list){
    Iterator iterator = list.iterator(); 
    while (iterator.hasNext()){
         format(iterator.next()); 
    }
}
```

**这里，为何锁住list集合就能够解决并发问题呢？**

这是因为在Collections类中，其内部的包装类的公共方法锁住的对象是this，其实就是上面代码中的list，所以，我们对list加锁后，就能够保证线程的安全性了。

在Java中，同步容器一般都是基于synchronized锁实现的，有些是通过包装类实现的，例如List、Set、Map等。有些不是通过包装类实现的，例如Vector、Stack、HashTable等。

**对于这些容器的遍历操作，一定要为容器添加互斥锁保证整体的原子性。**

## 写在最后

**如果你想进大厂，想升职加薪，或者对自己现有的工作比较迷茫，都可以私信我交流，希望我的一些经历能够帮助到大家~~**

**推荐阅读：**
* 《[从零到上亿用户，我是如何一步步优化MySQL数据库的？（建议收藏）](https://blog.csdn.net/l1028386804/article/details/119793397)》
* 《[我用多线程进一步优化了亿级流量电商业务下的海量数据校对系统，性能再次提升了200%！！（全程干货，建议收藏）](https://blog.csdn.net/l1028386804/article/details/119724650)》
* 《[我用多线程优化了亿级流量电商业务下的海量数据校对系统，性能直接提升了200%！！（全程干货，建议收藏）](https://blog.csdn.net/l1028386804/article/details/119632011)》
* 《[我用10张图总结出了这份并发编程最佳学习路线！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/119529877)》
* 《[高并发场景下一种比读写锁更快的锁，看完我彻底折服了！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/118887662)》
* 《[全网最全性能优化总结！！（冰河吐血整理，建议收藏）](https://blog.csdn.net/l1028386804/article/details/118175154)》
* 《[三天撸完了MyBatis，各位随便问！！（冰河吐血整理，建议收藏）](https://blog.csdn.net/l1028386804/article/details/118079011)》
* 《[奉劝那些刚参加工作的学弟学妹们：要想进大厂，这些并发编程知识是你必须要掌握的！完整学习路线！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/119341751)》
* 《[奉劝那些刚参加工作的学弟学妹们：要想进大厂，这些核心技能是你必须要掌握的！完整学习路线！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/116081409)》
* 《[奉劝那些刚参加工作的学弟学妹们：这些计算机与操作系统基础知识越早知道越好！万字长文太顶了！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/115923034)》
* 《[我用三天时间开发了一款老少皆宜的国民级游戏，支持播放音乐，现开放完整源代码和注释（建议收藏）！！](https://blog.csdn.net/l1028386804/article/details/116191713)》
* 《[我是全网最硬核的高并发编程作者，CSDN最值得关注的博主，大家同意吗？（建议收藏）](https://blog.csdn.net/l1028386804/article/details/116280491)》
* 《[毕业五年，从月薪3000到年薪百万，我掌握了哪些核心技能？（建议收藏）](https://blog.csdn.net/l1028386804/article/details/115677451)》
* 《[我入侵了隔壁妹子的Wifi，发现。。。（全程实战干货，建议收藏）](https://blog.csdn.net/l1028386804/article/details/116477701)》
* 《[千万不要轻易尝试“熊猫烧香”，这不，我后悔了！](https://blog.csdn.net/l1028386804/article/details/115450665)》
* 《[清明节偷偷训练“熊猫烧香”，结果我的电脑为熊猫“献身了”！](https://blog.csdn.net/l1028386804/article/details/115607742)》
* 《[7.3万字肝爆Java8新特性，我不信你能看完！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/115547910)》
* 《[在业务高峰期拔掉服务器电源是一种怎样的体验？](https://blog.csdn.net/l1028386804/article/details/115576102)》
* 《[全网最全Linux命令总结！！（史上最全，建议收藏）](https://blog.csdn.net/l1028386804/article/details/117917710)》
* 《[用Python写了个工具，完美破解了MySQL！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/118378477)》
* 《[SimpleDateFormat类到底为啥不是线程安全的？（附六种解决方案，建议收藏）](https://blog.csdn.net/l1028386804/article/details/118783851)》
* 《[MySQL 8中新增的这三大索引，直接让MySQL起飞了，你竟然还不知道！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/119442194)》
* 《[撸完Spring源码，我开源了这个分布式缓存框架！！（建议收藏）](https://blog.csdn.net/l1028386804/article/details/119861563)》

## 写在最后

> 如果你觉得冰河写的还不错，请微信搜索并关注「 **冰河技术** 」微信公众号，跟冰河学习高并发、分布式、微服务、大数据、互联网和云原生技术，「 **冰河技术** 」微信公众号更新了大量技术专题，每一篇技术文章干货满满！不少读者已经通过阅读「 **冰河技术** 」微信公众号文章，吊打面试官，成功跳槽到大厂；也有不少读者实现了技术上的飞跃，成为公司的技术骨干！如果你也想像他们一样提升自己的能力，实现技术能力的飞跃，进大厂，升职加薪，那就关注「 **冰河技术** 」微信公众号吧，每天更新超硬核技术干货，让你对如何提升技术能力不再迷茫！


![](https://img-blog.csdnimg.cn/20200906013715889.png)