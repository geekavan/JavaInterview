# Java程序员的自我修养_乱序版

推荐关注(排名不分先后)：

1.程序员小灰

2.马士兵教育

3.毕向东视频(优点：讲课幽默风趣，一惊一乍，娓娓道来；缺点：jdk版本陈旧，线程池未讲，java8新特性未讲，讲了一些没必要学的内容，比如swing等等)

注意：

1.本武功心法中有很多项目相关的东西，如果没有做过可能看起来比较吃力，建议略过

2.本武功心法建议以字典的形式来进行题对答案的查找，而不是从头到尾的观看

3.本武功心法答案不一定正确，请酌情使用练习

## CAS的原理是什么？

答：jdk中的cas机制是通过调用JNI的代码进行实现的，java native interface 即java允许程序调用其他语言实现的代码，cas底层就是c语言调用cpu指令进行实现的

就Intelx86来说，最终映射为cmpxchg指令，该指令进行cas的时候会在**总线上加锁**，这样就解决了多线程下的问题

## JDK中对于CAS的支持？

答：在sun.misc.unsafe类下，包含CompareAndSwap***系列的方法，包括

```java
@ForceInline
public final boolean compareAndSwapObject(Object o, long offset,
                                              Object expected,
                                              Object x) {
        return theInternalUnsafe.compareAndSetReference(o, offset, expected, x);
    }
@ForceInline
public final boolean compareAndSwapInt(Object o, long offset,
                                           int expected,
                                           int x) {
        return theInternalUnsafe.compareAndSetInt(o, offset, expected, x);
    }
@ForceInline
public final boolean compareAndSwapLong(Object o, long offset,
                                            long expected,
                                            long x) {
        return theInternalUnsafe.compareAndSetLong(o, offset, expected, x);
    }
```

其中o表示对象，offset表示对象的的偏移地址，expected表示期望值，x表示要赋予的新值

## CAS的问题？

1.ABA问题

2.多线程情况下存在效率问题

比如多个线程同时通过cas手段进行变量的赋值，但是某一个时刻只能有一个线程进行赋值，该线程赋值之后，其他线程先前获取的期望值就会是旧值，无法赋值成功，需要重新获取期望值并尝试进行赋值，这种情况下一个线程一个线程来工作就会产生性能问题，具体例子如下

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.LongAdder;

public class Main {
    private static AtomicInteger count = new AtomicInteger(0);
    private static LongAdder newCount = new LongAdder();
    public static void main(String[] args) throws InterruptedException {
        int threadCount = 100;
        int incrementCount = 10000000;
        testAtomicInteger(threadCount, incrementCount);
        testAtomicLongAdder(threadCount, incrementCount);
    }

    private static void testAtomicLongAdder(int threadCount, int incrementCount) throws InterruptedException {
        long startMillis = System.currentTimeMillis();
        CountDownLatch countDownLatch = new CountDownLatch(threadCount);
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        for (int j = 0; j < incrementCount; j++) {
                            newCount.increment();
                        }
                    }finally {
                        countDownLatch.countDown();
                    }
                }
            }, "LongAdder");
            thread.start();
        }
        countDownLatch.await();
        long endMillis = System.currentTimeMillis();
        System.out.println((endMillis-startMillis)+"...LongAdder..."+newCount);
    }

    public static void testAtomicInteger(int threadCount, int incrementCount) throws InterruptedException {
        long startMillis = System.currentTimeMillis();
        CountDownLatch countDownLatch = new CountDownLatch(threadCount);
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        for (int j = 0; j < incrementCount; j++) {
                            count.incrementAndGet();
                        }
                    }
                    finally {
                        countDownLatch.countDown();
                    }
                }
            }, "AtomicInteger");
            thread.start();
        }
        countDownLatch.await();
        long endMillis = System.currentTimeMillis();
        System.out.println((endMillis-startMillis)+"...AtomicInteger..."+count);
    }
}
/*
32693...AtomicInteger...1000000000
5905...LongAdder...1000000000
*/
```

我们看到就原子类的自加操作来说，如果开启100个线程，每个线程执行1000万次的自加操作，博主自己的计算机跑的毫秒数分别为32693和5905，我们看到性能已经不是同一个数量级了，LongAdder要更好一些，从这个例子中我们可以看出cas缺点的一些端倪

## Java动态代理

1.java中的动态代理有两种方式，一种是jdk动态代理，一种是cglib动态代理

2.jdk动态代理要求被代理类必须实现一个接口，这样的话代理类就可以通过实现这个接口来达到和被代理类的功能一致，用户想要调用被代理类的某些方法的时候就可以直接调用这个代理类的对应方法，并在代理类中做一些操作，以达到代码增强的效果，而cglib的话他是通过继承被代理类而达到和被代理类功能一致的

3.jdk动态代理中：

```java
public Object getProxyObject(){
    return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);//其中this是实现InvocationHandler的中间类Z
}
```

我们通过Proxy.newProxyInstance()方法来构建代理类我们称之为类D(拼音开头)，但是在执行这个方法之前，我们需要告诉它我们要构建的被代理类的类加载器是谁，被代理类的而接口是哪个，并且我们需要知道的是代理类只不过是一个空壳类，他需要调用另一个对象的方法，这个对象不能是代理对象，如果是代理对象的话，等于绕了一圈又绕回来了，这里java对这个类进行约束，他必须实现InvocationHandler接口(我们称之为中间类Z)，覆写invoke方法，所以当调用代理类的目标方法的时候就相当于调用中间类Z的intercept方法，而intercept方法又调用了被代理类（目标类M）的方法

```java
//中间类Z
public class Z implements InvocationHandler {

    //target为被代理类对象，也就是目标类对象
    private Object target;
    Z(Object target){
        this.target = target;
    }

    //其中proxy是代理类D名字为$Proxy数字比如  $Proxy0
    //其中method是用户调用的方法
    //args是用户调用方法时传进来的参数
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("代理对象调用invoke方法！");
        //这个表达式什么意思呢？mehtod表示要执行的方法，但是这个方法很多地方都有，我们要执行哪一个呢？target的method方法！即我们要执行目标类对象的这个method方法！！！
        method.invoke(target, args);
        return proxy;
    }

    public Object getProxyObject(){
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }
}
public class Main {
    public static void main(String[] args) {
        M m = new M();
        MInterface mProxy = (MInterface) new Z(m).getProxyObject();
        mProxy.sayHello();
    }
}
//被代理类，也就是目标类M
public class M implements MInterface{
    public void sayHello(){
        System.out.println("hello world!!!");
    }
}
//目标类实现的接口
public interface MInterface {
    void sayHello();
}
```

4.cglib动态代理中：

```java
public Object getProxyObject(Class<?> clazz){
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperClass(clazz);
    enhancer.setCallback(this);
    return enhancer.create();
}
```

我们通过enhancer.create()方法来构建代理类我们称之为类D(拼音开头)，但是在执行这个方法之前，我们需要告诉它我们要构建的代理类的父类（这个父类就是被代理类，或者说是目标类我们称之为M）是谁，并且我们需要知道的是代理类只不过是一个空壳类，他需要调用另一个对象的方法，这个对象不能是代理对象，如果是代理对象的话，等于绕了一圈又绕回来了，这里cglib对这个类进行约束，他必须实现MethodInterceptor接口(我们称之为中间类Z)，覆写intercept方法，所以当调用代理类的目标方法的时候就相当于调用中间类Z的intercept方法，而intercept方法又调用了被代理类（目标类M）的方法

```java
//中间类Z
public class Z implements MethodInterceptor {

    //第一个参数是代理类D（名字为M$$EnhancerByCGLIB$$c8f31597），第二个参数是用户调用的方法，第三个参数是用户调用方法时传入的参数，第四个参数是代理类的cglib方法
    public Object intercept(Object d, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("代理对象调用intercept方法");
        //我们这里不能使用method.invoke()方法，因为这里没有目标对象，只有代理类d即目标类的子类，所以我们通过invokeSuper()来反射执行目标方法
        methodProxy.invokeSuper(d, objects);
        return null;
    }
	//用于生成代理类D，当然这段代码完全不用写在中间类Z中，移动到其它类中也可以
    public Object getProxyObject(Class<?> clazz){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(this);
        return enhancer.create();
    }
}
public class Main {
    public static void main(String[] args) {
        M mProxy = ((M)new Z().getProxyObject(M.class));
        mProxy.sayHello();
    }
}
//目标类M，也就是被代理类
public class M {
    public void sayHello(){
        System.out.println("hello world!!!");
    }
}
/*输出
代理对象调用intercept方法
hello world!!!
*/
```

## 状态码

形如 200这样的状态码，这里的 3位数字中第1位数字，通常表示响应的类别（会有一两个例外），大致可以分成以下几类：

状态码	含义
1xx	请求正被处理
2xx	请求成功处理
3xx	请求需要附加操作，常见的例子如重定向
4xx	客户端出错导致请求无法被处理
5xx	服务端处理出错

```java
具体：
    
★200 OK  表示请求已经被正常处理，这个比较常见

★204 NO Content 正式请求的预请求，这个请求只需要确认后续的请求能不能通过，即只需要一个结果，而不需要返回其他内容，这类请求成功时就会返回204

★301 Moved Permanently 字面意思：资源被**永久**重定向了。这种情况下**响应**的头部字段`Location`中一般还会返回一个地址，用来表示要新地址

★302 Found 字面意思：资源**临时**重定向了。和301的唯一区别就在于一个是临时，一个是永久

★400 Bad Request 400的含义简单粗暴：“对不起，你的请求中有语法错误”，那具体是什么语法错误呢? 答案是 —— 不一定，一般来说响应报文里会有一些提示，例如：

- “哎呀，你多加了什么请求头，我不能接受呀”
- “哎呀，你地址不是不是写错了，这个uri不存在啊”
- “哎呀，你是不是请求方法错了，这个uri之只能用put而不是post”

★404
★500
```

## HashMap

建议研读HashMap的源码

1.容量问题：**HashMap的容量必须为2次幂**。如果初始化的时候指定了初始容量，HashMap会通过**tableSizeFor(arg)**方法来得到一个大于等于给定初始容量的数，而且这个数是2的次幂，比如给定容量为10，那么tableSizeFor(10)就会返回16这个给数

2.延迟加载：新建HashMap对象的时候并不会创建Node数组，而是在用户**第一次调用put方法的时候才会通过resize()方法创建Node数组**，这被称之为延迟加载

3.数据结构：数组+链表+红黑树。每一个key，value来了之后，通过put方法最终会被封装成一个Node放入数据结构中（或Node数组或链表或红黑树），这个Node的内部类包含了key与value属性还有next属性和hash属性，**这个hash属性是key的hash值经过“扰动”(HashMap中的hash方法，将key的hash右移16位与key的hash值相异或)得到的**。而进行插入值路由的时候通过的就是Node的hash值与Node数组长度减1

4.put方法：

通过Node的index = hash&(n-1)来进行路由，如果Node[index]==null那么直接填入就好

如果Node[index]!=null 说明该位置上有值，那么判断两者的key是否一致，如果一致，那么替换该值，返回旧值

如果不一致那么判断是否是TreeNode，如果是调用TreeNode的相关方法

否则进入链表查询，查询一致覆盖并返回旧值，不一致那么就追加节点

如果size>threhold则进行扩容

```java
//伪代码
Node p;int i;
if((p = Node[i = (n-1)&hash])==null){
    Node[i] = new Node(hash(key), key, v, null);
}else{
    if(p.key.equals(key))
        替换操作
    if(p instanceof TreeNode){
        TreeNode的插入操作
    }else{
        链表替换或插入操作
    }
}
if(++size>threhold){
    resize()
}
```

5.resize方法

如果某一个node的next位null那么就直接进行hash&（newCap-1）进行迁移

否则如果node为TreeNode节点，则进行TreeNode节点的迁移

否则node为链表，我们设计loHead，loTail与hiHead与hiTail，将node.hash与oldCap相与如果为0表示呆在该node呆在原位置不动，如果node.hash与oldCap相与如果为1表示新的index为原位置加上原容量oldCap。比如现在有一个链表第一个元素的hash为01111，第二个元素为11111，那么他们分别和oldCap相与分别为0和1为0的，在新的容量为32的node数组中，index仍然为15，而为1的，在新的node数组中index为15+oldCap=31

## 大文件小内存问题（待商榷）

**1.**现在主内存为1M，有一个10M大小的文件，文件中有汉字，假设每个汉字占用一个字节，想要根据汉字的个数进行排序请问怎么操作？

答：

1.首先遍历大文件，根据汉字数据的hash值分为若干个小文件，比如我分为20个文件，那么就会根据hash(汉字数据)%20来分文件，这样同一个hash的数据分配到同一个文件中，每个文件的大小可能就是500K这样我们就能将其读入内存了。

2.我们依次将每个小文件中的数据读入内存中再进行排序，之后落盘

3.将20个文件进行归并操作

**2.**待商榷的地方

1.java中文件以流读取，并不是说1M的内存就读不了10M的东西(当然这点上述分文件过程也有体现)

2.我建立一个map，key为byte的包装类型（占多少大小呀）用于存储汉字，value为汉字个数Integer

3.那么我1M的内存能装多少东西呢？byte对应的包装类里边一定有byte，Integer里边一定有int，byte占据一个字节，int占据4个字节，一个组合就要5个字节多，我们按10个字节来算的话，1M能存储100000即10万个汉字，而byte最多能保存256个汉字，并且我国汉字似乎也达不到10万，所以并不需要分文件，，，哈哈哈面试的时候就按正规的答就行了，不用考虑那么多

## 单例模式

```java
class Singleton {
    private Singleton() {
    }

    private static volatile Singleton singleton;

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

**volatile关键字的作用 **

为什么要加入volatile修饰singleton变量，因为singleton = new Singleton()包含三个步骤，第一开辟内存空间，第二初始化成员变量，第三赋值给singleton，如果第三步跑到了第二步之前，那么在线程a执行第三步之前，线程b判断到singleton==null并得到singleton对象并使用了，那么就会出现问题（因为对象还没有被初始化），所以要使用volatile关键字进行修饰

**为什么要双重检验**

答：第二层检验：线程a和线程b都执行到了synchronized前面，线程a获取了锁，等到线程a释放锁的时候，如果没有第二层检验，那么线程b就会创造新的对象出来。

第一层检验：如果直接加锁的话，每一个线程过来都要判断锁，开销比较大，只有当singleton为空的时候才需要判断锁，这样的话性能更高一点

## 怎么在Spring中实现事务

```java
@Transactional(isolation = Isolation.READ_COMMITTED, propagation = Propagation.REQUIRED)
public Object save1() {}
```

答：使用声明式事务比较简单，直接在方法前加入@Transacional注解就可以了，并且在参数中指明隔离级别以及传播方式

场景：项目中在某个帖子下评论的时候，这涉及到两个表discuss_post表里边的comment_count字段也就是这个帖子的总的评论数要加1，这个要显示在页面山的；另外一个表是comment表的数据插入，这两步操作是一个事务，因为网页上的帖子的评论数和总的评论是要对应起来的。这个时候就要使用spring的事务

## 事务的特性

事务的四大特性主要是：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）

1.官网上事务**一致性**的概念是：事务必须使数据库从一个一致性状态变换到另外一个一致性状态。

​    换一种方式理解就是：**事务按照预期生效，数据的状态是预期的状态。**

​    举例说明：**张三向李四转100元，转账前和转账后的数据是正确的状态，这就叫一致性**，如果出现张三转出100元，李四账号没有增加100元这就出现了数据错误，就没有达到一致性。

2.事务的隔离性是多个用户并发访问数据库时，数据库为每一个用户开启的事务，不能被其他事务的操作数据所干扰，多个并发事务之间要相互隔离。

3.原子性指的是事务内的一组操作要么全都没发生，要么全都发生，并不存在某些操作发生而某一些没有的状态

4.持久性是指事务一旦提交成功，那么就会记录在数据库中（这是通过redo log来实现的）

## spring 控制反转依赖注入的区别与联系

联系：控制如果反转了，那么就会依赖注入

下面的例子解释了控制反转，依赖注入，简单工厂模式，工厂模式

我们先来看一下简单工厂模式，并且博主在简单工厂模式中顺便也解释了什么是依赖注入，或者说解释依赖注入的同时解释了什么是工厂模式，现在我们想要制造一辆汽车，设计的依赖关系是轮子大小决定底盘大小，底盘大小决定车架子大小，车架子的大小决定车的大小，那么就有

```java
class Car{
    private Framework framework;   
    Car(){this.framework = new Framework(); }
}
class Framework{
    private Bottom bottom;
    Framework(){this.bootom = bottom;}
}
class Bottom{
    private Tire tire;
    Bottom(){this.tire = new Tire(); }
}
class Tire{
    private int size;
    Tire(){}
    Tire(int size){size = 3;}
}
那么我要新建立一辆车的时候就会有
Car car = new Car();//完事
```

但是如果我现在想要在新建车的时候控制轮子的大小呢？

```java
class Car{
    private Framework framework;   
    Car(int size){this.framework = new Framework(size); }
}
class Framework{
    private Bottom bottom;
    Framework(int size){this.bootom = new Bottom(size);}
}
class Bottom{
    private Tire tire;
    Bottom(int size){this.tire = new Tire(size); }
}
class Tire{
    private int size;
    Tire(int size){this.size = 3;}
}
那么我要新建立一辆车的时候就会有
Car car = new Car(9999);//可以控制轮子大小
```

我们看到这需要将Car所有的依赖类全部做更改，这很不利于程序扩展，这就是因为每一个类都是主动创建的依赖类，而不是依赖注入，其属性不是依赖由第三方的注入，我们再来看第二版

```java
class Car{
    private Framework framework;   
    Car(Framework framework){this.framework = frameork; }
}
class Framework{
    private Bottom bottom;
    Framework(Bottom bottom){this.bootom = bottom;}
}
class Bottom{
    private Tire tire;
    Bottom(Tire tire){this.tire = tire; }
}
class Tire{
    private int size;
    Tire(){}
    Tire(int size){this.size = 3;}
}
//那么我要新建立一辆车的时候就会有，这是如果我还想要控制车的大小，可以直接在new Tire的地方更改就可以了
Tire tire = new Tire();
Bottom bottom = new Bottom(tire);
Framework framework = new Framework(bottom);
Car car = new Car(framework);
```

我们看到更改为依赖注入的话，程序就相对于扩展比较友好了（满足依赖转置原则），但是创建对象似乎变得麻烦了，该怎么办呢？将建立Car的那一部分代码封装起来，这就是简单工厂！

简单工厂的特点是，一个工厂创建所有的产品（汽车啊，飞机啊，大炮啊），并不满足开闭原则，因为如果你新加入一个产品类，比如飞机类，那么工厂内部的代码就要改动，这时候便有了工厂模式

工厂模式相比于简单工厂的话就是满足开闭原则，新来一个产品的话就建立一个工厂生产这种产品，原来的工厂类并不需要改动

那么这些工厂都是生产产品的，所以也抽象出了一个工厂的接口，新建立的工厂都会实现这个接口

## 动态规划

