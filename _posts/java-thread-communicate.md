---
layout: post
title:  "线程间通信 wait notify"
categories: java
---

不同的线程完成对，同一个资源的访问。

<!-- more -->

## 等待/唤醒机制，涉及方法：
**前提：这些方法都必须定义在同步中，必须是多个线程使用同一把锁。**
1. wait()：让线程处于冻结状态，被wait的线程会被存储到线程池中。只有被同一个对象调用 notify 或者 notifyAll 方法才能唤醒此线程.
2. notify()：唤醒线(程池中)同一对象调用 wait 处于休眠状态的线程（任意的）。
3. notifyAll()：唤醒线程池中的使用同一对象调用 wait 出于休眠状态的所有线程。

## 为什么操作线程的方法wait，notify，notifyAll定义在Object类中？
因为这些方法是监视器的方法，监视器其实就是锁。锁可以使任意的对象，任意的对象调用的方法一定定义在Object类中。

## 线程间的通讯：多个线程在处理同一个资源，但任务却不同。

```
class Resource {
   private String name;
   private String sex;
   private boolean flag = false;

   public synchronized void set(String name, String sex) {
      if (flag)
        try {
           this.wait();
} catch (InterruptedException e) {}
      this.name = name;
      this.sex = sex;
      flag = true;
      this.notify();
   }

   public synchronized void out() {
      if (!flag)
        try {
           this.wait();
} catch (InterruptedException e) {}
      System.out.println(name + "...+...." + sex);
      flag = false;
      notify();
   }
}

// 输入
class Input implements Runnable {
   Resource r;

Input(Resource r) {
      this.r = r;
   }

   public void run() {
      int x = 0;
      while (true) {
        if (x == 0) {
           r.set("mike", "nan");
} else {
           r.set("丽丽", "女女女女女女");
        }
        x = (x + 1) % 2;
      }
   }
}

// 输出
class Output implements Runnable {

Resource r;

Output(Resource r) {
      this.r = r;
   }

   public void run() {
      while (true) {
        r.out();
      }
   }
}

class ResourceDemo3 {
   public static void main(String[] args) {
      Resource r = new Resource();
      Input in = new Input(r);
      Output out = new Output(r);
      Thread t1 = new Thread(in);
      Thread t2 = new Thread(out);
      t1.start();
      t2.start();
   }
}

```
