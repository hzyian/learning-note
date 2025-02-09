# Java并发应用

## 生产者与消费者模型


### synchronize 

#### 基于synchronize 方法

```
    public synchronized void produce() throws InterruptedException {
        while (present) {
            wait();
        }
        System.out.println(Thread.currentThread() + " Producer produce meal " + count);
        count++;
        this.setPresent(true);
        notifyAll();
    }

    public synchronized void consume() throws InterruptedException {
        while (!present) {
            wait();
        }
        System.out.println(Thread.currentThread() + " consumer present meal" + count);
        this.setPresent(false);
        notifyAll();

    }
```

#### 使用synchronize 锁对象

```
    public void createByObject() throws InterruptedException {
        synchronized (this) {
            while (present) {
                wait();
            }
        }
        count++;
        System.out.println(Thread.currentThread() + " Producer produce meal " + count);
        this.setPresent(true);
        synchronized (this) {
            notifyAll();
        }
    }

    public void consumerByObject() throws InterruptedException {
        synchronized (this) {
            while (!present) {
                wait();
            }
        }
        System.out.println(Thread.currentThread() + " consumer present meal " + count);
        this.setPresent(false);
        synchronized (this) {
            notifyAll();
        }
    }
```

- 更进一步细分锁粒度，区分生产者与消费者对象。


#### 问题代码
```
    public void createByObject() throws InterruptedException {
        while (present) {
            // 轻量级锁等待的时候，如果另个线程先获取锁，并notifyAll
            // 那么可能两个线程都进入wait状态
            synchronized (this) {
                wait();
            }
        }
        count++;
        System.out.println(Thread.currentThread() + " Producer produce meal " + count);
        this.setPresent(true);
        synchronized (this) {
            notifyAll();
        }

    }

    public void consumerByObject() throws InterruptedException {
        while (!present) {
            // 轻量级锁等待的时候，如果另个线程先获取锁，并notifyAll
            // 那么可能两个线程都进入wait状态
            synchronized (this) {
                wait();
            }
        }
        System.out.println(Thread.currentThread() + " consumer present meal " + count);
        this.setPresent(false);
        synchronized (this) {
            notifyAll();
        }
    }

    // 正确做法 将synchronize放循环外
   public void createByObject() throws InterruptedException {
        synchronized (this) {
            while (present) {
                wait();
            }
        }
        ... 
    }
```


### 基于ReentrantLock 结合 condition
- 使用condition作为等待队列
```
class Consumer implements Runnable {

    private ReentrantLock lock;

    private Condition condition;

    public Consumer(ReentrantLock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {

        try {
            while (!Thread.interrupted()) {
                try {
                    lock.lock();
                    while (!ProConDemo.flag){
                        condition.await();
                    }
                    System.out.println( Thread.currentThread()+ " consumer shout !!!!");
                    ProConDemo.flag = false;
                    condition.signalAll();
                }
                finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Producer implements Runnable {

    private ReentrantLock lock;

    private Condition condition;

    public Producer(ReentrantLock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                try {
                    lock.lock();
                    while (ProConDemo.flag){
                        condition.await();
                    }
                    System.out.println( Thread.currentThread()+ " producer shout~~~~");
                    ProConDemo.flag = true;
                    condition.signalAll();
                }
                finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

- 进阶：细分生产者队列与消费者队列

### 基于BlockingQueue
使用阻塞队列实现生产者与消费者模型
- 消费者：`queue.take();`
- 生产者：`queue.put(obj)`

## 多线程顺序输出
### 基于Synchronize锁对象
```
class Thread1 implements Runnable {

    private Object obj;

    public Thread1(Object obj) {
        this.obj = obj;
    }