 解码方法

说明：

1.该解法很有可取之处，首先我们知道如果dp.length = s.length()那么dp的i对应s的i，如果dp.length=s.length()+1那么dp的i对应s的i-1，考虑到我们还需要考查dp的i的前一位也就是i-1而对应到s就是要考察i-2的位置，所以要考虑到坐标的问题！！！**而此节法中将dp的i从2取起，完美地解决了这一个问题！！！**

2.至于dp[0]和dp[1]的取值，我们只要让使其让前几个dp成立即可，因为k成立的时候k+1一定也成立，数学归纳

```java
class Solution {
    public int numDecodings(String s) {
        if (s == null || s.length() == 0)
            return 0;
        if(s.charAt(0)=='0')
            return 0;
        int[] dp = new int[s.length() + 1];
        dp[0] = 1;
        dp[1] = 1;
        for (int i = 2; i < dp.length; i++) {
            char mid = s.charAt(i - 1);
            char lo = s.charAt(i - 2);
            if (mid == '0' && (lo == '1' || lo == '2')) {
                dp[i] = dp[i - 2];
            } else if (mid == '0' && (lo == '0' || lo > '2')) {
                return 0;
            } else if ((lo == '1' && mid >= '1' && mid <= '9') || (lo == '2' && mid >= '1' && mid <= '6')) {
                dp[i] = dp[i - 1] + dp[i - 2];
            } else {
                dp[i] = dp[i - 1];
            }
        }
        return dp[dp.length - 1];
    }
}
```

## 进程和线程

进程是操作系统资源分配的基本单位，每个进程都有独立的代码和数据空间（程序上下文），程序之间的切换会有较大的开销，一个进程包含1-n个线程

而线程是处理器任务调度和执行的基本单位，同一类线程共享代码和数据空间，每个线程都有自己独立的运行栈和程序计数器（PC），线程之间切换的开销小。

## 类加载双亲委托模型

在介绍双亲委派模型之前先说下类加载器。对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立在 JVM 中的唯一性，每一个类加载器，都有一个独立的类名称空间。类加载器就是根据指定全限定名称将 class 文件加载到 JVM 内存，然后再转化为 class 对象。

类加载器分类：
1.启动类加载器(Bootstrap ClassLoader)用来加载java核心类库，无法被java程序直接引用。
2.扩展类加载器(extensions class loader):它用来加载 Java 的扩展库。Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。
3.应用程序类加载器（Application ClassLoader）。负责加载用户类路径（classpath）上的指定类库，我们可以直接使用这个类加载器。一般情况，如果我们没有自定义类加载器默认就是用这个加载器。

![2020080901](F:\.image\2020080901.png)

双亲委派模型：如果一个类加载器收到了类加载的请求，它首先不会自己去加载这个类，而是把这个请求委派给父类加载器去完成，每一层的类加载器都是如此，这样所有的加载请求都会被传送到顶层的启动类加载器中，只有当父加载无法完成加载请求（它的搜索范围中没找到所需的类）时，子加载器才会尝试去加载类。

**双亲委派的好处**

```java
1.避免了类被重新加载
2.避免了java核心类被篡改，比如用户自定了Object类，通过双亲委派机制从应用程序类加载器->扩展类加载器->启动类加载器，启动类加载器发现Object已经加载过了，就不会再进行加载了
```

## 死锁怎么发生的

**1.互斥使用（资源独占）**：一个资源每次只能给一个进程使用
**2.占有且等待（请求和保持，部分分配）**：进程在申请新的资源的同时保持对原有资源的占有。
**3.不可抢占（不可剥夺）**：资源申请者不能强行的从资源占有着手中多去资源，资源只能由占有着自愿释放
**4.循环等待**
存在一个进程等待队列｛P1，P2，......，Pn｝，其中P1等待P2占有的资源，P2等待P3占有的资源，......，Pn等待P1占有的资源，形成一个进程等待环路。

## 手写一个死锁程序

```java
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (A.class){
                    System.out.println("tryAcquireB");
                    try {
                        Thread.sleep(1000);
                        synchronized (B.class){
                            System.out.println("fadsjk");
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (B.class){
                    System.out.println("tryAcquireA");
                    try {
                        Thread.sleep(1000);
                        synchronized (A.class){
                            System.out.println("fdsadsakj");
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        thread1.start();
        thread2.start();
    }
    class A{}
    class B{}
}
```

## Jvm运行时数据区

线程私有：**程序计数器**：指向下一句要执行的字节码语句；**本地方法栈**：调用本地方法时使用到的栈；**虚拟机栈**：由栈帧构成，栈帧包括局部变量表，操作数栈，方法出口等

线程共有：**虚拟机堆**：对象所在区域；方法区：**类结构信息，常量，静态变量**

jdk1.8以后，**运行时常量池**放在了堆中（之前放在方法区中），元空间替代了永久代（方法区的实现方式），元空间使用了直接内存，不容易溢出

**局部变量表**：存储一个方法内部的局部变量

**操作数栈**：用于运算，比如某方法中有int a  = 1;int b = 2;int c = a+b;那么a的值1会入栈，b的值2也会入栈然后进行相加。。。。

**方法出口**：比如main方法中调用了add方法，那么add方法入栈后栈帧中就会存在方法出口信息，因为方法出口信息指明了add方法执行完之后该回到main方法的哪个地方

## JVM参数设置

-Xms堆初始大小 -Xmx堆最大大小

-Xmn新生代大小

堆大小-新生代大小=老年代大小

-Xss栈大小

-XX:MetaspaceSize原空间大小

## OOM

这里仅仅演示OOM的一种情况，并主要看垃圾去向流程

参数设置-Xms20m -Xmx20m -Xmn10m将新生代设小一点是为了伊甸园区很快满掉，进行minor gc（发生在整个新生代）以短时间内就观察到垃圾去向，将整个堆设小一点是为了老年代小一点（因为堆大小-新生代大小=老年代大小）图上看垃圾好看出来，不然大的老年代，小的垃圾变化在图上就根本看不出来

程序如下，使用了jdk8

```java
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        byte[] bytes = new byte[1024*100];
        List<Main> list = new ArrayList<>();
        while(true){
            list.add(new Main());
            Thread.sleep(10);
        }
    }
}

```

使用jvisualvm（java visualVM）观察到

![20200708140029](F:\.image\20200708140029.png)

我们看到GC的时候，伊甸园区占用空间减少，幸存区0与幸存区1占用空间交替变化（一会0无1有一会1无0有），老年代逐渐增加（这里看不清楚）

## volatile关键字

volatile关键字保证了**可见性**，**禁止指令重排**，**但是不保证原子性**

## B+树

**b+树与b树相比有3个优势**：

1.非叶子节点不会存储卫星数据，所以同样大小的数据页将存储更多的数据，这意味着索引树更加的矮胖，意味着IO的次数更少，速度更快

2.由于b树非叶子节点和叶子节点都存储卫星数据，所以它的性能并不稳定，最好的情况是查找到根节点就得到结果，最坏的结果是查找到叶子节点。b+树的话每一次查找都会找到叶子节点，性能稳定

3.b+树叶子节点之间存在双向链表，做范围查询更加方便

## 红黑树

红黑树主要有如下若干个特点：

**1.首先树的节点有颜色，不是黑色就是红色**；

**2.根节点是黑颜色的**；

**3.所有的叶子节点都是空的黑子节点**；

**4.不能够出现连续的红颜色的节点**；

**5.从根结点出发到任意的子节点的路径上黑节点的数目相同**；

这五个特征保证了在红黑树中**最长路径不超过最短路径的两倍**，这样也就保证了查找的效率问题

## 垃圾回收器

答：Serial（sei饿瑞藕）收集器（复制算法): 新生代**单线程**收集器，标记和清理都是单线程，优点是简单高效；

ParNew收集器 (复制算法): 新生代收并行集器，实际上是Serial收集器的多线程版本，在多核CPU环境下有着比Serial更好的表现；

Parallel Scavenge收集器 (复制算法): 新生代并行收集器，追求高吞吐量，高效利用 CPU。吞吐量 = 用户线程时间/(用户线程时间+GC线程时间)，高吞吐量可以高效率的利用CPU时间，尽快完成程序的运算任务，适合后台应用等对交互相应要求不高的场景；

Serial Old收集器 (标记-整理算法): 老年代单线程收集器，Serial收集器的老年代版本；

Parallel Old收集器 (标记-整理算法)： 老年代并行收集器，吞吐量优先，Parallel Scavenge收集器的老年代版本；

CMS(Concurrent Mark Sweep)收集器（**标记-清除算法**）： 老年代并行收集器，**以获取最短回收停顿时间为目标**的收集器，具有高并发、低停顿的特点，追求最短GC回收停顿时间。（其他版本：**以牺牲吞吐量为代价，获取最短回收停顿时间**）

G1(Garbage First)收集器 (标记-整理算法)： Java堆并行收集器，G1收集器是JDK1.7提供的一个新收集器，G1收集器基于“标记-整理”算法实现，也就是说不会产生内存碎片。此外，G1收集器不同于之前的收集器的一个重要特点是：G1回收的范围是整个Java堆(包括新生代，老年代)，而前六种收集器回收的范围仅限于新生代或老年代。(其他版本：**兼顾了吞吐量和回收停顿时间**)

## Java 基本数据类型

byte short int long

boolean char float double

## Http 请求头里有哪些参数

GET / HTTP/1.1 #**请求行:包含了请求方式GET/POST,请求资源路径(这里没有),所用协议HTTP1.1**
#以下部分是请求头,如果有请求体,请求头和请求体之间要有一空行
**Host: 192.168.1.106:9527请求访问的ip地址与端口号**
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36#这是客户端所用的浏览器版本等信息,有些网页的某些信息显示不了,提示需要更高版本的浏览器就是用这些信息判断的
**Accept**: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
**客户端浏览器支持的文件类型**
**Accept-Encoding**: gzip, deflate#**客户端浏览器所支持的压缩文件类型**,如果请求的网页文件过大,服务器会根据浏览器所支持的压缩文件类型将网页资源压缩发送给浏览器
**Accept-Language**: zh-CN,zh;q=0.9#**浏览器支持的语言**,这个在浏览器上可以设置
#以上部分是请求头

## Https的通信过程

客户端发送请求到服务器端
服务器端返回证书和公开密钥，公开密钥作为证书的一部分而存在
客户端验证证书和公开密钥的有效性，如果有效，则生成共享密钥并使用公开密钥加密发送到服务器端
服务器端使用私有密钥解密数据，并使用收到的共享密钥加密数据，发送到客户端
客户端使用共享密钥解密数据
SSL加密建立

## MYSQL创建索引

```sql
CREATE INDEX index_name ON table_name (column_list)
```

## 七层/五层模式

![2020070900](F:\.image\2020070900.png)

## 三次握手，四次挥手

![2020070903](F:\.image\2020070903.png)

```java
所谓三次握手（Three-Way Handshake）即建立TCP连接，就是指建立一个TCP连接时，需要客户端和服务端总共发送3个包以确认连接的建立。整个流程如下：

第一次握手：Client将标志位SYN置为1(表示请求联机)，随机产生一个值(Sequence Number顺序号码)seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态(请求连接信号已发送状态)，等待Server确认。
第二次握手：Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK（确认标志位）都置为1，ack（确认号码）=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。
第三次握手：Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED(已建立)状态，完成三次握手，随后Client与Server之间可以开始传输数据了。
```

![2020070904](F:\.image\2020070904.png)

```java
第一次分手：主机1（可以使客户端，也可以是服务器端），设置Sequence Number和Acknowledgment Number，向主机2发送一个FIN报文段；此时，主机1进入FIN_WAIT_1状态；这表示主机1没有数据要发送给主机2了；

第二次分手：主机2收到了主机1发送的FIN报文段，向主机1回一个ACK报文段，Acknowledgment Number为Sequence Number加1；主机1进入FIN_WAIT_2状态；主机2告诉主机1，我“同意”你的关闭请求；

第三次分手：主机2向主机1发送FIN报文段，请求关闭连接，同时主机2进入LAST_ACK状态；

第四次分手：主机1收到主机2发送的FIN报文段，向主机2发送ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段以后，就关闭连接；此时，主机1等待2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，主机1也可以关闭连接了。
```

## 为什么要进行三次握手

```java
已失效的连接请求报文段”的产生在这样一种情况下：client发出的第一个连接请求报文段并没有丢失，而是在某个网络结点长时间的滞留了，以致延误到连接释放以后的某个时间才到达server。本来这是一个早已失效的报文段。但server收到此失效的连接请求报文段后，就误认为是client再次发出的一个新的连接请求。于是就向client发出确认报文段，同意建立连接。假设不采用“三次握手”，那么只要server发出确认，新的连接就建立了。由于现在client并没有发出建立连接的请求，因此不会理睬server的确认，也不会向server发送数据。但server却以为新的运输连接已经建立，并一直等待client发来数据。这样，server的很多资源就白白浪费掉了。采用“三次握手”的办法可以防止上述现象发生。例如刚才那种情况，client不会向server的确认发出确认。server由于收不到确认，就知道client并没有要求建立连接。”
```

## 为什么要进行四次挥手

```java
Server在LISTEN状态下，收到建立连接请求的SYN报文后，可以直接把ACK和SYN放在一个报文里发送给Client。而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送。
```

## InnoDB和MyISAM引擎之间的区别

**1、InnoDB支持事务，MyISAM不支持；**

2、InnoDB数据存储在共享表空间，MyISAM数据存储在文件中；

**3、InnoDB支持行级锁，MyISAM只支持表锁；**

**4、InnoDB支持崩溃后的恢复，MyISAM不支持；**

5、InnoDB支持外键，MyISAM不支持；

## Redis在SpringBoot中的配置

1.事实上，springboot对redis进行了整合，所以说如果我们在pom.xml中使用的是整合之后的包的话（spring-boot-starter-data-redis），redis是不用进行配置的，是开箱即用的，也就是说我们可以直接在程序中通过

```java
@Autowired
private RedisTemplate redisTemplate;
```

使用redis了，但是在springboot自动配置的redis的key和value是Object类型的，这样兼容性更强，可以存储任意的对象，但是我们在项目中最常使用的key与value分别是String，Object类型的所以我们需要自己重新配置一下以方便使用

所以我们在项目的config包下建立RedisConfig类

```java
package com.nowcoder.community.config;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // 设置key的序列化方式
        template.setKeySerializer(RedisSerializer.string());
        // 设置value的序列化方式
        template.setValueSerializer(RedisSerializer.json());
        // 设置hash的key的序列化方式
        template.setHashKeySerializer(RedisSerializer.string());
        // 设置hash的value的序列化方式
        template.setHashValueSerializer(RedisSerializer.json());

        template.afterPropertiesSet();
        return template;
    }
}
```

这样我们就能够方便的使用redis工具了

## 项目中哪些地方使用到了redis？

1.登录凭证我们使用了redis存储，因为http是无状态的协议，在这个请求中我们登陆了用户，但是在下一次请求中服务器就不知道点赞的用户是谁了，这样我们就需要登录凭证，当然这个登录凭证可以选择使用session存储，但是session存储的主要缺点就是数据存储在本服务器上，如果是多服务器应用，而下一次nginx将请求发送到哪个服务器进行处理就不确定了（当然这取决于nginx采用的是何种负载均衡算法），这样的话用户的状态就丢失了，使用redis处理的话，我们使用的是同一个redis，或者是同一个集群的redis，内部的信息具有一致性，这样我们就能够正确获取到用户登陆状态，这个用户的登录凭证除了session还有其他办法可以进行存储，这里进行统一的说明

|---可以使用cookie进行存储，但是cookie的话是将信息存储到客户端，这样的话信息不够安全

|---可以将信息存储到session中，但是问题是如果两次的请求不打到同一台服务器上，用户的登陆状态还是无法维持

|---可以将登陆凭证信息存储到db（数据库）中，前期的项目的处理方式就是这样（多个服务器但是是同一个db所以不会出现session的问题），使用的是login_ticket表，里边包含了id，user_id，ticket，status，expired字段，其中第一次登陆的时候，服务端会随机生成一个字符串作为正在登录的用户的标识，并将这个随机字符串以及对应的随机字符串分别写在user_id与ticket字段中，status为默认值0表示登录状态，1为退出状态，其中expired字段为登陆凭证的过期时间，如果没有勾选登陆页面的记住我的选项，该凭证的expired字段会生成一个默认的较短的时间，如果点击了记住我，那么就会生成一个较长时间的expired时间（timestamp）

|---使用redis，使用db的主要问题就是性能不高，因为查询用户的登陆凭证是访问十分频繁的功能（在mvc的拦截其中，每一个请求都会先查询），所以对性能的要求是很高的，我们使用redis进行存储，直接废弃掉原来的login_ticket表

## TCP和UDP的区别？

答：从**可靠性，数据量大小，速度**三个方面来回答

TCP的话，传输数据之前要进行三次握手，是可靠的数据连接。可以进行大量的数据传输。因为通知之前需要建立连接，速度会降低

UDP并不需要建立连接，是不可靠传输，直接发送数据报，源地址和目标地址都写在了数据报中，每个数据报的大小为64k，超过了要分报发送

## Java运算符号的优先级

![202000907](F:\.image\202000907.jpg)

**注意：**

if(某某==某某 && 某某==某某)**里边并不用**使用括号，因为==的优先级大于&& || & |的优先级！！真棒！！！

```java
Integer count = null==map.get(a)?0:map.get(a)
```

==号和?:的优先级大于=，所以上述程序的顺序你已经应该晓得了，是不是很美！！！

## 长连接

答：我们如果与mysql建立了连接而不做任何动作，那么默认8小时后就会断开连接，这时候再执行sql语句的时候就会报错，而如果我们在一个连接中经常使用各种语句的话，这个连接就不会断开，这就是一个长连接，那么长连接有哪些利弊呢？**利**不需要频繁地建立连接，节约性能，**弊**在一个连接中我们使用到的内存都是维护在这个连接对象里的，只有当一个连接断开之后我们才能够释放这些内存，所以如果一个连接过长，可能会出现oom，导致操作系统kill掉mysql进程

综上，我们要定期断开长连接，并且在一些比较大的sql语句执行之后，我们可以使用mysql_reset_connection来重新初始化连接资源，这个命令不会涉及重连和身份权限验证

## 数据库的隔离级别

**不提交读**，在该级别下，一个事务中对数据进行了修改但是并没有进行提交，另一个事务也会有数据的变化，这样就会出现**脏读**的现象，就是说在一个事务中读到了另一个事务的未提交的数据（万一头一个事务回滚，那么另外一个事务读到的就是错误的数据）

**提交读**，该级别主要是解决脏读的现象，就是说虽然在一个事务中对数据进行了更改，但是只要在这事务并没有被提交，另一个事务中就不会有数据的变化，但是这种隔离模式下会有不可重复的问题，就是在一个事务中前后两次读取的数据并不一致（因为在这期间头一个事务进行了提交操作）

**可重复读**，在该级别下，一个事务中查询同一份数据不会有数据变化的问题，但是可能会有数据总数的问题，也就是幻读现象

**串行读**，没有任何问题，但是性能最低

## 为什么重写equals一定要重写hashcode

1.因为Java规范规定两个对象equals那么他们的hashcode一定相等

