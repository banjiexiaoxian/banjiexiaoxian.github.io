---
layout: post
title: "Java多线程之死锁情形（持续更新)"
date: 2017-03-18 20:41:31 +0800
comments: true
categories: JDK
---
本文中的所有情形收集来自博客、论坛、github及自己在项目中遇到的情形。
持续更新中
<!—more—>
###1、多生产者多消费者问题中由于所有角色共享一个同步锁而发生死锁  [***来源链接***](http://www.2cto.com/kf/201410/344213.html)

```
package CreatorAndConsumer;
 
import java.util.ArrayList;
import java.util.List;
 

//盘子，表示共享的资源，在Plate类中维护一个eggs列表
public class Plate {
    private List<object> eggs = new ArrayList<object>();
     
    /*
     * 获取蛋
     */
    public Object getEgg()
    {
        System.out.println("消费者取蛋");
        Object egg = eggs.get(0);
        eggs.remove(0);
        return egg;
    }
     
     
    /**
     * 加入蛋
     */
    public void addEgg(Object egg)
    {
        System.out.println("生产者生蛋");
        eggs.add(egg);
    }
     
    /**
     * 获取蛋个数
     */
    public int getEggNum()
    {
        return eggs.size();
    }
}
```

接下来是消费者
```
package CreatorAndConsumer;
 
public class Consumer implements Runnable {
    /**
     * 线程资源
     */
    private Plate plate;
 
    public Consumer(Plate plate) {
        this.plate = plate;
    }
 
    @Override
    public void run() {
        synchronized (plate) {//消费者获取锁
            
            while (plate.getEggNum() < 1) {
                try {
                    // 如果蛋不够，消费者需要等待，进入wait，便会释放持有的锁，但不能保证是生产者获得了这个锁。进入wait之后需要等待其它线程的notify或者notifyAll唤醒
                    plate.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 被唤醒后，再次持有锁，放心地取蛋
            plate.getEgg();
            //取完蛋后，唤醒其它等待的线程
            plate.notify();
        }
        //释放锁
    }
}
```
之后是生产者
```
package CreatorAndConsumer;
 
/**
 * 生产者
 */
public class Creator implements Runnable {
    /**
     * 线程资源
     */
    private Plate plate;
 
    public Creator(Plate plate) {
        this.plate = plate;
    }
 
    @Override
    public void run() {
        synchronized (plate) {//生产者获取锁
            // 如果此时盘中的蛋已放不下时，进入wait，暂时释放锁。但不能保证该锁被消费者拿到。等待消费者消费egg后被唤醒
            while (plate.getEggNum() >= 5) {
                try {
                    plate.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            // 唤醒后，生产蛋
            Object egg = new Object();
            plate.addEgg(egg);
            //生产了蛋之后唤醒其他线程
            plate.notify();
        }
        //释放锁
    }
}
```
那么什么情况下，会出现死锁呢？
举个栗子，当有多个生产者时，不能保证生产者释放的锁一定被消费者取得。若生产者释放的锁被另外一个生产者所取得，那么极可能出现两个生产者都进入wait之后，消费者才获取到锁，接着启动notify唤醒其中一个生产者。如此循环，另一个生产者可能总是处于wait状态。
多个消费者时也同理。
解决这个问题的思路是，生产者释放给消费者的锁不能被另一个生产者拿到。
解决方案就是，增加一个给生产者专用的锁和给消费者专用的锁，保证每一次生产者需要释放锁给消费者时，其他的生产者都无法进入竞争环节。
以消费者举例，修改后的代码如下：

```
public class Consumer implements Runnable {
    /**
     * 线程资源
     */
    private Plate plate;
 
    /**
     * 消费者锁
     */
    private static Object consumerLocker = new Object();
 
    public Consumer(Plate plate) {
        this.plate = plate;
    }
 
    @Override
    public void run() {
        // 必须先获得消费者锁才能消费，限制只有一个消费者线程能进入竞争共享资源锁Plate的环节
        synchronized (consumerLocker) {
            synchronized (plate) {
                while (plate.getEggNum() < 1) {
                    try {
                        plate.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                plate.getEgg();
                plate.notify();
            }
        }
    }
}

```



###2、并发下的HashMap Rehash导致死锁   [***来源链接1***](https://github.com/oldratlee/fucking-java-concurrency)  [***来源链接2***](http://coolshell.cn/articles/9606.html)


###3、两个线程对两个共享资源加锁顺序不同导致死锁  [***来源链接***](https://github.com/oldratlee/fucking-java-concurrency) 

```
/**
 * @author Jerry Lee (oldratlee at gmail dot com)
 */
public class SymmetricLockDeadlockDemo {
    static final Object lock1 = new Object();
    static final Object lock2 = new Object();

    public static void main(String[] args) throws Exception {
        Thread thread1 = new Thread(new ConcurrencyCheckTask1());
        thread1.start();
        Thread thread2 = new Thread(new ConcurrencyCheckTask2());
        thread2.start();
    }

    private static class ConcurrencyCheckTask1 implements Runnable {
        @Override
        public void run() {
            System.out.println("ConcurrencyCheckTask1 started!");
            while (true) {
                synchronized (lock1) {
                    synchronized (lock2) {
                        System.out.println("Hello1");
                    }
                }
            }
        }
    }

    private static class ConcurrencyCheckTask2 implements Runnable {
        @Override
        public void run() {
            System.out.println("ConcurrencyCheckTask2 started!");
            while (true) {
                synchronized (lock2) {
                    synchronized (lock1) {
                        System.out.println("Hello2");
                    }
                }
            }
        }
    }
}
```
这个例子理解起来比较简单，多线程环境下，Thread1获得B锁，等待A锁，Thread2获得A锁，等待B锁，造成了死锁。

