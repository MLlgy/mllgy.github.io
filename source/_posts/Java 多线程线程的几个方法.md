---
title: 线程的 wait、sleep、join、yeild 方法
tags: [Thread]
date: 2019-09-05 15:53:17
---

### 0x0001 wait()

首先，需要明确的是 wait() 方法为 Object 类中的方法，所以 Object 对象才可以执行 wait() 方法，但是该方法和线程的各个状态息息相关。

当一个线程执行到 wait() 方法时，它就进入到一个 **和该对象相关的等待池** 中，同时 **该线程失去(释放)对象所持有的锁**，使得其他线程可以访问。用户可以使用 notify、notifyAll 或者指定睡眠时间来唤醒当前等待池中的线程。

需要注意的是：wait()、notify()、notifyAll() 必须放在 synchronized block 中，否则会抛出异常。

<!-- more -->

```
public class Main {
    
    private static final Object mLockObject = new Object();
    public static void main(String[] args) {
        waitAndNotifyAll();
    }

    private static void waitAndNotifyAll(){
        System.out.println("主线程执行");
        Thread thread = new WaitThread();
        thread.start();
        long startTime = System.currentTimeMillis();
        // 锁对象为 mLockObject
        synchronized (mLockObject){
            try {
                System.out.println("主线程等待");
                // 调用 mLockObject 对象的 wait 方法，使当前线程失去该对象的锁。
                mLockObject.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        long timsMs = System.currentTimeMillis() - startTime;
        System.out.println("主线程继续 -> 等待耗时："  + timsMs + " ms");
    }

    static class WaitThread extends Thread{
        @Override
        public void run() {
            super.run();
            // 锁对象为 mLockObject，主线程和子线程持有的锁对象为同一个，达到同步效果。
            synchronized (mLockObject){
                try {
                    System.out.println("子线程执行,3s 后执行 notifyAll");
                    Thread.sleep(3000);
                    // 显式的调用 notifyXX 方法，
                    mLockObject.notifyAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
执行日志：

```
主线程执行
主线程等待
子线程执行,3s 后执行 notifyAll
主线程继续 -> 等待耗时：3007 ms
```

可以看到在主线程在执行 mLockObject 的 wait() 方法后就进入到等待状态，在子线程中执行 mLockObject 的 notifyAll() 方法来唤醒等待池中睡眠的主线程，使主线程继续执行。


### 0x0002 sleep


该方法时 Thread 的静态方法，作用是 **使调用线程进入睡眠状态**。因为 sleep 方法时 Thread 的静态方法，因此它 **不能改变对象所持有的锁**，所以当在一个 synchronized 块中调用 sleep 方法时，虽然线程休眠了，但是所持有的对象锁并不会被释放，其他线程也无法访问这个对象(睡着也要霸占锁的主)。



### 0x003 join

等待目标线程 **执行完成后**， 再继续执行，即 **获得插队执行的权利**。

阻塞当前调用 join() 方法时 **所在的线程**，直到目标线程(调用 join 方法的线程)执行完毕，阻塞的线程得以继续执行。


```
    private static void joinTest() {
        // 目标线程 1
        Worker worker1 = new Worker("worker-1");
        // 目标线程 2
        Worker worker2 = new Worker("worker-2");
        worker1.start();

        long start = System.currentTimeMillis();
        try {
            System.out.println("启动线程 1");
            // 调用 worker1 中的 join 函数，主线程会阻塞直到 worker1 执行完成
            worker1.join();
            System.out.println("启动线程 2");
            // 启动线程2 ，调用 worker2 中的 join 函数，主线程会阻塞直到 worker2 执行完成
            worker2.start();
            worker2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("主线程继续执行");
        System.out.println("主线程等待线程 1 和线程 2 的时间为：" + (System.currentTimeMillis() - start) + " ms");
    }


    static class Worker extends Thread {
        public Worker(String name) {
            super(name);
        }

        @Override
        public void run() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("Work in: " + getName());
        }
    }