2.如果不从写hashcode的话会造成一些问题，比如使用HashMap容器的时候，我们使用自定义对象作为key，重写了equals方法，但是没有重写hashcode方法，那么同一个对象或者说equals相等的两个对象就会可能由于hashcode的不同打到不同的桶位上，这与预期不符，预期的话这种情况会达到同一个桶位上，又因为两个对象equals所以会覆盖的

## redirect和forward的区别

**1.从地址栏显示来说**

forward是服务器请求资源,服务器直接访问目标地址的URL,把那个URL的响应内容读取过来,然后把这些内容再发给浏览器.浏览器根本不知道服务器发送的内容从哪里来的,所以它的地址栏还是原来的地址.

redirect是服务端根据逻辑,发送一个状态码,告诉浏览器重新去请求那个地址.所以地址栏显示的是新的URL.

**2.从数据共享来说**

forward:转发页面和转发到的页面可以共享request里面的数据.

redirect:不能共享数据.

**3.从运用地方来说**

forward:一般用于用户登陆的时候,根据角色转发到相应的模块.

redirect:一般用于用户注销登陆时返回主页面和跳转到其它的网站等.

**4.从效率来说**

forward:高.

redirect:低.

## 线程池的优势

**1.线程池可以重复利用已创建的线程，一次创建可以执行多次任务，有效降低线程创建和销毁所造成的资源消耗；**
**2.线程池技术使得请求可以快速得到响应，节约了创建线程的时间；**（解析：无线程池，那么执行多线程任务的时候就有创建线程+执行线程任务+销毁线程三个步骤，有了线程池，执行多线程的时候就只会有执行线程这一步操作，大大节约了性能消耗，提高了响应速度）
3.线程的创建需要占用系统内存，消耗系统资源，使用线程池可以更好的管理线程，做到统一分配、调优和监控线程，提高系统的稳定性。

解析：线程模型分为用户线程模型(ULT  user-level-thread)和内核线程模型（KLT kernal level-thread）,java虚拟机规范中并没有规定java中的线程使用什么模型来实现，目前的JVM一般使用的是KLT线程模型来实现java中的多线程，也就是说java中的一个线程对应系统下的一个线程，比如我们现在有如下一个程序

```java
public class Main {
    public static void main(String[] args) {
        for (int i = 0; i < 1000; i++) {
            Thread t = new Thread(new Task());
            t.start();
        }
    }
}
class Task implements Runnable {
    @Override
    public void run() {
        while (true){
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

打开任务管理器，当执行此段程序的时候我们发现任务管理器上的线程数就会增加1000个左右了，这就是JVM使用的是KLT实现多线程的证据

我们在启动多线程的时候，JVM底层会调用系统软件的API来创建线程，内核态的创建和销毁线程都很耗费系统性能，为了能够让线程重复地执行任务，减少创建和销毁线程的性能消耗，我们便有了线程池的技术

**请问：上述程序如果线程数很多，比如说欲建立1千万个线程该会发生什么情况呢？**

答：Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"

你会看到任务管理器性能中，线程数开始逐步上升，上升到一定程度java程序出现oom异常，线程数瞬间回落

## Callable接口的返回值怎么取到

答：通过实现Callable接口覆写call方法定义的任务需要使用线程池执行

```java
import java.util.concurrent.*;

public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService es = Executors.newFixedThreadPool(2);
        Future<String> f = es.submit(new TaskCallable());
        String s = f.get();

        System.out.println(s);
        es.shutdown();
    }
}
class TaskCallable implements Callable<String> {

    @Override
    public String call() throws Exception {
        return "通过实现Callable接口覆写call方法定义的任务执行啦！";
    }
}
```

说明：

**1.Executor是线程池的祖师爷级别的接口，所有线程池类的父类接口**

**2.Executors是一个工具类，类似于Arrays，Collections，能够方便地创建线程池，但是阿里禁止使用这个工具类来创建线程池**

**3.ExecutorService是Executor的一个子类，有很多Executor没有的方法，包括submit，shutdown等等**

**4.ExecutorService类的submit方法的返回值是Future接口类型，通过该接口的get方法我们能够取得任务中的call方法的返回值**

## Java的int类型

**1.int类型的取值范围**

答：-2^31到2^31-1

**2.为什么是31次方而不是32次方**

答：因为最高位有一个符号位

**3.为什么最大范围是2^31-1**

答：因为整数最高位位0，32位能代表的最大值为

01111111 11111111 11111111 11111111

**4.为什么最小值位2^31次方**

答：负数的二进制表示为对应的整数最高位置为1，其余位取反，在加上1，比如**-1的二进制为(全是1记住)**

11111111 11111111 11111111 11111111

那么0的表示为全是0，那么-0呢？按照上述规则为

10000000 00000000 00000000 00000000

但是数学上没有-0，这个数可以用来表示最小的数即-2^31

## 手写一个自定义注解

```java
@Retention(RetentionPolicy.RUNTIME)//起效时间
@Target(ElementType.METHOD)//作用在什么地方，表明这个注解是在方法上起作用
public @interface Test {
}
```

注解是通过反射起作用的，这里给出全部代码，注意这里的代码还包含了cglib动态代理

现在有Solution类

```java
import annotation.Test;

class Solution {
    @Test
    public int numDecodings(String s) {
        System.out.println(s);
        System.out.println("jklnlkn");
        if (s == null || s.length() == 0)
            return 0;
        if(s.charAt(0)=='0')
            return 0;
        int[] dp = new int[s.length() + 1];
        dp[0] = 1;
        dp[1] = 1;
        for (int i = 2; i < dp.length; i++) {
            char mid = s.charAt(i - 1);
            char lo = s.charAt(i - 2);
            if (mid == '0' && (lo == '1' || lo == '2')) {
                dp[i] = dp[i - 2];
            } else if (mid == '0' && (lo == '0' || lo > '2')) {
                return 0;
            } else if ((lo == '1' && mid >= '1' && mid <= '9') || (lo == '2' && mid >= '1' && mid <= '6')) {
                dp[i] = dp[i - 1] + dp[i - 2];
            } else {
                dp[i] = dp[i - 1];
            }
        }
        return dp[dp.length - 1];
    }
}
```

有中间类（cglib中的内容，类里边有用于生成代理对象的方法，有代理对象调用的方法）

```java
import annotation.Test;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class InterClass implements MethodInterceptor {

    public Object getSolutionProxy(){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Solution.class);
        enhancer.setCallback(this);
        return enhancer.create();
    }

    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        Test test = method.getAnnotation(Test.class);
        Object res = null;
        if(test==null) {
            res = methodProxy.invokeSuper(o, objects);
        }else{
            //do something
        }
        return res;
    }
}

```

我们可以在//do something处生成测试用例，用于检测我们在Solution类中写的方法是否正确

main方法如下

```java
public class Main {
    public static void main(String[] args){
        Solution solution = (Solution) new InterClass().getSolutionProxy();//生成的是代理对象
        int d = solution.numDecodings("123");
        System.out.println(d);
    }
}
```

## Java中的流体系

![20200712114812](F:\.image\20200712114812.png)

**1.所有的流都实现了Closeable接口，表明所有的流都是可关闭的**

**2.Reader与Writer为字符流，InputStream与OutputStream为字节流**

**3.FileReader与FileWriter为与文件打交道的字符流，FileInoutStream与FileOutputStream为与文件打交道的字节流**

**4.BufferedInputStream与BuferedOutputStream为字符流缓冲器，使用装饰器模式，用来装饰缓冲InputStream的子类，BufferedReader与BufferedWriter类似**

**5.InputStream与OutputStream为转换器可以将字节流转换为字符流**，同时也分别是某某的父类

## 装饰器模式

本节内容配合《Java中的流体系》中BufferedInputStream与BufferedReader等理解效果更佳，因为Buffered***使用的就是装饰器模式

```java
class Demo21_18_19{
    public static void main(String[] args){
        NewWorker newW = new NewWorker();

        newW.walk();
    }
}
abstract class Person{
    int age;
    String name;
    Person(){}
    Person(String name, int age){this.name = name;this.age = age;}
    abstract void walk();
}
class Worker extends Person{
    Worker(){}
    Worker(String name, int age){
        super(name, age);
    }
    void walk(){
        System.out.println("两条腿走路");
    }
}
class NewWorker extends Worker{
    void walk(){
        System.out.println(this.name+"......"+this.age+"......."+"坐地铁");
        super.walk();
    }
}
/*
null......0.......坐地铁
两条腿走路
*/
```

1.上述程序中，我们先有了一个类Worker，类Worker中有一个方法walk，随着程序的发展，类Worker中的walk方法升级了，需要对原有的walk方法进行**功能的扩展和增强**，我们必须要明确的一点是不能在原有的Worker类里边进行修改，因为改动原有的已经正确的程序可能是一场灾难

2.很多小伙伴可能会想到利用Java的继承机制，上面的程序中我们就是以Java的继承机制实现的，我们这里先讨论Java的继承机制的缺点，再引出装饰器设计模式

这种方式有什么缺点呢？比如说我们又有一个类为Student，这个类继承了Person类，也有walk方法，但是我们也要对其进行功能的增强，使得Student对象的walk方法也具备"坐地铁"的高效性，这样子的话，这个Person体系为：

    Person
        |--Worker
            |--NewWorker
        |--Student
            |--NewStudent

那么如果我再出现其他的类别呢？比如说又出现了Doctor类，他也具备walk的功能，我也想让他具备"坐地铁"的高效性，我要专门为了提高walk的效率而再次新建一个类NewDoctor这样子的话就会使得Person的体系越来越臃肿

怎么解决呢？不让体系变得臃肿而且使得已存在的一些类中的一些方法变得高效？我们可以将高效的方法抽离出来，封装成一个独立的对象，如:

```java
class Demo21_18_19_2{
    public static void main(String[] args){

        Worker w = new Worker();
        NewPerson newP = new NewPerson(w);

        newP.walk();
    }
}
abstract class Person{
    int age;
    String name;
    Person(){}
    Person(String name, int age){this.name = name;this.age = age;}
    abstract void walk();
}
class Worker extends Person{
    Worker(){}
    Worker(String name, int age){
        super(name, age);
    }
    void walk(){
        System.out.println("两条腿走路");
    }
}
class NewPerson extends Person{
    private Person p;
    NewPerson(Person p){
        this.p = p;
    }
    void walk(){
        System.out.println(p.name+"......"+p.age+"......"+"坐地铁");
        p.walk();
    }
}
/*
null......0......坐地铁
两条腿走路
*/
```

我们看到上述程序中，我们没有使用继承机制，而是建立了一个NewPerson的类，里边封装了"坐地铁"的使walk方法变得高效的方法，这样子的话无论我们传入什么Peroson的子类，无论是Worker还是Student类对象都可以使用，使得自己的walk方法变得高效，这样子的话Person的体系就为:

    Person
        |--Worker
        |--Student
        |--NewPerson

体系结构清晰多了，这里使用的就是装饰设计模式

## 深浅拷贝

1.拷贝即复制，浅拷贝拷贝的是对象地址，深拷贝将对象的所有信息又复制了一份，下面的图很能说明问题

![20200712203644](F:\.image\20200712203644.png)

2.怎么进行深拷贝？

通过clone方法，通过各种第三方包，比如Gson，jackson，apache commons Lang等，一般的思路都是将对象先进行序列化，之后再反序列化

3.通过clone方法怎么做？

```java
import java.io.FileWriter;
import java.util.Arrays;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) throws CloneNotSupportedException {
        Address address = new Address("成都");
        User user = new User("张三",address);

        User userClone = user.clone();
        
        System.out.println(userClone==user);
        System.out.println(userClone.getName());
        System.out.println(userClone.getAddress().getName());
    }
}
class User implements Cloneable{
    String name;
    Address address;

    @Override
    public User clone() throws CloneNotSupportedException {
        User user = (User)super.clone();
        user.setAddress(this.address.clone());
        return user;
    }
}

class Address implements Cloneable{
    private String name;

    @Override
    public Address clone() throws CloneNotSupportedException {
        return (Address)super.clone();
    }
}
/*
false
张三
成都
*/
```

上述程序中省去了get，set与构造方法

1.使用clone方法进行深克隆，"本体"也就是被克隆的对象需要**实现Cloneable空接口**，**覆写上帝类Object的clone方法**，**将clone方法权限改为public**，**返回值改为"本体类型"**，**内部调用上帝类的clone方法并强转为本体对象**，并且如果本体中存在类类型的属性（类属性的类也要实现Cloneabe接口，覆写clone方法等）那么也要这clone方法内部通过类类型属性所属的类的clone方法进行赋值与赋值（意思就是程序中user.setAddress(this.address.clone())这段代码要有）

## SpringBoot中常用的注解

@Autowired自动注入注解，用于注入属性对象

@Bean与@Configuration配置类注解与Bean注解，在配置Quartz的时候我们将配置类使用@Configuration修饰，要装配的Bean使用@Bean修饰，参见《BeanFactory与FactoryBean的区别》章节

@Controller注解，表明这是一个SprinMVC层面的Bean，@Service表明这是一个Service层面的Bean

@RequestMapping注解，可以修饰在类和方法上，表明资源的访问地址

@RequestBody注解，表明这个方法返回的是一个字符串，而不是默认的网页

## BeanFactory与FactoryBean的区别

（博主自己认为的，不一定对）

1.首先我们要知道spring本质是一个工厂，用来生产和管理各种bean解决bean之间的依赖，用到了工厂模式，所有的bean都是产品，按照工厂模式的话，新来一个产品的话，那么一定会有一个对应的工厂（当然如果bean的创建比较简单的话就不需要工厂），这些个工厂都是生产bean的，spring为这些工厂抽象出了一个共同的接口，这个接口就是FactoryBean

2.BeanFactory的话他是spring容器的顶层接口，里边提供了一些操作bean的方法，比如根据类名获取bean，根据类名和类类型来获取bean，判断bean是否为单例模式，判断bean是否为原型模式等，ApplicationContext是它的一个常用子接口

3.那么我们项目哪些地方体现了这些东西呢？首先对于简单的新来的产品，**有些sprinboot进行了自动装配**，比如kafka等，**对于一些产品来说springboot的自动装配并不满足我们的需求（或者没有整合，没有自动装配比如生成验证码的kapcha组件），我们要重新/自己装配**，比如redis，自动装配的redis的key与value都是Object类型的，我们经常使用的是key是String类型，value是Object类型，这就需要我们利用@Configuration注解与@Bean注解进行重新装配。**而有一些产品生产太复杂，我们需要利用工厂**，比如quartz的配置，首先配置quartz的话需要配置JobDetails与Tirgger，而这两个类的配置太麻烦了怎么办，quartz与spring的整合中已经为我们想好了办法，拿JobDetails来说，建立一个JobDetailsFactoryBean类来实现FactoryBean，覆写获取bean的方法getObject()，这样复杂的创建逻辑就被整合框架封装好了，封装在了JobDetailsFactoryBean这个工厂里中，我们再打算建立JobDetails的时候，只要更改工厂里的一些属性就可以了！

## Spring的事务

Spring中声明式事务的开启很简单只要将方法上加入@Transactional注解就可以了，里边的参数指明事务的隔离级别，以及传播方式

## Synchronized锁升级的过程

答案来源：B站马士兵教育《因为一个volatile，阿里面试官对面试的Java程序员展开了。。。。。。》

1.对象包括以下4个部分，首先是**1.markword**，8个字节64位，里边记录了hashcode，gc年龄，锁信息；**2.类型指针**，4个字节32位，指向该对象对应的class的地址，我们使用的getClass()方法就是通过该指针；**3.成员信息**；**4.补齐字节**必须是8的倍数，比如我们在程序中建立Object o = new Object()那么这个对象占据16个字节，对象头(markword,类型指针)占据12个字节，补齐字节4个字节，一共16个字节；

2.锁信息记录在锁对象的markword里边，首先第一个线程获取到锁的时候，锁对象的markword的标志位为偏向锁；随后**如果有其他线程也要抢夺这把锁，那么这把锁就升级为自旋锁**，每一个线程都会进行自旋操作，试图通过cas方法将锁上的线程信息修改为自己的信息从而获取到锁。

3.在前面的JDK版本中1.7，**自旋数量超过10次，或者自旋的线程数超过upc核数的一般的时候就会升级为重量级锁**，这个可以通过jvm启动参数进行调优，现版本中都是自适应的不需要自己动手去调节了

4.自旋锁线程占用cpu，重量级锁需要操作系统参与，不占用cpu

## SpringBoot的自动配置原理

1.springboot是通过启动类上方的@SpringBootApplication来完成自动配置的，其中@SpringBootApplication注解又主要包含了@SpringBootConfiguration注解，@EnableAutoConfiguration注解与@ComponentScan注解

2.其中@SpringBootConfiguration注解表明这是一个配置类注解，@ComponentScan表明Spring会扫描配置类所属包下的及其子包下的所有bean进行管理，包括被@Controller，@Service，@Repository，@Component，@Bean注解修饰的类/方法等

3.而@EnableAutoConfiguration内部通过@Import注解导入了一个配置类AutoConfigurationImportSelector.class，而深入到这个类的内部最终会找到它提及到的spring-boot-autoconfigure包下的META-INF包下的spring.factories文件，里边提及了配置类的全类名，而默认的配置在也在这个包下的名为spring-configuration-metadata.json文件中，比如你想要使用的redis服务器的host配置，端口配置，这里都会找到默认配置

## 手写Autowired注解

类Service

```java
public class Service {}
```

类UserController

```java
public class UserController {

    @Autowired
    private Service service;

    public Service getService() {
        return service;
    }

    public void setService(Service service) {
        this.service = service;
    }
}
```

注解Autowired

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Autowired {}
```

通过反射使注解生效的Main方法(遍历写法一)

```java
import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.util.Arrays;

public class Main {
    public static void main(String[] args) throws IllegalAccessException, InstantiationException {
        UserController userController = new UserController();
        Class<? extends UserController> clazz = userController.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (int i = 0; i < fields.length; i++) {
            fields[i].setAccessible(true);
            Annotation annotation = fields[i].getAnnotation(Autowired.class);
            if(annotation!=null){
                Class<?> type = fields[i].getType();
                Object o = type.newInstance();
                fields[i].set(userController, o);
            }
        }
        System.out.println(userController.getService());
    }
}
```

通过反射使注解生效的Main(Stream流写法二)(下面的写法略去了异常的try与catch)

```java
import java.lang.reflect.Field;
import java.util.Arrays;

public class Main {
    public static void main(String[] args) throws IllegalAccessException, InstantiationException {
        UserController userController = new UserController();
        Class<? extends UserController> clazz = userController.getClass();
        Field[] fields = clazz.getDeclaredFields();
        //Stream.of(fields).
        Arrays.stream(fields).filter(field -> field.getAnnotation(Autowired.class)!=null)
                .forEach(field -> {
                        field.setAccessible(true);
                        Class<?> type = field.getType();
                        Object o = type.newInstance();
                        field.set(userController, o);
                });
        System.out.println(userController.getService());
    }
}
```

## SpringBoot怎么更改默认配置

1.在springboot的起步依赖中，我们打开pom.xml文件，里边会有

