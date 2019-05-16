# Volatile

> 在 java 垃圾回收整理一文中，描述了jvm运行时刻内存的分配。其中有一个内存区域是jvm虚拟机栈，每一个线程运行时都有一个线程栈，线程栈保存了线程运行时候变量值信息。** 当线程访问某一个对象时候值的时候，首先通过对象的引用找到对应在堆内存的变量的值，然后把堆内存变量的具体值load到线程本地内存中，建立一个变量副本， ** 之后线程就不再和对象在堆内存变量值有任何关系，而是直接修改副本变量的值，在修改完之后的某一个时刻（线程退出之前），自动把线程变量副本的值回写到对象在堆中变量。这样在堆中的对象的值就产生变化了。

![](https://images0.cnblogs.com/blog/531072/201310/11172340-1b4ffc1abd6047798761edf5c5070ec1.jpg)

## 定义
- 一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：
  - 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
  - 禁止进行指令重排序。 

## Demo
```
public class VolatileTest extends Thread {

    boolean flag = false;
    int i = 0;

    public void run() {
        while (!flag) {
            i++;
        }
    }

    public static void main(String[] args) throws Exception {
        VolatileTest vt = new VolatileTest();
        vt.start();
        Thread.sleep(2000);
        vt.flag = true;
        System.out.println("stope" + vt.i);
    }
}
```

上面的叙述看似并没有什么问题，“似乎”完全正确。那就让我们把程序运行起来看看效果吧，执行mian方法。2秒钟以后控制台打印stope-202753974。

可是奇怪的事情发生了 程序并没有退出。vt线程仍然在运行，也就是说我们在主线程设置的 vt.flag = true;没有起作用。

原因： 
- 首先 vt线程在运行的时候会把 变量 flag 与 i (代码3,4行)从“主内存”  拷贝到 线程栈内存（上图的线程工作内存）

- 然后 vt线程开始执行while循环 。while (!flag)进行判断的flag 是在线程工作内存当中获取，而不是从 “主内存”中获取。```i++;``` 将线程内存中的```i++;``` 加完以后将结果写回至 "主内存"，如此重复。

- 然后再说说主线程的执行过程。
- 主线程将vt.flag的值同样 从主内存中拷贝到自己的线程工作内存 然后修改flag=true. 然后再将新值回到主内存。
- 这就解释了为什么在主线程（main）中设置了vt.flag = true; 而vt线程在进行判断flag的时候拿到的仍然是false。那就是因为vt线程每次判断flag标记的时候是从它自己的“工作内存中”取值，而并非从主内存中取值！

## Demo2
- 在flag前面加上volatile关键字，强制线程每次读取该值的时候都去“主内存”中取值。程序正常退出

```
public class VolatileTest extends Thread {
    
    volatile boolean flag = false;
    int i = 0;
    
    public void run() {
        while (!flag) {
            i++;
        }
    }
    
    public static void main(String[] args) throws Exception {
        VolatileTest vt = new VolatileTest();
        vt.start();
        Thread.sleep(2000);
        vt.flag = true;
        System.out.println("stope" + vt.i);
    }
}
```


## volatile保证原子性吗？
```
public class Test {
    public volatile int inc = 0;
     
    public void increase() {
        inc++;
    }
     
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
         
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```

- 大家想一下这段程序的输出结果是多少？也许有些朋友认为是10000。但是事实上运行它会发现每次运行结果都不一致，都是一个小于10000的数字。

- 这里面就有一个误区了，volatile关键字能保证可见性没有错，但是上面的程序错在没能保证原子性。可见性只能保证每次读取的是最新的值，但是 ** volatile没办法保证对变量的操作的原子性。 **

- 在前面已经提到过，自增操作是不具备原子性的，它包括读取变量的原始值、进行加1操作、写入工作内存。那么就是说自增操作的三个子操作可能会分割开执行，就有可能导致下面这种情况出现：
> 假如某个时刻变量inc的值为10，
线程1对变量进行自增操作，线程1先读取了变量inc的原始值，然后线程1被阻塞了；
>
然后线程2对变量进行自增操作，线程2也去读取变量inc的原始值，由于线程1只是对变量inc进行读取操作，而没有对变量进行修改操作，所以不会导致线程2的工作内存中缓存变量inc的缓存行无效，也不会导致主存中的值刷新，所以线程2会直接去主存读取inc的值，发现inc的值时10，然后进行加1操作，并把11写入工作内存，最后写入主存。
>
然后线程1接着进行加1操作，由于已经读取了inc的值，注意此时在线程1的工作内存中inc的值仍然为10，所以线程1对inc进行加1操作后inc的值为11，然后将11写入工作内存，最后写入主存。
>
那么两个线程分别进行了一次自增操作后，inc只增加了1。
根源就在这里，自增操作不是原子性操作，而且volatile也无法保证对变量的任何操作都是原子性的。


### volatile保证有序性
- volatile关键字禁止指令重排序有两层意思：
 - 当程序执行到volatile变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；
 - 在进行指令优化时，不能将在对volatile变量的读操作或者写操作的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

## volatile的应用场景

- synchronized关键字是防止多个线程同时执行一段代码，那么就会很影响程序执行效率，而volatile关键字在某些情况下性能要优于synchronized，但是要注意volatile关键字是无法替代synchronized关键字的，因为volatile关键字无法保证操作的原子性。通常来说，使用volatile必须具备以下2个条件：
 - 对变量的写操作不依赖于当前值
 - 该变量没有包含在具有其他变量的不变式中

- 下面列举几个Java中使用volatile的几个场景。
 - 状态标记量

 ```
 volatile boolean flag = false;
 //线程1
while(!flag){
    doSomething();
}
  //线程2
public void setFlag() {
    flag = true;
}
 ```
 根据状态标记，终止线程。
 
 - 单例模式中的double check
 
 ```
 class Singleton{
    private volatile static Singleton instance = null;
 
    private Singleton() {
 
    }
 
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
 ```
 
** 为什么要使用volatile 修饰instance？ **
> 主要在于instance = new Singleton()这句，这并非是一个原子操作，事实上在 JVM 中这句话大概做了下面 3 件事情:

> 1. 给 instance 分配内存
> 2. 调用 Singleton 的构造函数来初始化成员变量
> 3. 将instance对象指向分配的内存空间（执行完这步 instance 就为非 null 了）。

> 但是在 JVM 的即时编译器中存在指令重排序的优化。也就是说上面的第二步和第三步的顺序是不能保证的，最终的执行顺序可能是 1-2-3 也可能是 1-3-2。如果是后者，则在 3 执行完毕、2 未执行之前，被线程二抢占了，这时 instance 已经是非 null 了（但却没有初始化），所以线程二会直接返回 instance，然后使用，然后顺理成章地报错。




