## 【高并发】你知道吗？大家都在使用Redisson实现分布式锁了！！

## 写在前面

> 忘记之前在哪个群里有朋友在问：有出分布式锁的文章吗～@冰河？我的回答是：这周会有，也是【高并发】专题的。想了想，还是先发一个如何使用Redisson实现分布式锁的文章吧？为啥？因为使用Redisson实现分布式锁简单啊！Redisson框架是基于Redis实现的分布式锁，非常强大，只需要拿来使用就行了，至于分布式锁的原理啥的，后面再撸一篇文章就是了。

Redisson框架十分强大，基于Redisson框架可以实现几乎你能想到的所有类型的分布式锁。这里，我就列举几个类型的分布式锁，并各自给出一个示例程序来加深大家的理解。有关分布式锁的原理细节，后续专门撸一篇文章咱们慢慢聊！

## 1.可重入锁（Reentrant Lock）

Redisson的分布式可重入锁RLock Java对象实现了java.util.concurrent.locks.Lock接口，同时还支持自动过期解锁。

```java
public void testReentrantLock(RedissonClient redisson){
	RLock lock = redisson.getLock("anyLock");
	try{
		// 1. 最常见的使用方法
		//lock.lock();
		// 2. 支持过期解锁功能,10秒钟以后自动解锁, 无需调用unlock方法手动解锁
		//lock.lock(10, TimeUnit.SECONDS);
		// 3. 尝试加锁，最多等待3秒，上锁以后10秒自动解锁
		boolean res = lock.tryLock(3, 10, TimeUnit.SECONDS);
		if(res){ //成功
		// do your business
		}
	} catch (InterruptedException e) {
		e.printStackTrace();
	} finally {
		lock.unlock();
	}
}
```

Redisson同时还为分布式锁提供了异步执行的相关方法：

```java
public void testAsyncReentrantLock(RedissonClient redisson){
	RLock lock = redisson.getLock("anyLock");
	try{
		lock.lockAsync();
		lock.lockAsync(10, TimeUnit.SECONDS);
		Future<Boolean> res = lock.tryLockAsync(3, 10, TimeUnit.SECONDS);
		if(res.get()){
		// do your business
		}
	} catch (InterruptedException e) {
		e.printStackTrace();
	} catch (ExecutionException e) {
		e.printStackTrace();
	} finally {
		lock.unlock();
	}
}
```

## 2.公平锁（Fair Lock）

Redisson分布式可重入公平锁也是实现了java.util.concurrent.locks.Lock接口的一种RLock对象。在提供了自动过期解锁功能的同时，保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。

```java
public void testFairLock(RedissonClient redisson){
	RLock fairLock = redisson.getFairLock("anyLock");
	try{
		// 最常见的使用方法
		fairLock.lock();
		// 支持过期解锁功能, 10秒钟以后自动解锁,无需调用unlock方法手动解锁
		fairLock.lock(10, TimeUnit.SECONDS);
		// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
		boolean res = fairLock.tryLock(100, 10, TimeUnit.SECONDS);
	} catch (InterruptedException e) {
		e.printStackTrace();
	} finally {
		fairLock.unlock();
	}
}
```

Redisson同时还为分布式可重入公平锁提供了异步执行的相关方法：

```java
RLock fairLock = redisson.getFairLock("anyLock");
fairLock.lockAsync();
fairLock.lockAsync(10, TimeUnit.SECONDS);
Future<Boolean> res = fairLock.tryLockAsync(100, 10, TimeUnit.SECONDS);
```

## 3.联锁（MultiLock）

Redisson的RedissonMultiLock对象可以将多个RLock对象关联为一个联锁，每个RLock对象实例可以来自于不同的Redisson实例。

```java
public void testMultiLock(RedissonClient redisson1,RedissonClient redisson2, RedissonClient redisson3){
	RLock lock1 = redisson1.getLock("lock1");
	RLock lock2 = redisson2.getLock("lock2");
	RLock lock3 = redisson3.getLock("lock3");
	RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
	try {
		// 同时加锁：lock1 lock2 lock3, 所有的锁都上锁成功才算成功。
		lock.lock();
		// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
		boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
	} catch (InterruptedException e) {
		e.printStackTrace();
	} finally {
		lock.unlock();
	}
}
```

## 4.红锁（RedLock）

Redisson的RedissonRedLock对象实现了Redlock介绍的加锁算法。该对象也可以用来将多个RLock对象关联为一个红锁，每个RLock对象实例可以来自于不同的Redisson实例。

```java
public void testRedLock(RedissonClient redisson1,RedissonClient redisson2, RedissonClient redisson3){
	RLock lock1 = redisson1.getLock("lock1");
	RLock lock2 = redisson2.getLock("lock2");
	RLock lock3 = redisson3.getLock("lock3");
	RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
	try {
		// 同时加锁：lock1 lock2 lock3, 红锁在大部分节点上加锁成功就算成功。
		lock.lock();
		// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
		boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
	} catch (InterruptedException e) {
		e.printStackTrace();
	} finally {
		lock.unlock();
	}
}
```

## 5.读写锁（ReadWriteLock）

Redisson的分布式可重入读写锁RReadWriteLock,Java对象实现了java.util.concurrent.locks.ReadWriteLock接口。同时还支持自动过期解锁。该对象允许同时有多个读取锁，但是最多只能有一个写入锁。

```java
RReadWriteLock rwlock = redisson.getLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();
// 支持过期解锁功能
// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);
// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

## 6.信号量（Semaphore）

Redisson的分布式信号量（Semaphore）Java对象RSemaphore采用了与java.util.concurrent.Semaphore相似的接口和用法。

```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.acquire();
//或
semaphore.acquireAsync();
semaphore.acquire(23);
semaphore.tryAcquire();
//或
semaphore.tryAcquireAsync();
semaphore.tryAcquire(23, TimeUnit.SECONDS);
//或
semaphore.tryAcquireAsync(23, TimeUnit.SECONDS);
semaphore.release(10);
semaphore.release();
//或
semaphore.releaseAsync();
```

## 7.可过期性信号量（PermitExpirableSemaphore）

Redisson的可过期性信号量（PermitExpirableSemaphore）实在RSemaphore对象的基础上，为每个信号增加了一个过期时间。每个信号可以通过独立的ID来辨识，释放时只能通过提交这个ID才能释放。

```java
RPermitExpirableSemaphore semaphore = redisson.getPermitExpirableSemaphore("mySemaphore");
String permitId = semaphore.acquire();
// 获取一个信号，有效期只有2秒钟。
String permitId = semaphore.acquire(2, TimeUnit.SECONDS);
// ...
semaphore.release(permitId);
```

## 8.闭锁（CountDownLatch）

Redisson的分布式闭锁（CountDownLatch）Java对象RCountDownLatch采用了与java.util.concurrent.CountDownLatch相似的接口和用法。

```java
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.trySetCount(1);
latch.await();
// 在其他线程或其他JVM里
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.countDown();
```

## 写在最后

> 如果觉得文章对你有点帮助，请微信搜索并关注「 **冰河技术** 」微信公众号，跟冰河学习高并发编程技术。

最后，附上并发编程需要掌握的核心技能知识图，祝大家在学习并发编程时，少走弯路。

![sandahexin_20200322](images/sandahexin_20200322.jpg)