```xml
<parent>
   <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
   <version>2.3.0.RELEASE</version>
   <relativePath/> <!-- lookup parent from repository -->
</parent>
```

也就是说springboot规定所有的类一定要继承这个parent类，而在parent类中有

```xml
<build>
  <resources>
    <resource>
      <filtering>true</filtering>
      <directory>${basedir}/src/main/resources</directory>
      <includes>
        <include>**/application*.yml</include>
        <include>**/application*.yaml</include>
        <include>**/application*.properties</include>
      </includes>
    </resource>
```

也就是说springboot会扫描src/main/resources包下的\*\*/application\*.properties等文件，我们只需要在这里修改配置就可以了，比如我们更改redis的database就可以写为

```java
spring.redis.database=11
```

## 怎么样自己写一个starter

**说明：频率低，就蚂蚁金服面过一次**

|---首先建立一个lru-spring-boot-starter的空maven项目，只有pom.xml依赖于lru-spring-boot-autoconfigure包

|---建立一个使用maven的springboot项目，pom.xml中只保留spring-boot-starter-parent与spring-boot-starter，防止打jar时有没有必要的包进来，建立LRUAutoConfiguration类，这个类里边定义@Bean也就是我们以后要在springboot项目中能够自动注入的类对象，并在resource/META-INF/spring.factories文件中告知springboot这个组件的自动配置类为此类即LRUAutoConfiguration

|---一般还会有LRUProperties类，用于存储配置文件信息，似乎也用于接收用户的配置信息，我们使用此类的capacity属性来自定义LRU缓冲区的大小

|---使用maven工具，将lru-spring-boot-autoconfigure与lru-spring-boot-starter依次打包即可，使用时依赖lru-spring-boot-starter即可

3.举例子，具体怎么做。比如我们现在想自己写一个lru-spring-boot-starter，我们就要新建立一个空的maven项目，里边的pom.xml依赖lru-spring-boot-autoconfigure（现在这个jar包还不存在等建完这个补上）

![20200529223552](F:\.image\20200529223552.png)

3.通过start.spring.io创建lru-spring-boot-autoconfigure项目，并在pom.xml中删除过多依赖（不然也会将多余的依赖打包），只保留spring-boot-starter-parent与spring-boot-starter

![20200529224658](F:\.image\20200529224658.png)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.3.0.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <groupId>com.mycompanyname</groupId>
   <artifactId>lru-spring-boot-autoconfigure</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <name>lru-spring-boot-autoconfigure</name>
   <description>lru-spring-boot-autoconfigure</description>

   <properties>
      <java.version>14</java.version>
   </properties>

   <dependencies>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter</artifactId>
      </dependency>
   </dependencies>

</project>
```

4.建立LRUAutoCofiguration类，并在resource文件夹下META-INF文件夹下spring.factories文件中配置该类

```java
package packagename;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

//@EnableConfigurationProperties(LRUProperties.class)
//@Configuration
public class LRUAutoConfiguration{

    @Bean
    public LRUCache lruCache(){
        LRUProperties lruProperties = new LRUProperties();
        LRUCache lruCache = new LRUCache();
        lruCache.setLruProperties(lruProperties);
        lruCache.init();
        return lruCache;
    }
}
```

spring.factories文件中有：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=packagename.LRUAutoConfiguration
```

5.其中LRUProperties类与LRUCache类为

```java
package packagename;

import org.springframework.boot.context.properties.ConfigurationProperties;

//@ConfigurationProperties(prefix = "lru")
public class LRUProperties {

    private int capacity = 2;

    public int getCapacity() {
        return capacity;
    }

    public void setCapacity(int capacity) {
        if(capacity<1)
            throw new IllegalArgumentException("capacity必须大于等于1");
        this.capacity = capacity;
    }
}
```

```java
package packagename;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class LRUCache {
    private LRUProperties lruProperties;
    private Map<Integer, Integer> map = new HashMap();
    private List<Integer> list = new ArrayList();
    private int size;
    private int capacity;

    public LRUCache() {
    }

    public void setLruProperties(LRUProperties lruProperties) {
        this.lruProperties = lruProperties;
    }

    public void init() {
        this.capacity = this.lruProperties.getCapacity();
    }

    public int get(int key) {
        if (this.map.containsKey(key)) {
            this.list.remove(this.list.indexOf(key));
            this.list.add(key);
            System.out.println(this.map.get(key));
            return (Integer)this.map.get(key);
        } else {
            return -1;
        }
    }

    public void put(int key, int value) {
        if (this.map.containsKey(key)) {
            this.list.remove(this.list.indexOf(key));
        } else if (this.size < this.capacity) {
            ++this.size;
        } else {
            int keyToRemove = (Integer)this.list.get(0);
            this.list.remove(0);
            this.map.remove(keyToRemove);
        }

        this.list.add(key);
        this.map.put(key, value);
    }
}
```

之后就可以再一般的springboot项目中使用这个类了，只要通过@Autowired注入就可以了

后记补充：**Install是将包打到.m2库中的命令**

## SpringBoot怎么解决循环依赖的问题

**说明：频率低，就阿里菜鸟面过一次**

1.概括说的话，很简单。在spring中对象的实例化是通过反射实现的，而这大体分为两个步骤，第一步通过类的构造函数进行“半成品”的实例化，第二步进行属性的注入，完成全部实例化。知道了这个，循环依赖就很好理解了，类A有属性B，类B有属性A，那么实例化类A的时候，先进行了类A的半成品实例化，之后进行属性注入的时候发现类A依赖类B，便实例化类B(通过doGetBean函数)，进行完类B的构造函数初始化后，发现类B依赖类A，那么此时便将类A的引用注入进来就可以了，类B完全实例化之后再将引用赋值给类A属性就可以了，至此便完成了循环依赖对象的实例化

2.从源码来说的话，一个角度是这样子的。在实例化对象的时候会调用getBean方法，而getBean方法是一个重载的空壳方法，内部调用doGetBean方法，在doGetBean方法的内部会先调用一次（另一个类的）getSingleton方法，这个方法会先检查**singletonObjects容器**内部是否有该bean，这个容器内部装的是可以直接使用的bean，有些我们常说的spring容器的味道，但是并不是，这里有bean则直接返回，没有的话会检查**earlySingletonObjects容器**，这个容器内装的是半成品bean对象，有则直接返回，没有的话还会查看**singletonFactories容器**这个容器给我的感觉是装了能够生产bean的工厂**们**，我们根据bean name来获取对象的singletonFactory(ObjectFactory的子类？)如果这个工厂存在，则并根据它的getObject方法来实例化bean

3.在doGetBean中，如果第一次调用getSingleton方法没有获取到bean的话，就会调用重载的getSingleton()，并且在这个方法中我们会根据doCreateBean方法来创建半成品bean并且封装为singletonFactory（ObjectFactory）放入singletonFactories容器中，这样第二次调用doGetBean方法的时候就一定能在第一次调用getSingleton的时候获取到bean

4.这里2，3步是可以和1中对应起来的，对应起来以理解为什么要有那三个容器（**singletonObjects容器**，**earlySingletonObjects容器**，**singletonFactories容器**）

## Kafka的配置与使用

1.springboot也对Kafka进行了整合，有自动配置，我们项目中没有进行再配置，kafka的使用也是很简单的

1.生产消息：调用kafkaTemplate的send方法即可，第一个参数为主题，第二个参数为消息（我们一般称之为事件）的json字符串

```java
kafkaTemplate.send(event.getTopic(), JSONObject.toJSONString(event));
```

2.消费消息：在方法前加入@KafkaListener(topics={})标明该方法监听的主题，ConsumerRecord对象中包含了消息对象，我们使用JSONObject将其解析出来即可

```java
@KafkaListener(topics = {TOPIC_COMMENT, TOPIC_LIKE, TOPIC_FOLLOW})
public void handleCommentMessage(ConsumerRecord record) {
    Event event = JSONObject.parseObject(record.value().toString(), Event.class);
}
```

## 项目中kafka的使用的

1.项目中的系统通知是使用kafka来完成的，因为系统通知是用户a给用户b进行点赞，关注，或对用户b的帖子进行评论的时候才发生的，而系统通知无非是在message表中插入一条数据，这涉及到了数据库的操作，相对来说比较耗时间，如果在主线程中完成这一个操作可能会影响用户使用，而将这个事件加入到消息队列中等消费者线程来消费，虽然这样做会有一定的延时，但是是可以接受的，并且不会影响用户请求线程

2.具体使用：首先我们将传入消息队列中的事件对象定义出来为Event，我们要知道这个Event是要传入到消费者那边进行消费的，而消费者根据这些信息是要在message表中插入数据并且这些数据是能够支持页面显示的，页面显示的时候会有用户a对你的某某帖子进行了点赞，或者用户a对你的某某帖子进行了评论，或者用户a对你进行了关注，所以Event中要有userId也就是这里的用户a，帖子还是评论进行了点赞，无论帖子还是评论都是实体所以Event中要有实体entityType与entityId属性，这两个属性用于用户定位是哪个实体（帖子或评论）等等这些是Event的事情

3.接下来编写生产者代码，生产者代码比较简单，注入kafkaTmplate调用其中的send方法就可以了，发布到的主题从Event中取（点赞，评论或关注），第二个参数我们将Event对象转换为json字符串传入

4.接下里编写消费者代码，消费者监控点赞，评论，关注主题，这里使用@KafkaListener(topic=)注解就可以了，对应的方法参数上写ConsumerRecord这样的话该主题的消息就会被注入到方法的参数中，我们再进行json对象的解析，解析为Event再进行Message对象的封装，插入到message表中就可以了

## redis中的持久化方式

1.rdb方式是快照方式，他会把当前redis在内存中的数据存储到磁盘中，但是过程是堵塞的（实际上redis进程会fork子进程来完成这件事，用户请求只是会在fork子进程的时候会被阻塞），恢复较快

2.aof（Append only file）日志的方式，他是以追加的方式将执行过的redis语句添加到磁盘中，优点是速度快，缺点是文件一般占有的空间较大，恢复比较耗时，因为恢复是将语句再一条条执行一遍

## redis中键值对的过期方式

**redis中对过期的键的处理**

首先我们来了解一下redis中对过期的键的处理方式，redis中将所有设定了过期时间的key维护在一个map的结构中，每隔100ms选取20个key，将其中的过期的key进行删除，如果删除的key大于25%，那么继续选取20个key做这样的处理；另外如果使用者调用了一个key，但是redis发现这个key已经过期了，这是就会将这个key进行删除，并且不会返回任何结果，这就是所谓的**惰性删除**，上边的是**定期删除**

**淘汰策略**

我们知道redis是可以指定最大使用内存的，一旦最大内存不够用了怎么办，这时候就要使用淘汰策略，我们要淘汰一些键，常见的淘汰策略有：

1.noeviction不淘汰NoEviction，就是在内存不够用的时候直接进行报错处理

2.allkey-lru，最近最少被使用，这里是在所有的key中执行lru方式的淘汰

3.volatile-lru，最近最少被使用，这里是仅在过期的key中执行lru方式的淘汰

4.allkey-random,随机淘汰，这里是指在所有key中执行随机淘汰

5.volatile-random，随机淘汰，这里是仅在过期的key中执行随机淘汰

6.volatile-ttl，淘汰过期的key中存活时间最短的

redis后发行的版本又加入了其他淘汰方式

7.allkey-lfu，根据最少被使用频率淘汰，在所有key中按照此方式淘汰

8.volatile-lfu，根据最少被使用频率淘汰，这里是仅在过期的key中按照此方式淘汰

lfu和lru不同在于，比如有一个key很久没有被使用，仅仅是最近偶尔被使用了一次，那么lru算法就会认为该key是热点数据不删除该key，lfu则会根据频率进行删除

## redis中的事务

redis中的事务和mysql中的不一样，redis会将事务中的操作缓存到一个队列里边，等到事务提交的时候再一起执行，所以不要在redis事务中做查询（此处是指：此事务中的更改在此事务中的查询不可见）

```java
// 编程式事务
    @Test
    public void testTransaction() {
        Object result = redisTemplate.execute(new SessionCallback() {
            @Override
            public Object execute(RedisOperations redisOperations) throws DataAccessException {
                String redisKey = "text:tx";

                // 启用事务
                redisOperations.multi();
                redisOperations.opsForSet().add(redisKey, "zhangsan");
                redisOperations.opsForSet().add(redisKey, "lisi");
                redisOperations.opsForSet().add(redisKey, "wangwu");

                System.out.println(redisOperations.opsForSet().members(redisKey));

                // 提交事务
                return redisOperations.exec();
            }
        });
        System.out.println(result);
    }
```

**项目中使用redis事务的地方**

项目中我们的点赞功能使用redis实现，因为点赞是十分常用的功能，使用mysql性能跟不上去。点赞数据我们使用String类型和Set类型来存储，其中String类型的数据中我们用于存储某个用户a收到的赞的个数，有一个用户b给用户a点赞，那么我们就将key为用户a的id的值通过increament方法进行加1操作，其中Set类型的key为某个实体，比如帖子，比如评论，一旦用户b给用户a进行点赞，那么不仅String数据类型那里用户a收到的赞数会加1，对应的Set类型里会存入用户b的id，这样子的话，如果用户b再次进行点赞即取消赞，如果我们查询到Set集合中有用户b的数据，那么我们就会将用户b移除，并且将用户a收到的赞的个数使用decreament进行减1操作，所以Set集合和String这两个的操作是要具备原子性的，我们使用redis事务来保证这种原子性

## redis的自定义配置

**redis为什么要进行自定义配置**

springboot对redis的自动配置使用的key与value都是Object类型，这种方式适用面更广，但是我们一般使用的redis的key都是String类型的，这样使用起来就不太方便，所以我们进行自定义配置，将key配置成为String类型

**redis自定义配置的时候为什么要指定序列化方式**

java中的数据类型是不能够直接存储到redis数据库中的，必须进行一定的转化，所以我们要进行序列化

## redis可能出现的问题

答案来源：架构之路  《高级必问：什么是缓存穿透，缓存击穿，缓存雪崩，及解决方案分析》

**1.缓存穿透。**（记忆：“透”即透心凉，缓存和db中都没有该数据）也就是用户访问了缓存中没有的，数据库中也没有的信息。用户访问来了之后，发现想要的数据缓存中没有，于是去访问数据库，发现数据库中也没有，而数据库中没有的数据是不会加载到缓存中的。这样如果这种请求很多的话，那么请求就都会打到数据库中，造成数据库的宕机。

解决办法：1.将数据库中没有的数据也在缓存中进行缓存，对应的value为null，存活时间设置为短一点的时间比如5min（因为缓存大小有限，所以该信息要设置比较小的存活时间）。这样请求来了之后就会从缓存中知道数据库中没有该信息，就不会访问数据库了。2.使用布隆过滤器，将数据库中有的信息映射到一个bitmap上，这样请求就根据这个bitmap就知道该数据数据库上有没有

**2.缓存雪崩。**同一时刻，缓冲中的数据大面积失效，导致所有的请求直接打到了数据库上。

解决办法：缓存中的数据的过期时间不要设置成相同的，设计一个随机的过期时间，或者固定时间加上一个随机时间等，总之不要让缓存中的数据同时过期

**3.缓存击穿。**缓存中的某个热点数据过期，同一时刻好多的请求要访问这个热点数据，这些请求并发打到服务器上，服务器可能就会出现问题。

解决办法：使用互斥锁（博主注：这似乎也是分布式锁的实现方式）

```java
public String get(key) { 
 String value = redis.get(key); 
 if (value == null) { //代表缓存值过期
 //设置3min的超时，防止del操作失败的时候，下次缓存过期一直不能load db 
 	if (redis.setnx(key_mutex, 1, 3 * 60) == 1) { //代表设置成功
 		value = db.get(key); 
 		redis.set(key, value, expire_secs); 
 		redis.del(key_mutex); 
 		} else { //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可
 			sleep(50); 
 			get(key); //重试
 		} 
 } else { 
 	return value; 
   } 
}
```

## 线程池的拒绝策略

JDK内置提供了四种拒绝策略

**1.AbortPolicy策略：**直接抛出异常

线程池默认的拒绝策略；该策略会直接抛出异常，任务不会被执行

**2.CallerRunsPolicy策略：**只用调用者所在线程来运行任务(博主注：如果任务比较耗时，可能会造成当前线程堵塞！)

只要线程池未关闭，该策略直接在调用者线程中，运行当前被丢弃的任务，但是，任务提交线程的性能极有可能会急剧下降

**3.DiscardOldestPolicy策略：**丢弃队列中即将执行的任务，并执行当前任务

该策略将丢弃最老的一个请求，也就是就是即将被执行的一个任务，并尝试再次提交当前任务

**4.DiscardPolicy策略：**不处理，直接丢弃

该策略默默地丢弃无法处理的任务，不予任何处理。如果允许任务丢失，这可能是最好的一个建议方案

博主注：CallerRunsPolicy没有抛弃任务

## 模拟并发测试的工具

答：模拟并发测试的工具有很多，项目中使用到的是apache的jmeter

**具体使用**

答：项目中我们的热点帖子使用Caffeine这个工具来进行缓存的（当然如果使用redis或者加上redis做二级缓存也是完全可以的），当调用Service层的查询热帖的功能的时候，我们会先在缓存中查看是否有我们要查询的数据，如果有的话那么我们就会直接从缓存中读取数据，没有的话我们会从数据库中查询这些数据并且缓存到Caffeine中

1.那么没加Caffeine与加了Caffeine性能有多少提升呢？我们使用jmeter来测试一下，首先我们建立线程组，也就是告诉测试软件，我们**要生成多少线程**（比如100个，这是因为自己的电脑性能不高，所以比较少），**这些线程在多长时间内生成完毕**（比如1秒），**这些线程循环请求多少次**（比如不间断地请求持续60秒）

2.接下来我们我们添加http相关的设置，就是说你是**通过什么协议访问的**（http），**使用什么方式访问的**（比如热帖的访问方式应该为get），**访问的主机（127.0.0.1），端口号(8080)，url是多少(community/index?model=1)**等其中model=1表示的是请求帖子页为热点帖子页

3.接下来就随机定时器，就是说各个请求不能够持续的打过来，要有一点**时间间隔**，比如我们设置为0-1000毫秒的随机间隔

4.接下来是聚合报告，也就是访问结果或者说访问性能的报告，主要**关注吞吐量指标**，发现没有缓存之前吞吐量为近10次/秒，也就是说每秒处理大概10个左右的请求，加上缓存之后，每秒可以处理大概170，180个这样一个数量的请求，也就是说两者不在同一个数量级上！差了近20倍！！！

## 手写LRU