```

打印日志：

```
启动线程 1
Work in: worker-1
启动线程 2
Work in: worker-2
主线程继续执行
主线程等待线程 1 和线程 2 的时间为：4010 ms
```

可以看到在开启线程后(执行 start() 方法)，执行指定线程的 join() 方法，主线程会一直阻塞，直到目标线程执行完毕后，主线程才得以执行执行。

### 0x0004 yield


线程礼让。

使调用该函数的线程让出执行时间给其他已经处于就绪状态的线程。

目标线程有由 **运行状态** 转换为 **就绪状态**，让出执行权限，让其他线程得以优先执行，但是其他线程是否能够优先执行是位置的， 由 CPU 时间分片决定。


```

    private static void yieldTest(){
        YieldThread yieldThread1 = new YieldThread("thread-1");
        YieldThread yieldThread2 = new YieldThread("thread-2");
        yieldThread1.start();
        yieldThread2.start();
    }

    static class YieldThread extends Thread {
        
        public YieldThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            super.run();
            for (int i = 0; i < 5; i++) {
                System.out.printf("%s 优先级为：%d ---> %d\n", this.getName(), this.getPriority(), i);
                // 当 i = 2 时，调用当前线程执行 yield 函数
                if (i == 2) {
                    System.out.printf("%s 进入到就绪状态 ---> %d %s \n", this.getName(), i,this.getName());
                    Thread.yield();
                }
                if (i == 3) {
                    System.out.printf("%s 重新回到运行状态 ---> %d  %s \n", this.getName(), i, this.getName());
                }
            }
        }
    }

```

打印结果：

```
thread-1 优先级为：5 ---> 0
thread-1 优先级为：5 ---> 1
thread-1 优先级为：5 ---> 2
thread-2 优先级为：5 ---> 0
thread-2 优先级为：5 ---> 1
thread-2 优先级为：5 ---> 2
thread-2 进入到就绪状态 ---> 2 thread-2 
thread-1 进入到就绪状态 ---> 2 thread-1 
thread-2 优先级为：5 ---> 3
thread-2 重新回到运行状态 ---> 3  thread-2 
thread-2 优先级为：5 ---> 4
thread-1 优先级为：5 ---> 3
thread-1 重新回到运行状态 ---> 3  thread-1 
thread-1 优先级为：5 ---> 4
```

以上日志打印情况只是众多情况中的一种，此时所说的众多情况是在 i = 2 之前，因为线程获得 CPU 时间分片是随机的，所以这是众线程的执行顺序可以说是零乱的，但是当一个线程中执行 yield(),那么该线程就处于就绪状态，需要等待其他线程执行完毕后才可以继续执行。

此例中，
当 thread-2 执行到 i = 2 时：
    thread-2 处于就绪状态，thread-1 进入运行状态
当 thread-1 执行到 i = 2 时：
    thread-1 处于就绪状态，thread-2 进入运行状态，此时 thread-2 的状态不会再改变，会一直执行，直到结束。
当 thread-2 执行完毕时：
    thread-1 继续执行，直到结束。

当一个线程中执行 yield)() 方法，同时存在处于 yield  就绪状态的线程和正常就绪状态的线程，那么正常状态的线程会得到优先执行权利，直到所有的线程均处于 yield 就绪状态，那么 yield 状态的线程会按照进入此状态的顺序恢复正常状态，好绕啊，具体验证可以查看 [GitHub 代码](https://github.com/leeGYPlus/JavaCode/blob/master/src/thread/methods/Main.java) 中 yieldTest 方法以及注释。


可以看到 join 方法和 yield 方法的作用完全相反：

* 调用 join 方法的线程，将成功 **抢夺** 线程执行的权利
* 调用 yield 方法的线程，将 **让出** 自己线程执行的权利

---- 

**知识链接**

[Android 开发进阶:从小工到专家](https://book.douban.com/subject/26744163/)