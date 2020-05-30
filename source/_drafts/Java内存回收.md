---
title: Java内存回收
tags:
---

## GC Root 对象


### 1. 虚拟机栈（栈帧中的局部变量）中引用的对象作为 GC


局部变量作为 GC Root 在运行方法时，执行 GC ，局部变量对象不会被回收，但是如果方法执行完毕，那么该方法的栈帧会从该虚拟机栈中移除，此时执行 GC 则可以回收该方法中的局部变量，但是如果该局部变量持有其他对象，则具体情况具体分析。


### 2. 静态变量引用对象作为 GC Root

只有将 静态变量引用对象 置为 null 时，执行 GC 才会释放该对象持有内存空间。

### 3. 将活跃的线程作为 GC Root

正在运行中的线程会作为 RC Root，在线程没有执行完成之前，线程持有的对象的内存对象不会被回收，只有相应的对象置空后，GC 才能够回收相应的内存。


CustomRunnable customRunnable = new CustomRunnable();
Thread thread = new Thread(customRunnable);
thread.start();


在 Thread 执行完毕后，操作 customRunnable = nnull，那么 customRunnable 持有的对象的内存空间在 GC时才可以被清除。