```java
import java.util.HashMap;
import java.util.Map;

class LRUCache {

    private Node head = new Node();
    private Node tail = new Node();
    private int size = 0;
    private int capacity = 0;
    private Map<Integer, Node> map = new HashMap<>();

    public LRUCache(int capacity) {
        head.next = tail;
        tail.pre = head;
        this.capacity = capacity;
    }

    public int get(int key) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            update(node);
            return node.value;
        } else {
//这返回-1还是不太好的，万一有人设置的value值就是-1呢？你该怎么区分是有这个key对应的值是-1呢还是没有这个值呢？
            //当然这里返回-1可能是leetcode要求的返回
            //这里建议将返回值类型更改为Integer，当无key的时候返回null
            return -1;
        }
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            delete(node);
            node.value = value;
            push(node);
        } else {
            Node node = new Node(key, value);
            if (size + 1 > capacity) {
                map.remove(tail.pre.key);
                delete(tail.pre);
            }
            push(node);
            map.put(key, node);
        }
    }

    class Node {
        int key;
        int value;
        Node pre;
        Node next;

        Node() {
        }

        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private void delete(Node node) {
        //这的判断很好！！！虽然可能并不会发生
        if (node == null || node.pre == null || node.next == null)
            throw new IllegalArgumentException("参数异常！");
        node.pre.next = node.next;
        node.next.pre = node.pre;
        size--;
    }

    private void push(Node node) {
        //这的判断很好！！！虽然可能并不会发生
        if (head == null || node == null || head.next == null)
            throw new IllegalArgumentException("参数异常！");
        Node temp = head.next;
        head.next = node;
        node.pre = head;
        node.next = temp;
        temp.pre = node;
        size++;
    }

    private void update(Node node) {
        delete(node);
        push(node);
    }
}

```

## SpringMVC的工作原理

![20200717110710](F:\.image\20200717110710.png)

（以下是根据项目理解，不一定正确）

1.SpringMVC分为三个主要的组件，分别为Controller，Model，View。

2.当用户请求来了之后，DispatchServlet组件会将请求根据每个Controller中的RequestMapping(里边的访问路径)分配到对应Controller组件中的方法中。

3.这个方法会通过访问Service层面的方法，Service层面再通过调用dao层面的方法，最终从数据库中查询到相应的数据，并且封装为Model对象，返回给DispatchServlet。

4.DispatchServlet会将Model发送给View视图层，视图层通过模板引擎（我们项目中使用的是thymeleaf模板引擎）利用视图模板，再将Model中的数据放到这个视图模板相应的位置中，即完成整个的html网页，再返回给DispatchServlet。

5.DispatchServlet将html网页返回给用户，用户浏览器进行渲染就会呈现出网页

## 只拦截Get请求

(项目理解，不一定对)

在SpringBoot中我们如果要拦截请求的话，需要两个步骤（利用的是SpringMVC组件的拦截请求功能）

1.建立拦截器类实现HandlerInterceptor接口，这个接口里边的方法都是有默认实现的，我们可以根据自己的需要来选择进行实现或者说覆写，如果我们要拦截get请求的话，我们可以实现preHande方法，而这个方法参数中是有这个HttpServletRequest参数的，也就是有请求对象的，这个对象中也一定有可以获取到该请求方式的方法，比如说为getMethod()，那么我们就可以根据request.getMehtod()方法的返回值来判断请求方式是不是get，如果是get的话，那么我们就进行拦截之后的逻辑（比如说打印一条日志），不是的话就不做任何处理(请求该怎么走就怎么走)

2.建立拦截器类之后，我们还需要建立配置类使这个拦截器类生效，或者说是告诉SpringBoot这是一个拦截器类，我们建立配置类使用@Configuration注解修饰，并且实现WebMvcConfigurer接口，实现里边的addInterceptors方法，在这个方法里边通过registry.addInterceptor方法将拦截器类添加进去，这就是告诉了SpringBoot，我们自定义了一个拦截器类，你要去执行它的逻辑

## 显示登录信息:拦截器

1.我们登录网站的时候会发现网站的最上面的部分，我这里称之为状态栏，状态栏会有首页，注册，登录几个选项，但是如果我们已经登录了，这个状态栏就要变化，变化为首页，消息，个人头像这几个部分，点击个人头像，还会有账号设置，个人主页，退出这几个选项，根据用户是否登录而动态显示状态栏信息的功能就是这里所说的显示登录信息，或者说是检查登录状态

2.具体实现。首先需要我们写配置类WebMvcConfig实现WebMvcConfigurer接口，实现其中的addInterceptors方法，将拦截器类对象loginTicketInterceptor添加进去，配置该拦截器不需要拦截哪些请求（一般获取静态资源请求不需要拦截，包括头像资源等）

3.其次我们来写LoginTcketInterceptor类，该类需要实现HandleInteceptor接口，在这个类里边我们要覆写preHandle方法，postHandle方法与postCompletion方法，这三个方法分别在请求的controller方法之前执行，之后执行（并在调用模板处理html之前），调用完ThymeleafTemplate之后执行。

4.在preHandle方法中，我们通过HttpServletRequest方法获取请求中key为“ticket”的值，并根据这个值上redis中查找对应的login_ticket对象，试图获取其中的userId，如果获取成功，我们就将其（User）放置在HostHolder中（这个HostHolder类中维护了一个ThreadLocal容器），这个是我们在controller中调用当前登录用户，维持登录状态使用的。

5.在postHandle中我们将HostHolder中的User对象加入到视图modelAndView中（名为LoginUser），这样在html中我们就可以根据loginUser的值，来判断当前登录用户是谁/是否登录了。

6.在postHandle中，我们在HostHolder中删除该User对象，防止对象越来越多，撑爆内存

## AOP是什么

AOP是面向切面编程，在没有AOP的时候，如果我们想要在Service层的所有方法之前加上一个功能，就是一个记录日志的功能，该日志记录用户某某在某某个时间点访问了某某方法，如果没有AOP的话，那么只能够将该逻辑封装为一个方法，并在所有的Service层方法之前调用该方法，但是这样做工作量是十分巨大的而且并不优雅，有了AOP之后，类似于我们做了一个拦截（本质上是动态代理），拦截所有的Service层方法，并在拦截逻辑中实现记录日志的业务逻辑即可

**为什么不能够使用SpringMVC的拦截器来做这个日志的功能呢？**

因为SprigMVC的拦截器是工作在Controller层面的，也就是视图层，但是我们想要记录日志的地方是在Service层面上，所以并不能够使用SpringMVC的拦截器来做

## AOP组件的使用

注意：多数考察为Aop的实现原理，那就是动态代理了

这里给出aop的具体代码，看着说

```java
package com.nowcoder.community.aspect;

@Component//交由springboot容器管理
@Aspect//表明这不是一个普通的组件，这是一个Aspect组件
public class ServiceLogAspect {

    //因为我们想要在切点前记录日志，所以创建日志对象
    private static final Logger logger = LoggerFactory.getLogger(ServiceLogAspect.class);

    /*定义切点，该切点为com.nowcoder.community.service下的所有类的所有方法，不限返回值，不限方法的参数列表*/
    @Pointcut("execution(* com.nowcoder.community.service.*.*(..))")
    public void pointcut() {
    }

    /*表明该方法在切点前调用，这里记录的日志为用户***在何时访问了何方法*/
    @Before("pointcut()")
    public void before(JoinPoint joinPoint) {
        /*用户[1.2.3.4],在[xxx],访问了 [com.nowcoder.community.service.xxx()].*/
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        
        String ip = request.getRemoteHost();
        String now = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
        String target = joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName();

        logger.info(String.format("用户[%s],在[%s],访问了[%s].", ip, now, target));
    }
}
```

## 项目中的表

答：项目中一共16个表，其中11个表是Quartz相关的

1.首先是**user**表包含了**id**，**username**（varchar类型），**password**，**salt**，**type**（0表示普通用户，1表示超级管理员，2表示版主），**status**（0表示未激活，1表示激活），**activation_code**，**header_url**，**create_time**（timestamp类型）

2.**discuss_post**帖子表，包括**id**，**user_id**（也就是谁发的帖子），**title**（帖子的名字），**content**（text类型，帖子的内容），**type**（0表示普通帖子，1表示置顶帖子），**status**(0表示正常，1表示加精，2表示拉黑，也就是删除了帖子)，**create_time**创建时间，**comment_count**（冗余数据段）查询的话比较方便

3.**comment**评论表，评论表包括**id**，**user_id**也就是该条评论的发起人是谁，**entity_type**实体类型，如果该条评论评论的是帖子，那么entity_type的值为1，如果该条评论评论的是评论（我们这里称之为回复），那么entity_type的值为2，**entity_id**，如果entity_type为1的话，那么这个id指向的就是discusspost表中的帖子id，如果entity_type为2，也就是说这是一个回复的话，那么这个entity_id指向的就是comment表中的评论id，**target_id**，你的回复是对谁回复的，这个是用户id，**content**就是评论的内容，**status**评论的状态（比如正常啊，拉黑啊等），**create_time**创建时间

4.**message表**，站内信和系统通知所用的表，包括**id**，**from_id**，**to_id**，**conversation_id**（这个的话是通信的一个标识，标识是哪两个人进行的通信，无论是张三发给李四消息，还是李四发给张三消息，都是同一个conversation_id，这样的话，收到某个人的私信后，点开，那么 就是根据conversation_id查询的两个人的历史往来通信列表），**content**，**create_time**，**status**（0表示未读，1表示已读，2表示删除）

5.**login_ticket**身份标识表，这个后期优化的时候作废了，前期的话，用户的凭证信息是存储到这张表里的，表里边有基本的**id**，**user_id**，**ticket**，**status**，**expired**信息，就是说用户请求来了之后会携带之前服务器给他生成的用户凭证，我们从request请求对象中取到key为“ticket”的cookie之后，会从这张表中获取当前用户信息，维持登陆状态，但是每一个请求来了之后都会要访问这个数据库，来判断是不是在线（有没有退出登录，status标志是否为1），是不是凭证没有过期（expired字段来判断），这种查询是频繁使用的，为了提高性能，后期我们将这是数据缓存到了redis中

6.其余的11张表是quartz相关的内容

## 数据库的三大范式

**第一范式**。列不可再分，比如现在有一张记录个人信息的表，其中有一个地址属性，如果只列为一个地址的话可能就是可再分的，如果分开为省，市，县等列，那么粒度是更小的，如果要统计某市人口等信息后者也更加容易，当然这点还是要根据业务来看粒度要分为多细（说法不一定正确，不同帖子给出的答案不尽相同，这里只给出了比较好理解的答案）

**第二范式**。某些列只能依赖于同一个列，如果不同列依赖不同的列，那么就不满足第二范式了。比如现在有数据表字段为id，学生姓名，学号，课程名，课程学分，很明显这张表就不符合第二范式，因为学号列依赖于学生姓名列，而课程学分列却依赖于课程名，这样如果某些个学生选了同一个课程或者某一个学生选了多门课程都会出现数据冗余，应该拆分为三个表，（学生姓名，学号），（课程名，课程学分），（学号，选课）这三个表

**第三范式**。其他列必须直接依赖于主键列，不能够传递依赖。比如现在有（学生姓名，学号，所属学院，学院地址，学院电话）这样就存在学院地址，学院电话--->学院名称--->学号，这样也有可能出现数据冗余问题的，我们将其分为三个表即可(学生姓名，学号)，（学院名称，学院地址，学院电话），（学号，所属学院）三张表

## 手撕快排

1.快排最坏的时间复杂度为O(N^2)，情况是每一次选取的pivot都是数组的最大或最小元素，比如选取的是数组第一个数作为基准数pivot，但是数组是正序或者逆序排列的

2.快排的最好时间复杂度为O(NlogN)，情况是每一次的基准数都是中间的数字

3.快排的平均时间复杂度为O(NlogN)

4.快排是不稳定的排序算法，归并是稳定的排序算法，但是所有的不稳定的排序算法都可以在代码上进行一定处理，使其稳定

```java
class MyArrays {

    public static void quickSort(int[] arr) {
        quickSort(arr, 0, arr.length - 1);
    }

    public static void quickSort(int[] arr, int start, int end) {
        if (start >= end)
            return;
        int mid = findPrivot(arr, start, end);
        quickSort(arr, start, mid - 1);
        quickSort(arr, mid + 1, end);
    }

    private static int findPrivot(int[] arr, int start, int end) {
        int pivot = arr[start];
        int left = start;
        int right = end;
        while (left != right) {
            while (left < right && arr[right] > pivot)
                right--;
            while (left < right && arr[left] <= pivot)
                left++;
            if(left<right) {
                int temp = arr[left];
                arr[left] = arr[right];
                arr[right] = temp;
            }
        }
        int temp = arr[left];
        arr[left] = arr[start];
        arr[start] = temp;
        return left;
    }
}
```

## 输入url发生了什么

(不一定正确)

1.输入url之后，会依次查询本机host文件，dns服务器获取目标ip地址，浏览器根据http协议生成指定格式的数据，但是会暂时阻塞，调用内核方法请求建立连接

2.调用核方法，数据传输层会封装了三次握手需要的相关数据，比如seq字符串，SYN标志位为1等

3.网际层封装源和目的ip地址，路由器工作在这一层，内核会根据路由表计算网关地址（Genmask与目标ip相与，如果得到的是Destination的ip那么就走对应的Gateway地址）

4.这个Gateway地址是封装到下一层的也就是链路层，交换机工作的层级，本机如果有Gateway的MAC地址就直接封装，如果没有则通过ARP协议进行获取，也就是发出广播问谁有Destination的目的MAC

5.封装之后发送给服务器端，服务器端再进行一层层的解析，之后获取握手数据后再做相应的返回，三次之后连接确立，html格式的文件被发送过来

6.接下来就是服务器处理的过程，可以答一波《SpringMVC的工作原理》

**输入url发生了什么第二版**

1.输入url之后首先会进行解析，根据输入的url来生成http报文，比如你的请求方法是什么，你的请求的资源是什么，使用的是http协议还是https协议，用的第几版的协议，该浏览器支持的压缩文件类型是什么等等，这些信息就生成了请求行，请求头，请求体信息，浏览器就会调用内核的方法来进行下一步的操作

2.在进行下一层的处理之前，还要获取目的地址的ip，这个浏览器会查找自身缓存，本机host文件，如果都没有的话会向dns服务器发出请求，请求经过域名根服务器，顶级域名服务器等最终查询到目的地址的ip

3.下面进入网际层，该层通过tcp协议进行处理，tcp协议包含源端口号，目的端口号，序号（用来保证发送顺序），确认序号，ACK，SYN，FIN标志位，窗口大小（用来标明自己处理信息的能力大小）等，tcp给http报文添加头部之后，进入下一层处理

4.下一层是传输层，该层通过ip协议进行处理，ip协议包含了源地址ip和目标地址ip，这些信息在整个传输过程中都不会改变。路由器工作在这一层中，通过路由表（route -n命令查看），通过将目的ip和子网掩码相与并和destination比较我们会知道目的ip应该发往哪一个网关。

5.下一层为数据链路层，交换机工作在这一层。我们通过arp缓存查询是否记录了该网关的mac地址，如果没有的话，我们会通过arp协议在局域网内发送广播，询问谁有网关ip的mac地址。知道目的地址的mac之后，我们在这一层封装mac头部，包含源mac地址，目的mac地址，再通过物理层向外发送

6.网络节点中的交换机收到数据包之后，会查询本机的mac地址与对应端口号的位置，并将数据包发送给对应的端口号，如果没有的话，交换机会将数据包广播。

7.节点中的路由器收到数据包之后，会将mac层打开，并根据它的路由表重新进行mac头部封装等工作。

8.最终数据包会发送到服务器的网卡中，再经过一层层的打开，最终会传输到应用层，达到信息沟通的目的

## linux的指令

**建立tcp连接**

```shell
exec 8<> /dev/tcp/www.baidu.com/80
cd /proc/$$/fd
echo -e "GET / HTTP/1.0\n" 1>& 8
cat 0<& 8
```

1.其中exec是shell中的方法，可以执行其他程序替换shell程序或者向上述程序一样创建一个socket连接，这个连接连接的是百度主机的80端口，linux会把这个socket连接映射为一个文件，称之为文件描述符，8是文件描述符的标识，<标识输入，>标识输出，<>表明这个socket既可以输入又可以输出

2./proc虚拟文件系统，将内核与进程状态归档为文本文件**（系统信息都存放这目录下）**，其中$$表示当前的进程号（即shell进程），fd表示与该进程相关的文件描述符

3.echo -e "GET / HTTP/1.0\n" 1>& 8其中-e参数将\n识别为换行，>为输出重定向，1为输出文件描述符

7cat 0<& 8 其中0表示输入文件描述符，8表示先前建立的socket连接，&表明8不是一个普通的文件而是一个socket连接标识

**下面我们尝试和redis建立一个tcp连接，并且发送消息**

```shell
ps -fe | grep redis                   //找出redis进程信息，获取端口号信息
exec 88<> /dev/tcp/localhost/6379    //建立tcp连接，建立socket（四元数组，标识连接）
cd /proc/$$/fd                       //进入shell进程的文件描述符文件夹中
echo "set name zhangsan" 1>& 88      //通过连接通信，建立一个key为name，value为zhangsan的键值对
echo "get name" 1>& 88               //通过tcp连接取key为name的数据
cat 0<& 88                           //将88发回来的信息打印一下
```

我们看到redis的协议没什么特别的东西，就和执行redis-cli写语句一样！！！

## 读写锁

