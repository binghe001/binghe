## 高并发场景下一种比读写锁更快的锁，看完我彻底折服了！！（建议收藏）

**大家好，我是冰河~~**

最近公司的事情比较多，拖了很久的书稿终于和猫大人一起在这个周末写完了，总体就一个字：累。剩下的就是对稿件的修修补补了，后面的进度就应该会很快了。前段时间原本想着续更【[精通高并发系列](https://blog.csdn.net/l1028386804/category_9719933.html)】专题，一直没时间，所以，这个事情也被一直搁浅着。现在，书稿写完了，就有一些时间续更这个专题了。之前，我把【[精通高并发系列](https://blog.csdn.net/l1028386804/category_9719933.html)】专题的文章整理成了一本电子书——《深入理解高并发编程》，全书内容如下所示。

![](https://img-blog.csdnimg.cn/20210327002830968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2wxMDI4Mzg2ODA0,size_16,color_FFFFFF,t_70#pic_center)

这本电子书全网一度下载高达 **35W+**，目前这个数字仍然在上升，如果你还没下载的话，那就在 **冰河技术** 公号 回复 **并发编程** 领取下载链接吧。

从今天开始，我们继续更新【[精通高并发系列](https://blog.csdn.net/l1028386804/category_9719933.html)】专题。今天为大家介绍一个在高并发环境下，比读写锁性能更高的锁。可能很多小伙伴都不知道StampedLock是啥，至少我身边的很多小伙伴都没使用过StampedLock锁，今天，我们就一起来聊聊这个在高并发环境下比ReadWriteLock更快的锁——StampedLock。

## 什么是StampedLock?

ReadWriteLock锁允许多个线程同时读取共享变量，但是在读取共享变量的时候，不允许另外的线程多共享变量进行写操作，更多的适合于读多写少的环境中。那么，**在读多写少的环境中，有没有一种比ReadWriteLock更快的锁呢？**

**答案当然是有！那就是我们今天要介绍的主角——JDK1.8中新增的StampedLock！没错，就是它！**

StampedLock与ReadWriteLock相比，在读的过程中也允许后面的一个线程获取写锁对共享变量进行写操作，为了避免读取的数据不一致，使用StampedLock读取共享变量时，需要对共享变量进行是否有写入的检验操作，并且这种读是一种乐观读。

**总之，StampedLock是一种在读取共享变量的过程中，允许后面的一个线程获取写锁对共享变量进行写操作，使用乐观读避免数据不一致的问题，并且在读多写少的高并发环境下，比ReadWriteLock更快的一种锁。**

## StampedLock三种锁模式

这里，我们可以简单对比下StampedLock与ReadWriteLock，ReadWriteLock支持两种锁模式：**一种是读锁，另一种是写锁**，并且ReadWriteLock允许多个线程同时读共享变量，在读时，不允许写，在写时，不允许读，读和写是互斥的，所以，ReadWriteLock中的读锁，更多的是指悲观读锁。

StampedLock支持三种锁模式：**写锁、读锁**（这里的读锁指的是悲观读锁）**和乐观读**（很多资料和书籍写的是乐观读锁，这里我个人觉得更准确的是乐观读，为啥呢？我们继续往下看啊）。其中，写锁和读锁与ReadWriteLock中的语义类似，允许多个线程同时获取读锁，但是只允许一个线程获取写锁，写锁和读锁也是互斥的。

另一个与ReadWriteLock不同的地方在于：StampedLock在获取读锁或者写锁成功后，都会返回一个Long类型的变量，之后在释放锁时，需要传入这个Long类型的变量。例如，下面的伪代码所示的逻辑演示了StampedLock如何获取锁和释放锁。

```java
public class StampedLockDemo{
    //创建StampedLock锁对象
    public StampedLock stampedLock = new StampedLock();
    
    //获取、释放读锁
    public void testGetAndReleaseReadLock(){
        long stamp = stampedLock.readLock();
        try{
            //执行获取读锁后的业务逻辑
        }finally{
            //释放锁
            stampedLock.unlockRead(stamp);
        }
    }
    
    //获取、释放写锁
    public void testGetAndReleaseWriteLock(){
        long stamp = stampedLock.writeLock();
        try{
            //执行获取写锁后的业务逻辑。
        }finally{
            //释放锁
            stampedLock.unlockWrite(stamp);
        }
    }
}
```

**StampedLock支持乐观读，这是它比ReadWriteLock性能要好的关键所在。** ReadWriteLock在读取共享变量时，所有对共享变量的写操作都会被阻塞。而StampedLock提供的乐观读，在多个线程读取共享变量时，允许一个线程对共享变量进行写操作。

我们再来看一下JDK官方给出的StampedLock示例，如下所示。

```java
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    void move(double deltaX, double deltaY) { // an exclusively locked method
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    double distanceFromOrigin() { // A read-only method
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        if (!sl.validate(stamp)) {
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    void moveIfAtOrigin(double newX, double newY) { // upgrade
        // Could instead start with optimistic, not read mode
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX;
                    y = newY;
                    break;
                }
                else {
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
}
```

在上述代码中，如果在执行乐观读操作时，另外的线程对共享变量进行了写操作，则会把乐观读升级为悲观读锁，如下代码片段所示。

```java
double distanceFromOrigin() { // A read-only method
    //乐观读
    long stamp = sl.tryOptimisticRead();
    double currentX = x, currentY = y;
    //判断是否有线程对变量进行了写操作
    //如果有线程对共享变量进行了写操作
    //则sl.validate(stamp)会返回false
    if (!sl.validate(stamp)) {
        //将乐观读升级为悲观读锁
        stamp = sl.readLock();
        try {
            currentX = x;
            currentY = y;
        } finally {
            //释放悲观锁
            sl.unlockRead(stamp);
        }
    }
    return Math.sqrt(currentX * currentX + currentY * currentY);
}
```

这种将乐观读升级为悲观读锁的方式相比一直使用乐观读的方式更加合理，如果不升级为悲观读锁，则程序会在一个循环中反复执行乐观读操作，直到乐观读操作期间没有线程执行写操作，而在循环中不断的执行乐观读会消耗大量的CPU资源，升级为悲观读锁是更加合理的一种方式。

## StampedLock实现思想

StampedLock内部是基于CLH锁实现的，CLH是一种自旋锁，能够保证没有“饥饿现象”的发生，并且能够保证FIFO（先进先出）的服务顺序。

在CLH中，锁维护一个等待线程队列，所有申请锁，但是没有成功的线程都会存入这个队列中，每一个节点代表一个线程，保存一个标记位(locked)，用于判断当前线程是否已经释放锁，当locked标记位为true时， 表示获取到锁，当locked标记位为false时，表示成功释放了锁。

当一个线程试图获得锁时，取得等待队列的尾部节点作为其前序节点，并使用类似如下代码判断前序节点是否已经成功释放锁:

```java
while (pred.locked) {
    //省略操作 
}
```

只要前序节点(pred)没有释放锁，则表示当前线程还不能继续执行，因此会自旋等待；反之，如果前序线程已经释放锁，则当前线程可以继续执行。

释放锁时，也遵循这个逻辑，线程会将自身节点的locked位置标记为false，后续等待的线程就能继续执行了，也就是已经释放了锁。

StampedLock的实现思想总体来说，还是比较简单的，这里就不展开讲了。

## StampedLock的注意事项

在读多写少的高并发环境下，StampedLock的性能确实不错，但是它不能够完全取代ReadWriteLock。在使用的时候，也需要特别注意以下几个方面。

**StampedLock不支持重入**

没错，StampedLock是不支持重入的，也就是说，在使用StampedLock时，不能嵌套使用，这点在使用时要特别注意。

**StampedLock不支持条件变量**

第二个需要注意的是就是StampedLock不支持条件变量，无论是读锁还是写锁，都不支持条件变量。

**StampedLock使用不当会导致CPU飙升**

这点也是最重要的一点，在使用时需要特别注意：如果某个线程阻塞在StampedLock的readLock()或者writeLock()方法上时，此时调用阻塞线程的interrupt()方法中断线程，会导致CPU飙升到100%。例如，下面的代码所示。

```java
public void testStampedLock() throws Exception{
    final StampedLock lock = new StampedLock();
    Thread thread01 = new Thread(()->{
        // 获取写锁
        lock.writeLock();
        // 永远阻塞在此处，不释放写锁
        LockSupport.park();
    });
    thread01.start();
    // 保证thread01获取写锁
    Thread.sleep(100);
    Thread thread02 = new Thread(()->
                           //阻塞在悲观读锁
                           lock.readLock()
                          );
    thread02.start();
    // 保证T2阻塞在读锁
    Thread.sleep(100);
    //中断线程thread02
    //会导致线程thread02所在CPU飙升
    thread02.interrupt();
    thread02.join();
}
```

运行上面的程序，会导致thread02线程所在的CPU飙升到100%。

这里，有很多小伙伴不太明白为啥` LockSupport.park();`会导致thread01会永远阻塞。这里，冰河为你画了一张线程的生命周期图，如下所示。

![](https://img-blog.csdnimg.cn/20200215004335203.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2wxMDI4Mzg2ODA0,size_16,color_FFFFFF,t_70#pic_center)

这下明白了吧？在线程的生命周期中，有几个重要的状态需要说明一下。

- NEW：初始状态，线程被构建，但是还没有调用start()方法。
- RUNNABLE：可运行状态，可运行状态可以包括：运行中状态和就绪状态。
- BLOCKED：阻塞状态，处于这个状态的线程需要等待其他线程释放锁或者等待进入synchronized。
- WAITING：表示等待状态，处于该状态的线程需要等待其他线程对其进行通知或中断等操作，进而进入下一个状态。
- TIME_WAITING：超时等待状态。可以在一定的时间自行返回。
- TERMINATED：终止状态，当前线程执行完毕。

看完这个线程的生命周期图，知道为啥调用` LockSupport.park();`会使thread02阻塞了吧？

**所以，在使用StampedLock时，一定要注意避免线程所在的CPU飙升的问题。那如何避免呢？**

那就是使用StampedLock的readLock()方法或者读锁和使用writeLock()方法获取写锁时，一定不要调用线程的中断方法来中断线程，如果不可避免的要中断线程的话，一定要用StampedLock的readLockInterruptibly()方法获取可中断的读锁和使用StampedLock的writeLockInterruptibly()方法获取可中断的悲观写锁。

最后，对于StampedLock的使用，JDK官方给出的StampedLock示例本身就是一个最佳实践了，小伙伴们可以多看看JDK官方给出的StampedLock示例，多多体会下StampedLock的使用方式和背后原理与核心思想。

## 写在最后

**如果你想进大厂，想升职加薪，或者对自己现有的工作比较迷茫，都可以私信我交流，希望我的一些经历能够帮助到大家~~**

**推荐阅读：**

* 《[全网最全性能优化总结！！（冰河吐血整理，建议收藏）](https://blog.csdn.net/l1028386804/article/details/118175154)》
* 《[三天撸完了MyBatis，各位随便问！！（冰河吐血整理，建议收藏）](https://blog.csdn.net/l1028386804/article/details/118079011)》
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
* 《[SimpleDateFormat类到底为啥不是线程安全的？（附六种解决方案，建议收藏）](https://binghe.blog.csdn.net/article/details/118783851)》

**好了，今天就到这儿吧，小伙伴们点赞、收藏、评论，一键三连走起呀，我是冰河，我们下期见~~**

![](https://img-blog.csdnimg.cn/20210522113750942.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2wxMDI4Mzg2ODA0,size_16,color_FFFFFF,t_70#pic_center)


## 写在最后

> 如果你觉得冰河写的还不错，请微信搜索并关注「 **冰河技术** 」微信公众号，跟冰河学习高并发、分布式、微服务、大数据、互联网和云原生技术，「 **冰河技术** 」微信公众号更新了大量技术专题，每一篇技术文章干货满满！不少读者已经通过阅读「 **冰河技术** 」微信公众号文章，吊打面试官，成功跳槽到大厂；也有不少读者实现了技术上的飞跃，成为公司的技术骨干！如果你也想像他们一样提升自己的能力，实现技术能力的飞跃，进大厂，升职加薪，那就关注「 **冰河技术** 」微信公众号吧，每天更新超硬核技术干货，让你对如何提升技术能力不再迷茫！


![](https://img-blog.csdnimg.cn/20200906013715889.png)