    @Override
    public void run() {

        try {
            while (!Thread.interrupted()) {
                synchronized (obj) {
                    while (!SynchronizeObject.flag) {
                        obj.wait();
                    }
                    System.out.println(Thread.currentThread() + " this is thread1");
                    SynchronizeObject.flag = false;
                    obj.notify();
                }
            }

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Thread2 implements Runnable {

    private Object obj;

    public Thread2(Object obj) {
        this.obj = obj;
    }

    @Override
    public void run() {

        try {
            while (!Thread.interrupted()) {
                synchronized (obj) {
                    while (SynchronizeObject.flag) {
                        obj.wait();
                    }
                    System.out.println(Thread.currentThread() + " this is thread2~~");
                    SynchronizeObject.flag = true;
                    obj.notify();
                }
            }

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```


### 基于Reentrant Lock 
方法1：设立一个flag，防止非公平锁抢锁输出，导致的顺序混乱问题。

方法2： 使用Reentrant Lock 公平锁
```

ReentrantLock lock = new ReentrantLock(true);

    public void run() {

        while (!Thread.interrupted()) {
            try {
                lock.lock();
                System.out.println(Thread.currentThread() + " consumer shout !!!!");
            } finally {
                lock.unlock();
            }
        }
    }
```


## 线程安全的类定义
1. 无状态的类：没有任何成员变量的类，如无任何方法的枚举类型。
2. 让类不可变	
   1. 加final关键字
   2. 不提供修改成员变量，也不提供获取成员变量方法
3. 使用volatile，保证类的可见性，不能保证线程安全。适合一写多读的场景
4. 加锁和CAS，使用synchronized、lock、原子变量AtomicInteger等
   1. 如StringBuffer 修改的方法都使用synchronize修饰。
   2. 如concurrentHashMap 使用自旋加CAS修改。
   3. 使用Atomic包的基本类型，如AtomicInteger、AtomicReference、AtmoicStampReference修饰变量。
   

### 枚举类为什么是线程安全？

普通的一个枚举类
```
public enum t {
    SPRING,SUMMER,AUTUMN,WINTER;
}
```

反编译后的代码
```
public final class T extends Enum
{
    private T(String s, int i)
    {
        super(s, i);
    }
    public static T[] values()
    {
        T at[];
        int i;
        T at1[];
        System.arraycopy(at = ENUM$VALUES, 0, at1 = new T[i = at.length], 0, i);
        return at1;
    }

    public static T valueOf(String s)
    {
        return (T)Enum.valueOf(demo/T, s);
    }

    public static final T SPRING;
    public static final T SUMMER;
    public static final T AUTUMN;
    public static final T WINTER;
    private static final T ENUM$VALUES[];
    static
    {
        SPRING = new T("SPRING", 0);
        SUMMER = new T("SUMMER", 1);
        AUTUMN = new T("AUTUMN", 2);
        WINTER = new T("WINTER", 3);
        ENUM$VALUES = (new T[] {
            SPRING, SUMMER, AUTUMN, WINTER
        });
    }
}
```

1. `public final class T extends Enum`，说明，该类是继承了Enum类的，同时final关键字告诉我们，这个类也是不能被继承的。
2. 类中的几个属性和方法都是static final类型的，说明static类型的属性会在类被加载之后被初始化便不可修改。
> 创建一个enum类型是线程安全的。


- 相关资料：https://www.cnblogs.com/z00377750/p/9177097.html

## 单订单重复退款请求
1. synchronize修饰退款方法。 
2. 缩小synchronize锁范围，使用对象锁。对象锁，创建弱引用的一个订单ID对象，放到同一的锁对象资源池中。
   - 清理锁对象可以使用守护线程的方法，基于Unsafe的包操作去清除。
3. 分布式应用，使用分布式锁来处理。


### 分布式锁的处理方案

1. 数据库锁，数据库乐观锁，数据库悲观锁。

2. redis 锁 或者 ZooKeeper锁

3. 使用消息队列顺序消费，保证不重复退款

## 消息批量发送设计
### 问题场景
某个活动需要对平台的客户进行短信的推销发送，假设对平台的10w用户推送某个活动。推送的用户数据由数据仓库已经推送到表t_user_promotion总共10w条数据。

而在调用短信批量发送服务的时候，经常有限制批量发送的手机号数目的，比如限制100个手机号。
1. 如何对10w条消息进行发送
2. 假设推送由运营人员进行触发，如何防止10w条消息出现重复触发的情况。

进阶问题：假如10w条消息，消息模板是不一致的，如何设计打批次的逻辑

设计点：
1. 表结构设计
2. 查询效率问题

### 解决方法
方法1：
设计一张中间表作为批次表，每100个用户打一个批次。用户表中增加批次id。

整体发送的步骤如下：
1. 用户表增加批次号字段，每100个号码打成一个批次，记录插入批次表。
2. 批次表有线程进行短信发送的调用，发送成功更新批次表状态。
   好处为：短信发送与数据处理逻辑分离。

方法2：
1. 用户表使用状态代表发送和未发送。

---

问题点：
该问题的难点在于如何处理多线程的分工及互斥问题。多线程三大核心问题：分工、协作、互斥。

- 分工：每个批量发送的接口只能固定发送100个手机号，因此每个线程负责100条记录的处理。
- 协作：该场景的线程协作，只用通知短信发送的主线线程这个数据处理完了。比如countDownLaunch，自线程完成之后countDown
- 互斥：每个线程负责处理100条数据，如果默认limit 100，那么必然会有互斥问题。

解决分工及互斥几种方式：
1. 避免互斥。
   1. 如果取10个线程，那么每个线程可以默认取主键id%10=y的记录进行处理，避免了互斥。
   2. 查询出所需要处理的记录，使用id排序，每个线程根据pageIndex+pageNo，进行数据的处理。
      - > sql可能会因为pageIndex+pageNo 导致慢查询，需要使用延迟关联的方式进行sql优化
2. 加锁
   1. 悲观锁，使用mysql的行锁。
   2. 使用mysql的version乐观锁，会造成大量的乐观锁更新失败问题，不建议使用。

### 进阶问题处理
1. 如何解决重复发送问题，短信批量发送的请求更新为中间状态发送中。
2. 消息模版不一致问题，使用数据分组，相同消息模版的数据统一处理。

### 相关类似资料
[批量任务体现多线程的威力！](https://juejin.cn/post/6844903774234869774)