要点：读锁对读锁并不互斥，写锁对读锁及写锁互斥

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class Main {
    private static int source = 0;

    public static void main(String[] args) throws InterruptedException {
//        ReentrantLock lock = new ReentrantLock();

        ReentrantReadWriteLock reentrantReadWriteLock = new ReentrantReadWriteLock();
        Lock readLock = reentrantReadWriteLock.readLock();
        Lock writeLock = reentrantReadWriteLock.writeLock();

        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Main.read(readLock);
                }
            }).start();
        }
        for (int i = 0; i < 2; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Main.write(writeLock);
                }
            }).start();
        }

    }

    public static void read(Lock lock) {
        lock.lock();
        try {
            System.out.print("reading...");
            Thread.sleep(1000);
            System.out.println(source);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void write(Lock lock) {
        lock.lock();
        try {
            System.out.print("writing...");
            Thread.sleep(1000);
            System.out.println(++source);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

使用ReentrantReadWriteLock的时候读锁与读锁并不互斥，对于读多的场景来说很好地提高了性能

## CountDownLatch与CyclicBarrier

**CountDownLatch**可以保证某线程在某些线程之后完成，比如下面例子中主线程堵塞在countDownLatch.await()方法中，等待另两个线程完成才继续执行

```java
import java.util.concurrent.CountDownLatch;

class Main {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(5000);
                    System.out.println("run");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    countDownLatch.countDown();
                }
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    System.out.println("run");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    countDownLatch.countDown();
                }
            }
        }).start();
        countDownLatch.await();
        System.out.println("finish");
    }
}
```
**CyclicBarrier**循环栅栏，给定一个数字，一旦等待的线程数目等于该给定数字，那么放倒栅栏，这些个等待线程一起执行，适用于需要多线程的操作结果才能继续执行的场景

```java
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

class Main {
    public static void main(String[] args) throws InterruptedException, BrokenBarrierException {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> System.out.println("车满，发车啦！"));
        for (int i = 0; i < 15; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(new Random().nextInt(10000));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    try {
                        System.out.println("上车");
                        cyclicBarrier.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        System.out.println("finish");
    }
}
/*
finish
上车
上车
上车
上车
上车
车满，发车啦！
上车
上车
上车
上车
上车
车满，发车啦！
上车
上车
上车
上车
上车
车满，发车啦！
*/
```

## redis的哨兵模式

说明：可参考B站视频服用

redis的哨兵模式是redis高可用问题的解决方案

1.redis集群通常为一主多从模式，也就是一个主服务器作为写的服务器，另外的服务器作为读的服务器，这样的话就实现了redis读写分离，分担了redis集群的读的压力

2.如果redis挂了要怎么办呢？将其中的一台slave从服务器作为主服务器使用slaveof no one命令，就是将该redis作为master，其他slave使用slaveof host port 命令变为该master的slave比如slaveof 127.0.0.1 6380 就是将本redis作为127.0.0.1主机的6380的redis的从服务器

3.哨兵模式下，哨兵帮我们做了这个工作，哨兵的工作主要是三个部分，第一是监控，第二是通知，第三是故障切换

4.**监控**，首先哨兵sentinal（读音山t弄）会向master发送信息，获取master及master下面的slave的状态信息，包括master的runid，slave的host及port以及runid等，同时为了通信方便，两者之间会存在cmd连接。slave通过从master处获取的信息会去连接slave，各个sentinal之间也会建立连接以达到消息共享

5.**通知**，sentinal会定时向master发送信息以查看master是否存活，并将信息发布传递给所有的sentinal

6.**故障切换**，当某个时刻某个sentinal检测出master死掉了，那么就会将这个master标记为sdown状态，并通知给其他的sentinal，当超过半数的sentinal都检测此master死掉了的话，就会标记该master为odown状态，就会进行选举机制。**首先**sentinal会进行内部选举，选出代表去主从集群中做重新选举master这件事情，sentinal会向其他sentinal发送信息，这个信息携带了自己的标识，自己参与的投票次数等信息，其他的sentinal会将选票投递给最先收到的那个sentinal这样的话一轮或几轮下来就会选举出sentinal中的代表。**其次**，这个代表会根据一些原则，比如节点必须在线，节点不能够回复时间过慢，与上一个master的最后断开时间不能够过长，offset要小，runid要小等特征选取出master，**最后**，将该节点与上一届的master连接断开，设置他为新的master，并将其他的slave的master设置为该节点！

7.题外话：一般我们一台机子上可能有多个redis比如有4个，但是sentinal的话我们一般一台机子上设置一个就可以了；sentinal也是redis数据库，但是并不提供数据服务；master上可读可写，slave上只能够读，不允许有写的操作！

8.启动sentinal的话与启动redis类似，使用redis-sentinal 某某.conf文件就可以，该配置文件中有sentinal的端口号与主机的host与port信息，而启动redis主从的话都为redis-server 某某.conf

## intern方法

1.intern方法就是就是查看当前SCP（字符串常量池）中有没有该字符串对象，如果有的话，那么直接返回该对象的内存地址；如果没有的话，在字符串常量池中建立这样一个对象，并且返回其内存地址

2.直接赋值的情况都是会先去字符串常量中查看是否有该对象的，如果有直接返回其内存地址，如果没有那么建立一个并返回其内存地址

```java
import java.util.concurrent.BrokenBarrierException;

class Main {
    public static void main(String[] args) throws InterruptedException, BrokenBarrierException {
        String s1 = "a";//常量池中没有，建立一个并且返回其地址
        String s2 = "a";//常量池中已经有了，直接返回其地址，所以s1==s2返回为true;
        String s3 = new String("a");//见到new关键字就是在堆中新建立一个对象
        String s4 = new String("a");//见到new关键字就是在堆中新建立一个对象
        System.out.println(s1==s2);//地址一致当然返回true
        System.out.println(s2==s3);//一个是SCP里边的地址，一个是堆中的地址，当然返回是false
        System.out.println(s3==s4);//new出来的两个对象，地址当然是不一样的false
        s1 = s3.intern();//在SCP中新建立一个和s3对象equals的对象，并返回该地址
        System.out.println(s1==s3);//但是肯定还是不等的，因为s3指向的仍然是堆内存中的地址false
        s3 = s3.intern();//将SCP中与s3对象equals的对象的地址返回给了s3，那么自此s3指向也就变化了，指向了SCP中的对象
        System.out.println(s1==s3);//true
    }
}
/*
true
false
false
false
true
*/
```

## for循环嵌套优化

请优化下列代码：

```java
for(int i = 0; i<1000;i++){
    for(int j = 0;j<1000;j++){
        System.out.println(i+j);
    }
}
```

分析：

1.如果小伙伴没有接触过这个题目，肯定会有点懵，O(n^2)的时间复杂度，我使用空间换时间肯定可以减少时间复杂度的，但是你这个方法是实现啥功能的啊？

2.其实面试官是想考你另外一个点，在上述for嵌套的程序中，请问j被初始化了多少次呢？注意是初始化次数，显然i被初始化了一次而j被初始化了1000次，那么优化的思路就是让j也只初始化一次

```java
int j;
for(int i = 0;i<1000;i++){
    for(j=0;j<1000;j++){
        System.out.println(i+j);
    }
}
```

## finalize方法

答：finalize方法是垃圾回收器在释放垃圾的内存空间之前调用的方法，完成释放资源等操作，比如释放文件资源等；但是调用的时机具有不确定性，一般地我们也并不用它进行关闭资源的操作

1.不确定是指如果一个对象变得不可达了，但是并不会被立即回收，而是等待空间不够的时候才会被回收从而触发finalize方法，下面的程序就是finalize没有被触发的例子

```java
public class Main {
    public static void main(String[] args){
        Main main = new Main();
        main = null;
    }
    @Override
    public void finalize(){
        System.out.println("我没了");
    }
}
```

2.我们可以使用System.gc()方法强制虚拟机进行垃圾回收操作，以触发finalize方法

```java
public class Main {
    public static void main(String[] args){
        Main main = new Main();
        main = null;
        System.gc();
    }
    @Override
    public void finalize(){
        System.out.println("轻轻地挥一挥衣袖，不带走一片云彩！");
    }
}
```

3.在finalize方法中也可以完成对象的自我救赎，就是说在finalize方法中使root引用到了该对象

```java
public class Main {
    private static Main main = null;
    public static void main(String[] args) throws InterruptedException {
        Main main = new Main();
        System.out.println(main);
        main = null;
        System.gc();
        Thread.sleep(1000);//不加这个的话会使得finalize方法还没执行就执行了sout方法导致Main.main为null
        System.out.println(Main.main);
    }
    @Override
    public void finalize(){
        System.out.println("我没了");
        Main.main = this;
    }
}
/*
Main@10f87f48
我没了
Main@10f87f48
*///我们看到对象并没有被回收，原因是该对象的finalize方法中使一个对象引用指向了自己!
```



4.《深入Java虚拟机书》中说finalize方法很类似于析构函数，是为了c/c++程序员更容易接受java语言所做的妥协

## 数据库表设计

要求：

```java
课程系统数据表设计
教务处 排课表
 
学生 学号，姓名
老师 工号，姓名
教室 名字，位置，容量
课程 名称，介绍
时间
 
一个学生，可以选多门课
一个老师，可以教多门课
一门课程，可以被多个老师教
教室，一个教室在同一时间只能有一个班在上课
 
一门课，在一周内可以上多节
 
每个老师带的每一门课，要有老师对这门课的介绍
```
表设计：
```java
学生表：id，学号，姓名
教师表：id，工号，姓名
教室表：id，名字，位置，容量
课程表：id，名称，介绍
教师授课表：id，教师表id，课程表id，老师对课程介绍
教室分配表：id，教室表id，时间，教师授课id
学生选课表：id，学生表id，教室分配表id
```

## 进程通信

1.匿名管道 **管道**（有名管道）例子：

 	命名管道mkfifo test  

 	向管道中输入数据echo "hello" > test  

​	另一个进程读数据cat <test

2.写完数据就堵塞了，有没有解决方式？**消息队列**，进程a向消息队列中写入数据，进程b读取数据，写完数据直接返回程序，并不堵塞，但是如果传输的数据比较大，那么频繁的数据拷贝就不适合使用消息队列了，因为频繁数据拷贝耗费时间

3.**共享内存**，进程之间内存独立，但是独立的是虚拟内存，我们完全可以让通信之间的进程的虚拟地址映射到同一个物理内存上，这样的话就不存在数据拷贝消耗时间的问题了

4.**信号量**，使用共享内存的话肯定就会出现进程之间的安全问题，就类似于线程安全问题，我们使用信号量来解决，进程a使用共享内存的时候将信号量设置为0，进程本来的时候发现信号量不是1也就知道其他进程在使用共享内存，所以说信号量也是进程之间通信的一种方式

5.**Socket**，之前讨论的都是本机之间的各个进程之间的通信，那么如果要是网络上相距很远的进程进行通信呢？这就要使用到socket了

## 线程通信

启用两个线程，一个线程输出1，2，3……26，另外一个线程输出A，B，C……Z，输出的结果为1A2B……26Z

### ReentrantLock

**方法一，使用ReentrantLock锁**

注意：下面这种写法是可能出现A1B2……Z26这种情况的，我们可以使用CountDownLatch类使得输出数字的线程先执行，具体写法不再给出，可参考synchronized实现时候的CountDownLatch，都是一样的用法

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class Main {
    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        Condition letter = lock.newCondition();
        Condition number = lock.newCondition();
        new Thread(() -> {
            lock.lock();
            int start = 1;
            while (start <= 26) {
                System.out.print(start++);
                try {
                    letter.signal();//注意一定是先唤醒signal再等待await，不然都等待了还怎么唤醒？（调用await方法后就不继续执行了，但是释放锁了）
                    number.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                letter.signal();//注意没有这句话的话，程序就结束不了了
            }
            lock.unlock();
        }).start();
        new Thread(() -> {
            lock.lock();
            int start = 'A';
            while (start <= (int) 'Z') {
                System.out.print((char) start++);
                try {
                    number.signal();
                    letter.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            lock.unlock();
        }).start();
    }
}
```

### synchronized

**方法二，使用synchronized同步关键字**

```java
import java.util.concurrent.CountDownLatch;

public class Main {
    public static void main(String[] args) {
        Object o = new Object();
        CountDownLatch countDownLatch = new CountDownLatch(1);
        new Thread(() -> {
            try {
                countDownLatch.await();//注意要写在同步关键字外边，不然等待也不释放锁，另一个线程也会等待，程序锁死
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (o) {
                int start = 'A';
                while (start <= (int) 'Z') {
                    try {
                        System.out.print((char) start++);
                        o.notify();
                        o.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
        new Thread(() -> {
            synchronized (o) {
                int start = 1;
                while (start <= 26) {
                    System.out.print(start++);
                    countDownLatch.countDown();
                    try {
                        o.notify();
                        o.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    o.notify();
                }
            }
        }).start();
    }
}
```

### 自旋

**方法三，使用自旋方式**

```java
import java.util.concurrent.CountDownLatch;

public class Main {
    private static volatile int state = 0;

    public static void main(String[] args) {
        new Thread(() -> {
            int begin = 'A';
            while (begin <= (int) 'Z') {
                while (state != 0) {
                }
                System.out.print((char) begin++);
                state = 1;
            }
        }).start();
        new Thread(() -> {
            int begin = 1;
            while (begin <= 26) {
                while (state != 1) {
                }
                System.out.print(begin++);
                state = 0;
            }
        }).start();
    }
}
```

### 自旋+枚举

**方法四，实际上和方法三一样**

和上述自旋不同就是将int类型的state变为了枚举类型，枚举类型状态有限，和我们的使用场景更加贴合，枚举类型的类型值名称可以使得我们的state状态更具有可读性

```java
import java.util.concurrent.CountDownLatch;

public class Main {
    enum State{Letter, Number}
    private static volatile State state = State.Letter;
    public static void main(String[] args) {
        new Thread(() -> {
            int begin = 'A';
            while (begin <= (int) 'Z') {
                while (state != State.Letter) {
                }
                System.out.print((char) begin++);
                state = State.Number;
            }
        }).start();
        new Thread(() -> {
            int begin = 1;
            while (begin <= 26) {
                while (state != State.Number) {
                }
                System.out.print(begin++);
                state = State.Letter;
            }
        }).start();
    }
}
```

### LockSupport

**方法五，利用LockSupport类**

问题：请问为什么将线程t1与线程t2定义成局部变量程序就会出错呢？(idea显示LockSupport.unpark(t2)处错误)？

```java
import java.util.concurrent.locks.LockSupport;

public class Main {
    static Thread t1 = null, t2 = null;

    public static void main(String[] args) {
        t1 = new Thread(() -> {
            int begin = 'A';
            while (begin <= (int) 'Z') {
                LockSupport.park();
                System.out.print((char) begin++);
                LockSupport.unpark(t2);
            }
        });
        t2 = new Thread(() -> {
            int begin = 1;
            while (begin <= 26) {
                System.out.print(begin++);
                LockSupport.unpark(t1);
                LockSupport.park();
            }
        });
        t1.start();
        t2.start();
    }
}
```



### 自旋与synchronized使用场景

自旋只是用户态的操作，synchronized涉及内核态的操作，比较耗性能

但是使用自旋的线程还是占用着cpu，而使用synchronized被阻塞的线程并没有占用cpu

**所以如果程序执行较快，并发数目较小，自旋几次就可以执行的话就使用自旋锁，而如果并发数较大，就适合使用synchronized锁。**假设有1W个线程执行上述程序，如果使用自旋锁的话，1个线程在执行的时候，999个线程都在等待并占用着cpu，这样的话线程就很低了，而使用synchronized的话，一个线程在执行的时候其他线程在等待队列中，并不占用cpu，这样性能就相对较好

## 强软弱虚引用

### 强引用

说明：我们一般使用的都是强引用，比如Thread t1 = new Thread();这就是一个强引用，只要t1一直引用着线程对象，那么该线程对象就不会被回收，即使发生了OOM

### 软引用

说明：

**1.在空间足够的情况下，即使垃圾回收也不会回收软引用所指向的对象**

**2.在空间不够的情况下，垃圾回收器会回收软引用所指向的对象**

```java
import java.io.IOException;
import java.lang.ref.SoftReference;

public class Main {

    public static void main(String[] args) throws IOException {
        SoftReference<byte[]> p = new SoftReference<>(new byte[1024*1024*10]);
        System.gc();
        System.out.println(p.get());
        byte[] oom = new byte[1024*1024*15];
        System.out.println(p.get());
    }
}
/**
[B@10f87f48
null
*/
```

**软引用的作用**

做**缓存**，有空间的时候缓存存在，提高查询的效率，空间不够的时候，缓存被清除

### 弱引用

**只要发生垃圾回收，就会回收掉软引用指向的对象(当然前提是没有强引用等指向它)**

```java
import java.io.IOException;
import java.lang.ref.WeakReference;

public class Main {

    public static void main(String[] args) throws IOException {
        WeakReference<byte[]> p = new WeakReference<>(new byte[1024*1024*10]);
        System.out.println(p.get());
        System.gc();
        System.out.println(p.get());
    }
}
/*
[B@10f87f48
null
**/
```

**弱引用的作用**

**防止内存泄漏**，在ThreadLocal类中就使用到了弱引用，一般刚接触到ThreadLocal的同学可能会将ThreadLocal理解成一个类似于map的容器，它的key是当前线程对象，其实这么理解是有错误的，**ThreadLocal对象其实是一个key！**，每一个**线程对象thread中都维护了一个**名称为threadLocals的**map**，当一个ThreadLocal对象调用了set方法传进一个value的时候，set方法内部会先找出当前对象的名为threadLocals的map，并将调用set方法的**ThreadLocal对象作为该map的key**，value为传进的要存储的值，但是其中这个key和value是包装成Entry类的，而这个Entry类的话继承了WeakReference即弱引用类！初始化的时候调用了super（key）方法，也就是说这个**key并不是直接指向ThreadLocal对象的，而是使用了软引用的指向！！！**，下面给出一个例子来具体理解

```java
public class Main {

    static ThreadLocalSon tl = new ThreadLocalSon();
    public static void main(String[] args){
        new Thread(()->{
            tl.set(new ThreadLocalSon());
        });
        new Thread(()->{
            tl.set(new ThreadLocalSon());
        });
        tl = null;
        System.gc();
    }
}
class ThreadLocalSon extends ThreadLocal{
    @Override
    public void finalize(){
        System.out.println("我没了");
    }
}
```

例子中为了让ThreadLocal被回收的时候我们能够看到一些东西，我们使用了其他类来继承了ThreadLocal类，覆写了finalize方法，程序中我们将tl设置为null的时候不要以为就没有指针指向前面实例化出来的ThreadLocalSon对象了，是有的，就在Thread的ThreadLocals这个map中，他是有key指向ThreadLocalSon对象的，但是这个指向是弱引用的，只要垃圾回收的时候就会被清除

### 虚引用

**注意：这一部分有问题，难以自圆其说，面试的时候可以直接说不是很了解：好像是用于管理堆外内存的**

使用虚引用指向的东西，get都get不到，就像存进去就没了一样！虚引用引用的对象被回收了之后会在ReferenceQueue队列中收到一条信息

虚引用用于管理堆外内存，比如说方法区（1.8之前是永久代实现，1.8之后是元空间实现）的内存，比如说directBuffer，如果说一个堆内对象指向了堆外的内存对象，那么这个堆外的内存怎么回收呢？我们将堆内内存对对外内存的引用更改为虚引用，一旦这个堆内引用对象被回收了，那么这个指向对象会被添加到队列中，我们再将堆外内存清除掉

```java
import java.io.IOException;
import java.lang.ref.PhantomReference;
import java.lang.ref.Reference;
import java.lang.ref.ReferenceQueue;
import java.util.ArrayList;
import java.util.List;

public class Main {
    static List<byte[]> list = new ArrayList<>();
    static ReferenceQueue<People> queue = new ReferenceQueue<>();

    public static void main(String[] args) throws IOException {

        PhantomReference<People> p = new PhantomReference<>(new People(), queue);
        new Thread(() -> {
            while (true) {
                Reference<? extends People> poll = queue.poll();
                if (poll != null) {
                    System.out.println("虚引用指向的对象被回收啦！" + poll.getClass().getName());
                }
            }
        }).start();
        new Thread(()->{
            while(true) {
                list.add(new byte[1024 * 1024]);
                System.out.println(p.get());
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}

class People {
    @Override
    public void finalize() {
        System.out.println("我没了");
    }
}
```



## cpu占用过高怎么排查

答：

1.首先使用top命令查看各个进程对cpu的占有情况，查看是哪个进程造成了cpu占用过高
2.使用ps H -eo pid,tid,%cpu | grep 进程号 查看是那个进程的哪个线程造成了cpu占用过高
3.使用jstack+进程号 命令查看线程情况，并根据上边查到的线程号来专门查看问题线程，查看是在哪一行代码中等信息
4.到具体位置去修改代码

参考https://www.bilibili.com/video/BV1yE411Z7AP

## 垃圾回收器

1.首先登上历史舞台的是**Serial（青年代）与Serial Old（老年代）**垃圾回收器，他们是jdk1.1，1.2的时候默认的垃圾回收器，单线程垃圾回收器，在需要进行垃圾回收的时候，这两个回收器会执行STW（stop the world），然后一个线程进行垃圾回收工作，这个时候的内存一般只有几M而已。

2.慢慢的我们的内存变大了，可以达到4G或者8G了，这个时候单线程垃圾回收器已经满足不了我们的需求了，因为如果空间很大，但是却只有一个线程在进行垃圾清除工作，这就会导致STW的时间很长，用户得到的感受就是程序不定时的长时间卡顿，在这种情况下**parallel scavenge（青年代）与parallel Old（老年代）**登场了，他们采用了多线程来进行垃圾回收（jdk1.8的默认垃圾回收器）

3.但是上述多线程垃圾回收器也是有很大问题的，那就是STW的时间还是过长，jvm厂商开始考虑是否能实现STW时间短一些的垃圾回收器，这时候**CMS**垃圾回收器登场了，CMS垃圾回收大体分为4各阶段，分别为初始标记，并发标记，重新标记，并发清除。在**初始标记**阶段，需要STW，CMS只会标记gc root对象，所以STW的时间相对较短，之后并发标记，所谓并发标记的意思就是说用户线程和垃圾回收线程一起都在运行，在**并发标记**阶段，CMS会将从GC ROOTS出发能够可达的所有节点都标记上，在**重新标记**阶段，CMS也会进行STW，这个阶段主要是处理那些在并发标记阶段引用改变的对象，因为改变的对象毕竟是少数的，所以这个阶段的STW也相对来说比较短。与CMS配合的青年代垃圾回收器是parallel NEW，parallel new垃圾回收器是由parallel scavenge改变而来的，能够适应cms的一些特性

4.CMS垃圾回收器是有一定毛病的，比如它使用的是标记清除算法，也就是说他会产生垃圾碎片的，当垃圾碎片过多的时候，它会变为parallel Old算法，这样的话就会有STW时间过长的问题了。所以没有任何一个jdk版本是以CMS作为默认垃圾回收器的。

5.随着内存的继续增大，32G或者100多G等，**G1**垃圾回收器诞生了（jdk9之后的默认垃圾回收器），G1垃圾回收器它将堆内存分为了很多的小块，青年代和老年代都是由许多的小块组成的，这样的话有一个好处就是垃圾回收的时候就收集某一些小块就可以了，并不用做整个堆的收集（G1垃圾回收器只知道这些是不太合格的，建议再去找一些资料来看）

## 观察者模式

注：总结于程序员小灰

观察者模式比较好玩也比较好理解，比如说现在有一款游戏，人物可以移动，周围存在陷阱，宝物等，如果玩家控制的人物走到了陷阱范围，那么人物落入陷阱，如果人物走到了宝物范围，那么人物可以发现并获取宝物，请问你该如何实现该功能？这里就可以使用观察者模式！！

人物是被观察者，陷阱与宝物都是观察者，人物类中维护了一个观察者队列，并且有注入，撤销和通知观察者们的功能，那么亲爱的读者们，你们能想到具体该怎么设计吗？或者说能够想到其中的一些点吗？

1.首先，人物类中实现了观察者队列，但是观察者们并不相同，我们的指向类型写什么好呢（就是List<?>方括号中？写什么好的意思）？为了有这个指向类型，我们要求观察者们通通实现统一的一个接口，比如Observer

2.除了玩家控制的人物，万一我们想要在游戏中加入一些npc呢？npc也要符合陷阱和宝物的相关规则，那么我们知道这个被观察者也要进行一层抽象，我们玩家控制的人物实现或者继承它（Subject）就好了

3.而我们Subject类或接口中的这个通知方法就是遍历观察者集合，都调用Observer中的某个更新方法就好了

相关程序如下

```java
//观察者
public interface Observer {
    public void update();
}

//被观察者
abstract public class Subject {

    private List<Observer> observerList = new ArrayList<Observer>();

    public void attachObserver(Observer observer) {
        observerList.add(observer);
    }

    public void detachObserver(Observer observer){
        observerList.remove(observer);
    }

    public void notifyObservers(){
        for (Observer observer: observerList){
            observer.update();
        }
    }
}
```

```java
//陷阱
public class Trap implements Observer {

    @Override
    public void update() {
        if(inRange()){
            System.out.println("陷阱 困住主角！");
        }
    }

    private boolean inRange(){
        //判断主角是否在自己的影响范围内，这里忽略细节，直接返回true
        return true;
    }
}
```

```java
public class Hero extends Subject{
    void move(){
        System.out.println("主角向前移动");
        notifyObservers();
    }
}
```

## Redis为什么速度那么快

**1.基于内存实现，完全内存计算**
**2.单线程操作，避免了线程上下文切换操作**
**3.多路I/O复用的线程模型，实现了一个线程监控多个IO流，及时响应请求**
redis对外部的依赖比较少，属于轻量级内存数据库

解析(后记：I/O多路复用是一个相对知识点较多的东西，这里只是简单的理解，请多查阅些资料)：
redis的线程模型多路I/O复用机制是一个比较重要并且常见的考察点。目前支持I/O多路复用的系统调用有select，poll，epoll等函数。I/O多路复用就是通过一种机制一个进程可以监视多个描述符，一旦某个描述符读就绪或者写就绪，其能够通知应用程序进行相应的读写操作。

多路I/O复用机制与多进程和多线程技术相比系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

## char和varchar的区别

1.char（len）和varchar（len）后边跟的都是字符的长度，而非字节的长度

2.**char（8）表示最多能够存储8个字符，**如果使用ascII码表来存储的话（一个字符对应1个字节），那么它占用的内存就是8个字节，**如果使用utf-8码表来存储的话（一个字符对应3个字节），那么它占用的内存就是8*3个字节**，如果使用gbk码表的话（一个字符对应2个字节），那么它占用的内存就是8\*2个字节，**如果没有存够8个字符，那么使用空白补充**

3..**varchar(8)也表示最多能够存储8个字符，但是不够8个字符的时候，也并不会使用空白补充**

4.**char在取值的时候会把存值后面的空格去除掉，varchar 如果后面有空格则会保留**（好玩哈底层保留的明面上却去掉了，底层去掉的明面上却保留了）

## dfs和bfs之间的区别

dfs是depth first search深度优先搜索算法，一般使用队列进行实现，算法运行规则类似于“一颗石子丢在湖面上溅出一圈圈涟漪”的感觉

bfs是breadth first search广度优先算法，一般使用递归实现，算法运行规则类似于“不撞南墙不回头”的感觉

## 多线程打印ABC

说明：该题和《线程通信》十分类似，只不过是面经上看到网易的该题，再练一下手而已

**问题：请建立ABC三个线程依次打印字母ABCABC………** 

分析：多线程是互联网大厂必考的知识点之一。这道题有诸多解法，本篇文章中将一一给出

------

**方法一**

**解法关键字**：volatile关键字+自旋等待

```java
public class Main {
    private static volatile int status = 0;
    public static void main(String[] args) {
        Thread ta = new Thread(new Runnable() {
            @Override
            public void run()
            {
                while (true) {
                    while (status != 0)sleep(100);
                    System.out.println("A");
                    status = 1;
                }
            }
        }, "A");
        Thread tb = new Thread(new Runnable() {
            @Override
            public void run()
            {
                while (true) {
                    while (status != 1)sleep(100);
                    System.out.println("B");
                    status = 2;
                }
            }
        }, "B");
        Thread tc = new Thread(new Runnable() {
            @Override
            public void run()
            {
                while (true) {
                    while (status != 2)sleep(100);
                    System.out.println("C");
                    status = 0;
                }
            }
        }, "C");
        tc.start();
        tb.start();
        ta.start();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
/**程序输出
 A
 B
 C
 A
 B
 C
 */
```

**方法一的改进版**

**解法关键字：**volatile + 自旋等待 + 枚举类

说明：上述的status只有较少的有限种状态，这特别适合使用枚举类，使用枚举类能够通过对名称的设计使得各个状态都比较清晰，可读性更强。给出这种写法，想必面试官一定会眼前一亮！

```java
public class Main {
    enum Status {acceptA, acceptB, acceptC}

    private static volatile Status status = Status.acceptA;
    public static void main(String[] args) {
        Thread ta = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    while (status != Status.acceptA) sleep(100);
                    System.out.println("A");
                    status = Status.acceptB;
                }
            }
        }, "A");
        Thread tb = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    while (status != Status.acceptB) sleep(100);
                    System.out.println("B");
                    status = Status.acceptC;
                }
            }
        }, "B");
        Thread tc = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    while (status != Status.acceptC) sleep(100);
                    System.out.println("C");
                    status = Status.acceptA;
                }
            }
        }, "C");
        tc.start();
        tb.start();
        ta.start();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
/**程序输出
 A
 B
 C
 A
 B
 C
 */
```

注意：

1. 枚举类就类似于class关键字，像一个类一样，记得enum是小写的哦
2. 内部的成员可以认为是静态的字符串类型，之间使用逗号进行来连接

------

**方法二**

**解法关键字：ReentrantLock+CountDownLatch**

说明：这里为了保证ABC的顺序而使用了CountDownLatch这个类，如果只是输出一遍ABC的话，就没有必要使用ReentrantLock直接使用CountDownLatch就可以了

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
public class Main {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        Condition acceptA = lock.newCondition();
        Condition acceptB = lock.newCondition();
        Condition acceptC = lock.newCondition();
        CountDownLatch second = new CountDownLatch(1);
        CountDownLatch third = new CountDownLatch(1);
        Thread ta = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    lock.lock();
                    System.out.println("A");
                    acceptB.signal();
                    second.countDown();
                    try {
                        acceptA.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.unlock();
                }
            }
        });
        Thread tb = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    second.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                while (true) {
                    lock.lock();
                    System.out.println("B");
                    acceptC.signal();
                    third.countDown();
                    try {
                        acceptB.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.unlock();
                }
            }
        });
        Thread tc = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        third.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.lock();
                    System.out.println("C");
                    acceptA.signal();
                    try {
                        acceptC.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    lock.unlock();
                }
            }
        });
        tc.start();
        ta.start();
        tb.start();
    }
}
```

------

**方法三**

**解法关键字：synchronized**

说明：synchronized只有一个监视器集，所以这道题并不适用，如果只是打印ABAB……那么就非常适合了。下面给出打印出ABAB的代码，并且同样地CountDownLatch只是为了确保打印A的程序先执行

```java
import java.util.concurrent.CountDownLatch;
public class Main {
    public static void main(String[] args) {

        CountDownLatch countDownLatch = new CountDownLatch(1);
        Thread ta = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    synchronized (Main.class) {
                        System.out.println("A");
                        try {
                            Main.class.notify();
                            countDownLatch.countDown();
                            Main.class.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        },"A");
        Thread tb = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    countDownLatch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                while (true) {
                    synchronized (Main.class) {
                        System.out.println("B");
                        try {
                            Main.class.notify();
                            Main.class.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        },"B");
        tb.start();
        ta.start();
    }
}
/**
 * 程序输出
 * A
 * B
 * A
 * B
 */
```

------

**方法四**

**解法关键字：CountDownLatch**

说明：前文提到过，如果只打印一遍ABC的话，那么可以使用CountDownLatch来完成，这里给出代码

```java
import java.util.concurrent.CountDownLatch;
public class Main {
    public static void main(String[] args) {

        CountDownLatch second = new CountDownLatch(1);
        CountDownLatch third = new CountDownLatch(1);
        Thread ta = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("A");
                second.countDown();
            }
        },"A");
        Thread tb = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    second.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("B");
                third.countDown();
            }
        },"B");
        Thread tc = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    third.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("C");
            }
        },"C");
        tc.start();
        tb.start();
        ta.start();
    }
}
/**
 * 程序输出
 * A
 * B
 * C
 * A
 * B
 * C
 */
```

------

**方法五**

**解法关键字：LockSupport**

说明：该方式应届生如果能够写得出来，那么一定会加分不少！

```java
import java.util.concurrent.locks.LockSupport;
public class Main {
    static Thread ta, tb, tc;
    public static void main(String[] args) {

        ta = new Thread(new Runnable() {
            @Override
            public void run() {
                while(true) {
                    System.out.println("A");
                    LockSupport.unpark(tb);
                    LockSupport.park();
                }
            }
        }, "A");
        tb = new Thread(new Runnable() {
            @Override
            public void run() {
                while(true) {
                    LockSupport.park();
                    System.out.println("B");
                    LockSupport.unpark(tc);
                }
            }
        }, "B");
        tc = new Thread(new Runnable() {
            @Override
            public void run() {
                while(true) {
                    LockSupport.park();
                    System.out.println("C");
                    LockSupport.unpark(ta);
                }
            }
        }, "C");
        tc.start();
        ta.start();
        tb.start();
    }
}
/**
 * 程序输出
 * A
 * B
 * C
 * A
 * B
 * C
 */
```

## Get和Post的区别

1.Get一般用来从服务器上查询获取资源；Post一般用来更新服务器上的资源；
2.Get方法将参数直接拼接在了URL后边，明文显示，可以通过浏览器地址栏直接访问；
3.Post请求用于提交表单，数据不是明文的，安全性更高；
4.Get请求有长度限制，Post请求没有

## 类加载过程

类加载的过程主要分为三个部分：

加载：是指将.class文件加载到内存中，将该字节码文件加载到jvm内存的方法区中，并在堆内存中建立对应的Class对象，用于封装类信息。
链接
初始化：初始化静态变量，静态代码块，构造代码块，构造函数的过程。

链接又可以细分为
验证：为了保证加载进来的字节流符合虚拟机规范，不会造成安全错误。
准备：为static修饰的变量分配内存并执行默认初始化。
解析：将符号引用替换为直接引用的过程，比如某个对象调用了hello方法，那么hello就是符号引用，直接引用就是他的内存地址，比如0xaabbccdd

## 聚簇索引和非聚簇索引

所谓的聚簇索引和非聚簇索引其实指的是一种**数据存储方式**，**聚簇索引中主键B+树存储的是主键和对应的行数据，而辅助键索引B+树存储的是辅助键和对应的主键**；**非聚簇索引中主键B+树和辅助键B+树存储的都是键和对应行数据的指针**（注意不是数据本身而是数据的地址），在聚簇索引数据存储方式中，如果我们使用了辅助键索引进行数据查找，数据会先在辅助键B+树中查找到对应的主键的值，再在主键B+树中查找到对应数据（这叫做回表），而非聚簇索引中就不需要回表操作，辅助键B+树和主键B+树之间没有什么联系，所以MYISAM引擎可以没有主键，但是INNODB就不行，必须得有主键

## INNODB没建立主键

那么问题来了，既然INNODB一定需要主键的话(《聚簇索引和非聚簇索引》章节有原因)，而用户建立表的时候没有指定主键怎么办？INNODB会将第一个非空且唯一的整形字段设置为主键，使用_rowId字段可查询，如果没有这样的字段，INNODB会自己建立一个隐藏字段作为主键（该主键查询不到）

## 乐观锁和悲观锁的区别

**后记：**没必要答这么多，就自己理解回答就可以了，比如乐观锁就直接回答为cas操作，AtomicInteger中的incrementAndGet()方法使用cas进行加1的操作(jdk14中使用的是Unsafe类中的cas操作)就是乐观锁的实现等；比如悲观锁synchronized关键字就是一种悲观锁，Vector类，Hashtable类，Collections.synchronizedMap()返回的Map类等方法上都有synchronized关键字修饰，都是悲观锁的实现

悲观锁和乐观锁是数据库用来保证数据并发安全防止更新丢失的两种方法(知乎回答上找的一句话，但似乎乐观与悲观锁的概念适应面更广)

悲观锁(Pessimistic Lock)

    顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。它指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度，因此，在整个数据处理过程中，将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制（也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）。

乐观锁(Optimistic Lock)

    顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库如果提供类似于write_condition机制的其实都是提供的乐观锁。

两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果经常产生冲突，上层应用会不断的进行retry，这样反倒是降低了性能，所以这种情况下用悲观锁就比较合适。

## 乐观锁的实现


乐观锁

    乐观锁（ Optimistic Locking ） 相对悲观锁而言，乐观锁假设认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则让返回用户错误的信息，让用户决定如何去做。
    
    上面提到的乐观锁的概念中其实已经阐述了他的具体实现细节：主要就是两个步骤：冲突检测和数据更新。其实现机制就是Compare and Set(CAS)

其实现机制

    CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。
    
    CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。（在 CAS 的一些特殊情况下将仅返回 CAS 是否成功，而不提取当前值。）CAS 有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。”这其实和乐观锁的冲突检查+数据更新的原理是一样的。
    
    这里再强调一下，乐观锁是一种思想。CAS是这种思想的一种实现方式

附加_CAS技术的问题_ABA问题

    CAS会导致“ABA问题”。
    
    CAS算法实现一个重要前提需要取出内存中某时刻的数据，而在下时刻比较并替换，那么在这个时间差类会导致数据的变化。
    
    比如说一个线程one从内存位置V中取出A，这时候另一个线程two也从内存中取出A，并且two进行了一些操作变成了B，然后two又将V位置的数据变成A，这时候线程one进行CAS操作发现内存中仍然是A，然后one操作成功。尽管线程one的CAS操作成功，但是不代表这个过程就是没有问题的。
    
    那么，乐观锁是如何解决ABA问题的呢？
    
    部分乐观锁的实现是通过version版本号的方式来解决ABA问题，乐观锁每次在执行数据的修改操作时，都会带上一个版本号，一旦版本号和数据的版本号一致就可以执行修改操作，否则就执行失败。因为每次操作的版本号都会随之增加，即使本次操作过程中数据被其他线程把数据从A改成B再改回A，那么在写操作进行的时候由于版本号与预期不一致也会执行失败。
## 线程池基本原理

答：通过ThreadExecutor类，或者通过Executors类，核心参数有7个，包括核心线程数，最大线程数，等待时间，等待时间的单位，等待队列，拒绝策略，线程工厂 ，核心线程数是指线程池中一直会存在的线程数，当来一个任务的时候就会派一个线程去处理，当核心线程都被使用完了之后会将任务放置在等待队列之中，如果等待队列满了，那么就会创建线程，直到线程等于最大线程数，当线程没有任务会执行就会等待一定的时间，这个时间由等待时间和等待时间的单位一起确定，时间一到，那么这些多余的线程就会被销毁，直到线程数又恢复到核心线程数

当等待队列满了，并且线程数已经达到最大线程数，那么就会执行拒绝策略，比如可能会直接将新任务丢弃并不作出处理，也可能会抛出异常等等

## HTTP和HTTPS 的区别

    HTTP（Hypertext Transfer Protocol）超文本传输协议是用来在Internet上传送超文本的传送协议，它可以使浏览器更加高效，使网络传输减少。但HTTP协议采用明文传输信息，存在信息窃听、信息篡改和信息劫持的风险。
    
    HTTPS(Secure Hypertext Transfer Protocol) 安全超文本传输协议是一个安全的通信通道，它基于HTTP开发，用于在客户计算机和服务器之间交换信息。HTTPS使用安全套接字层(SSL)进行信息交换，简单来说HTTPS是HTTP的安全版，是使用TLS/SSL加密的HTTP协议。
    
    HTTPS和HTTP的区别主要如下：
    
    https协议需要到ca申请证书，一般免费证书较少，因而需要一定费用。
    http是超文本传输协议，信息是明文传输，https则是具有安全性的ssl加密传输协议。
    http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。
    http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全.

## HTTPS 通信过程

    客户端发送请求到服务器端
    服务器端返回证书和公开密钥，公开密钥作为证书的一部分而存在
    客户端验证证书和公开密钥的有效性，如果有效，则生成共享密钥并使用公开密钥加密发送到服务器端
    服务器端使用私有密钥解密数据，并使用收到的共享密钥加密数据，发送到客户端
    客户端使用共享密钥解密数据
    SSL加密建立
## Collections.sort方法的实现

说明：这一部分资料比较少，建议b站上查找一下（有一个视频专门讲这个的）

Collections.sort方法底层调用了list.sort方法（这个是List接口的默认实现方法）而list.sort方法调用了Arrays.sort方法，这个方法的内部根据是否传入比较器分别调用了ComparableTimSort与TimSort这两个方法，这两个方法是差不多的，就是是不是根据传入的比较器进行对象的比较的区别

TimSort是Tim Peter于2002年发明的算法，集合了插入与归并排序算法

首先会根据用户传入集合的size决定是用何种算法，如果size小于32那么会使用插入算法（是用的是二分插入），具体为先查看集合头部的最长连续元素，如果为降序那么就进行反转，如果为升序，那么先不做任何处理，接下来则对后续元素进行数据的二分插入（因为前面的元素已经有序了，所以可以进行二分插入）；如果是size大于32那么会使用归并算法，在归并算法中，还会使用最长连续元素的思想，升序则反转，作者将每一个升序的分区称为一个run分区，根据最长连续元素的思想（降序反转）得出run分区，并且根据归并算法合并各个run分区！最终得出有序序列

**为什么使用归并而不是快排呢**

1.两者的平均时间复杂度都是O(NlogN)

2.归并是稳定排序而快排非稳定（博主瞎说的）

## 稳定算法

**稳定算法**

归并 + 插入 + 基数 + 冒泡	（画面记忆法）

**非稳定**

选择 + 快排 + 选择 + 希尔 + 堆排

## MVCC是什么

1.MVCC是指同一行数据在数据库中存在多个版本（事实上仅存一个版本，其他版本是通过undo log来实现的），它可以用来实现事务的隔离特性（似乎主要是实现读提交，可重复读隔离级别）

2.每一个版本的行数据都有一个属性为row trx_id，这个是记录哪一个事务将数据更改为现在这个样子的，在innodb中，每一个事务真正开启的时候，都会向innodb事务管理系统申请一个唯一的递增的id，当一个事务更改了某行数据的时候，这个行数据的row trx_id属性就是这个事务的id值

3.当一个事务开启的时候，它会建立一个目前正在活跃（表示开启了但是还没有提交）的事务数组（并维护这个数组），并且记录下这些事务的最大（称为高水位）与最小值（称为低水位），当这个事务查询某一个数据的时候，它会判断这行数据的row trx_id属性，如果大于高水位证明这行数据是当前事务开启之后才开启并且更改此行数据的，不予显示并且调用undo log日志查找以前的版本，如果查找到的是低水位之前的数据说明是以前已经更改过的，可以进行显示，如果落在高低水位之间，并且在数组中表示修改了数据但是还没提交，不予显示，如果不在数组中表示修改了数据并且提交了，予以显示（读提交隔离级别下）

4.**上述是读数据的情况，那么修改数据的情况呢？**，如果是修改数据，则必须遵循**当前读**规则，也就是说只会查询现在的版本而不会查询历史版本判断row trx_id，否则就会将其他事务的对数据的更改覆盖掉

## redo log和binlog

1.**redo log是为了crash safe**，**binlog是为了备份修复的（似乎也可以用于主从数据库之间的数据一致性）**

2.redo log是引擎层面的实现，是INNODB数据库的特有的日志，而binlog是Server层面的实现，是所有引擎都具有的日志，redo log是物理日志，记录着某某页的改变，binlog是逻辑日志，记录着原始的sql语句的逻辑。binlog是追加日志，日志不断的写，达到一个文件大小之后再另外开启一个文件继续写，而redo log的话是循环写，就是说日志会覆盖的，当然覆盖之前，redo log涉及的脏页已经刷新完毕

3.**为什么说binlog是用于备份修复的呢？**在一个数据库内，根据数据的重要性，有一天一备份，一周一备份等，比如我们采用的是一天一备份，那么比如我们现在发现12：00的时候我们误删了一个表格，那么我们可以找到昨天或者说今天凌晨00：00的备份数据得到临时数据库，再将binlog重00：00进行重放，重放到12：00那么我们现在就有未误删除表格之前的数据了，我们再按需恢复即可

4.**为什么说redo log是用于crash safe呢？**我们要知道更新语句的执行过程，比如现在要执行update table set k=k+1 where id = 10那么**1.**执行器会调用引擎接口，引擎走索引树搜索到id等于10的数据所在的数据页。如果该数据页在内存中则直接将这一行数据返回，如果不存在则将其从磁盘中读入。**2**引擎将这一行数据返回给执行器，执行器将k的值进行+1并且调用引擎的接口写入数据，**3**引擎将结果写入内存，并且记录redolog日志，并将其置为prepare状态，并告知执行器随时可以准备提交。**4**执行器收到以后记录binlog，将binlog写入磁盘之后告知引擎。**5**引擎将redolog设置为commit状态。这就是两阶段提交过程，两阶段提交过程是为了保证一旦数据库在数据库发生崩溃，那么内存中的数据就丢失了，但是用户调用那边却已经commit，这该怎么办呢？我们使用redo log进行内存数据的恢复。

## sql查询语句的执行过程

1.mysql内部分为Server层和引擎层，Server层分为连接器，缓存器，分析器，优化器，执行器，引擎层里边有我们常说到到的MYISAM和INNODB引擎等

2.用户通过连接器与mysql相连，连接器负责验证你的用户名与密码，如果正确会根据权限表查的该用户所拥有的权限

3.mysql的早期版本中存在缓存器，sql的查询语句会先看缓存器中是否存在该数据，如果存在那么就直接返回，如果不存在那么再继续执行。由于如果执行了更新语句或者插入，删除语句，那么缓存器中的数据就要被清除，对于数据库来说很多情况下更新等操作很频繁，这回造成缓存中的内容频繁失效，以致于维护它所消耗的性能要多于它所提升的性能，所以后期版本（8.0版本）中缓存器直接就被删除了

4.之后进入分析器，分析器分为词法分析器和语法分析器，词法分析器用于区分各个词语，比如从一整个的sql中区分出来select语句，哪个是表名等，如果不存在该表名，那么在该阶段就会报错，之后语法分析器分析是否合乎sql语法，各个语法报错就是这个阶段报出来的

5.之后进行优化器，优化器会判断我们查询数据的时候应该使用哪一种索引，这有些时候会造成一定程度的误判

6.执行器用于和各个引擎打交道，执行器会调用引擎的接口函数来执行sql语句

## 数据库隔离级别

答：不提交读，在该级别下，一个事务中对数据进行了修改但是并没有进行提交，另一个事务也会有数据的变化，这样就会出现**脏读**的现象，就是说在一个事务中读到了另一个事务的未提交的数据（万一头一个事务回滚，那么另外一个事务读到的就是错误的数据）

​	提交读，该级别主要是解决脏读的现象，就是说虽然在一个事务中对数据进行了更改，但是只要在这事务并没有被提交，另一个事务中就不会有数据的变化，但是这种隔离模式下会有不可重复的问题，就是在一个事务中前后两次读取的数据并不一致（因为在这期间头一个事务进行了提交操作）

​	可重复读，在该级别下，一个事务中查询同一份数据不会有数据变化的问题，但是可能会有数据总数的问题，也就是幻读现象

​	串行读，没有任何问题，但是性能最低



# 集合专题

## ArrayList、LinkedList、Vector区别

ArrayList底层是数组实现，查找比较快，增删相对较慢(**或者说大多数情况下比较慢，只有在一些特殊的情况下比较快，尽量引导面试官问该问题**)，线程不安全；LinkedList底层是链表实现，增删比较快，查找相对较慢，线程不安全；Vector与ArrayList差不多，只是将所有方法添加了Synchronized关键字，以保证线程安全

## ArrayList的add与delete方法的实现

add方法中，添加元素之前会先使用ensureCapacityInternal(size+1)来确保数组可以容纳该元素，add方法中调用了System.arraycopy(elementData,index,elementData,index+1,size-index)来添加元素，也就说它会复制elementData数组中从index开始的元素到index+1至末尾中，从而空出了index位置可供新插入的元素使用

delete方法中也是使用了System.arraycopy(elementData,index+1,elementData,index,size-index-1)方法，它将elementData数组中index+1至末尾的元素放置到elementData数组中index至末尾，这样的话就将原来index位置处的元素覆盖掉了，看起来就像删除一样

## ArrayList的增删方法一定慢吗

不一定的，经过add和delete方法的解析我们知道，ArrayList的增删之所以慢的原因是涉及增删位置后面的元素的复制，但是如果我们增删的是最后一个元素的话就不会存在增删速度慢的问题了，所以ArrayList还是很适合来做栈的数据结构的

## ArrayList怎么来进行扩容的呢

新建立一个数组，容量大小为原来的1.5倍（这可能的原因之一为1/2可用位运算来做，速度快），再将原数组中的元素赋值到新数组中即可

## LinkedList的查找复杂度

答：O(n/2)，底层是一个双向链表，在调用get方法的时候，会根据调用者传入的index离头节点近还是尾节点近来进行遍历，所以查找的时间复杂度是O(n/2)

具体：get方法内部首先会根据checkElementIndex(index)方法来检查index是否合法，之后会调用node(index)方法，该方法会返回Node对象，所以再返回Node对象的item属性就可以了

```java
public E get(int index){
    checkElementIndex(index);
    return node(index).item;
}
```

而node方法的内部就会根据index距离头节点还是尾节点近（根据链表的size来进行判断）来进行从头节点或尾节点的遍历

## HashMap的底层数据结构

jdk1.7是数组+链表，jdk1.8是数组+链表+红黑树

## HashMap的存取原理

hashmap底层首先有一个数组，jdk1.7叫做Entry，jdk1.8叫做Node，这两个内部类里边都封装了key和value属性（实际上还封装了key的hash值，next属性），当put进来一个元素的时候，我们会先计算key的hash值&（len-1）得到index，并把这个key和value对应的Node放入Node数组中，如果这个数组已经有元素了，那么就要考虑覆盖或者形成链表，或者形成红黑树，jdk1.8之后这个链表元素个数大于等于8的时候就会形成红黑树（实际上capacity还需要大于64），小于等于6的时候就会蜕化成链表

## HashMap是否线程安全

否，HashMap在多线程情况下并不安全，jdk1.7之前插入元素使用的是头插法，可能在扩容的时候链表会出现环的情况；jdk1.8之后插入元素使用的是尾插法，并不会出现环，但是还是线程不安全的，一个线程put元素之后，再get可能就不会取到期望的元素了（可能已经被其他线程更改）

**为什么使用头插法**

作者认为后插入的被查询的可能性较大，为了提高效率使用头插法

**头插法为什么可能会使链表成环**

因为使用头插法链表在扩容前后的次序可能改变，比如链表节点为A->B->C，扩容后B->A（C变为其他桶位），那么在多线程更改链表指向的情况下可能就会出现A->B->A这种循环的情况，而使用尾插法，链表原来为A->B那么扩容后仍然为A->B

## HashMap在jdk1.7和1.8的不同

1.底层实现不同。jdk1.7使数组+链表，jdk8是数组+链表+红黑树

2.插入机制不同。jdk1.7使用的是头插法，jdk1.8使用的是尾插法

3.扩容位置计算方式不同。jdk1.7新建数组之后，原来的Entry重新hash以确定新的位置，jdk1.8新建数组之后，他会根据原来的key的hash的最高位来判断比如原来的key的值为10000，原来的容量为16减一为是1111，扩容之后的容量为32减一为11111，那么根据key的hash最高位为1可以判断新的index为原来的index加上原来的容量16，如果原来的key的hash值得最高位为0，那么index不变。总的来说就是在index计算上做了一个小的优化

## HashMap什么时候进行扩容

当size>capacity*load_factor的时候进行扩容，容量变为原来的2倍，之所以是2倍是为了元素能够进行均匀分布，因为最后在进行index确定的时候，我们需要使用hash&（len-1）

## 举例说明ArrayList线程不安全

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class Main {
    private static List<Integer> list = new ArrayList<>();

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    list.add(1);
                }
                countDownLatch.countDown();
            }
        }).start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    list.add(1);
                }
                countDownLatch.countDown();
            }
        }).start();
        countDownLatch.await();
        System.out.println(list.size());
    }
}
```

说明：

1.线程的结果可能并不是20万，可能会少于20万

# 算法专题

## dfs

```java
棋盘 n*n
i，j 点有一个棋子
棋子只能从上下左右四个方向走，每次走一格
当走到棋盘的最边缘时，继续向外走，则认为棋子掉落。
问：k 步后，这个棋子还在棋盘的概率有多大？
```

```java
class Solution {
    private double drop = 0;

