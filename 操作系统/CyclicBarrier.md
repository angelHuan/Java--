# CyclicBarrier

## 简介：
![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/cyclicbarrier.png)

- 与CountDownLatch类似，都是通过计数器实现的，当某个线程调用await之后，计数器减1，当计数器大于0时将等待的线程包装成AQS的Node放入等待队列中，当计数器为0时将等待队列中的Node拿出来执行。

- 与CountDownLatch的区别：
	- CountDownLatch是一个线程等其他线程，CyclicBarrier是多个线程相互等待
	- CyclicBarrier的计数器能重复使用，调用多次

## 使用场景：
> 字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。

## 源码分析：
- CyclicBarrier类位于java.util.concurrent包下，CyclicBarrier提供2个构造器：

```
public CyclicBarrier(int parties, Runnable barrierAction) {
}

public CyclicBarrier(int parties) {
}
```
  - 第一个版本比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；
  - 第二个版本是让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。

## DEMO
- 假若有若干个线程都要进行写数据操作，并且只有所有线程都完成写数据操作之后，这些线程才能继续做后面的事情，此时就可以利用CyclicBarrier了：

```
public class Test {
    public static void main(String[] args) {
        int N = 4;
        CyclicBarrier barrier  = new CyclicBarrier(N);
        for(int i=0;i<N;i++)
            new Writer(barrier).start();
    }
    static class Writer extends Thread{
        private CyclicBarrier cyclicBarrier;
        public Writer(CyclicBarrier cyclicBarrier) {
            this.cyclicBarrier = cyclicBarrier;
        }
 
        @Override
        public void run() {
            System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
            try {
                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }catch(BrokenBarrierException e){
                e.printStackTrace();
            }
            System.out.println("所有线程写入完毕，继续处理其他任务...");
        }
    }
}
```
执行结果：
```
线程Thread-0正在写入数据...
线程Thread-3正在写入数据...
线程Thread-2正在写入数据...
线程Thread-1正在写入数据...
线程Thread-2写入数据完毕，等待其他线程写入完毕
线程Thread-0写入数据完毕，等待其他线程写入完毕
线程Thread-3写入数据完毕，等待其他线程写入完毕
线程Thread-1写入数据完毕，等待其他线程写入完毕
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
所有线程写入完毕，继续处理其他任务...
```

从上面输出结果可以看出，每个写入线程执行完写数据操作之后，就在等待其他线程写入操作完毕。
当所有线程线程写入操作完毕之后，所有线程就继续进行后续的操作了。