    public double func(int n, int i, int j, int k) {
        if (i < 0 || j < 0 || i >= n || j >= n || k < 0)
            throw new IllegalArgumentException("参数异常！");
        dfs(n, i, j, k, 0, 1);
        return (1 - drop);
    }

    private void dfs(int n, int i, int j, int k, int num, double pre) {
        if ((int) inRomain(n, i, j) == 0) {
            drop += pre;
            return;
        }
        if (num == k) {
            return;
        }
        dfs(n, i - 1, j, k, num + 1, 0.25 * pre);
        dfs(n, i + 1, j, k, num + 1, 0.25 * pre);
        dfs(n, i, j - 1, k, num + 1, 0.25 * pre);
        dfs(n, i, j + 1, k, num + 1, 0.25 * pre);
    }

    private double inRomain(int n, int i, int j) {
        if (n < 1)
            throw new IllegalArgumentException("参数异常！");
        if (i < 0 || i >= n || j < 0 || j >= n)
            return 0;
        return 1;
    }
}
```

## 树

### 重建二叉树

只要有中序遍历就可以重建二叉树，**前序遍历+中序遍历，后序遍历+中序遍历都可以重建二叉树，**但是前序遍历+后序遍历不可以重建二叉树

### 二叉树的最近公共祖先

方法一

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root == null || root == p || root == q) return root;
        TreeNode left = lowestCommonAncestor(root.left, p, q);
        TreeNode right = lowestCommonAncestor(root.right, p, q);
        if(left == null) return right;
        if(right == null) return left;
        return root;
    }
}
```

方法二

思路：利用中序遍历，中序遍历的返回值为布尔类型，意义为以root为根节点的子树包不包含p或者q节点

```java
class Solution {
    private TreeNode res;

    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null || p == null || q == null)
            return null;
        inOrder(root, p, q);
        return res;
    }

    //意义为以root为根节点的子树包不包含p或者q节点
    private boolean inOrder(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null)
            return false;
        boolean temp = root == p || root == q;
        boolean left = inOrder(root.left, p, q);
        boolean right = inOrder(root.right, p, q);
        if(temp && left && res==null)//如果当前节点包含p或者q节点并且其左子树也包含p或者q节点并且res没有被赋值过，那么当前节点root为目标节点，其他if条件类同
            res = root;
        if(temp && right && res==null)
            res = root;
        if(left && right && res==null)
            res = root;
        return temp || left || right;
    }
}
```

## 二分查找

### 非递归形式

```java
class Solution {
    public static int binarySearch(int[] arr, int key) {
        if (arr == null || arr.length == 0)
            return -1;
        int lo = 0, hi = arr.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo + 1) / 2;
            if (key == arr[mid])
                return mid;
            if (key < arr[mid])
                hi = mid - 1;
            else
                lo = mid + 1;
        }
        return -1;
    }
}
```

### 递归形式

```java
class Solution {
    public static int binarySearch(int[] arr, int key) {
        if (arr == null || arr.length == 0)
            return -1;
        return binarySearch(arr, 0, arr.length-1, key);
    }

    private static int binarySearch(int[] arr, int lo, int hi, int key) {
        if (lo > hi)
            return -1;
        int mid = lo + (hi - lo + 1) / 2;
        if (arr[mid] == key)
            return mid;
        if (key < arr[mid])
            return binarySearch(arr, lo, mid - 1, key);
        else
            return binarySearch(arr, mid + 1, hi, key);
    }
}
```

### 目标数在有序数组中的位置

题目(网易)：有序数组[3,5,5,5,7,8,9] target = 5 目标数在数组中出现的最左索引和最右索引

说明：仅仅演示了最左索引，最右索引只需要改动while处的代码就可以了，或者把while处的条件封装为一个方法，改动方法

```java
class Solution {
    public int binarySearch(int[] arr, int target) {
        if (arr == null || arr.length == 0)
            throw new IllegalArgumentException("参数异常!");
        return binarySearch(arr, 0, arr.length - 1, target);
    }

    private int binarySearch(int[] arr, int lo, int hi, int target) {
        if (lo > hi)
            return -1;
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) {
            while (mid - 1 >= 0 && arr[mid - 1] == target)
                mid--;
            return mid;
        }
        if (target < arr[mid])
            return binarySearch(arr, lo, mid - 1, target);
        else
            return binarySearch(arr, mid + 1, hi, target);
    }
}
```

# 水果

## 概率

**猿辅导2020苹果题**

**题目：**有一个苹果，两个人轮流抛一枚硬币，正面的话可以吃这颗苹果，请问第一个抛得的人吃到苹果的概率有多大？

选项有1/2，3/4等共四个选项，请问概率是多大？

**参考答案：**不是1/2也不是3/4。这里不给出博主的答案与计算过程，因为大家自己计算出结果的过程是无比happy的，如果计算出来了，那就是那个答案了！

# 高频面试题

## 项目中遇到了哪些困难

说明：技术面高频面试题，只要面试你的项目，10中有7都会问道这道题

## 你对菜鸟/滴滴有多少了解

说明：hr面高频面试题，大公司小公司都可能会问该题，菜鸟和滴滴问这道题的时候问的我措手不及

# HR面经

## 加班

1.如何看待加班？能忍受什么程度的加班？

**禁忌**：最好不要提钱，例如：我觉得加班是理所应当可以接受的，毕竟钱也是赚的多，，，，

**面试官心态**：是不是不给你加班费你就不加班了，，，，

**推荐**：我觉得加班对于年轻人来说是个好事，因为加班的时候也能学到很多的技术，这样的话我就能更快地成为技术大牛了，，，，，

## 地域

1.我看你是黑龙江的，为什么毕业选择来杭州呢？

**禁忌**：杭州互联网环境好，，，，，，

**面试官心态**：你这是想好找下家呀，，，，

**推荐**：可以从气候，美食等方面来作答，，，，

## 优缺点

1.准备三个优点，三个缺点

**禁忌**：优点当作缺点来说







