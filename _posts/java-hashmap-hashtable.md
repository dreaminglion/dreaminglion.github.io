---
layout: post
title:  "different hashmap hashtable"
categories: java
---

Java 中 hashmap 和 hashtable 的区别.

<!-- more -->

## 继承和实现区别
Hashtable是基于陈旧的Dictionary类，完成了Map接口；  
HashMap是Java 1.2引进的Map接口的一个实现（HashMap继承于AbstractMap,AbstractMap完成了Map接口。

## 线程安全不同
HashTable的方法是同步的，HashMap是未同步，所以在多线程场合要手动同步HashMap。

## 对null的处理不同
HashTable不允许null值(key和value都不可以),HashMap允许null值(key和value都可以)。即 HashTable不允许null值其实在编译期不会有任何的不一样，会照样执行，只是在运行期的时候Hashtable中设置的话回出现空指针异常。
HashMap允许null值是指可以有一个或多个键所对应的值为null。当get()方法返回null值时，即可以表示 HashMap中没有该键，也可以表示该键所对应的值为null。因此，在HashMap中不能由get()方法来判断HashMap中是否存在某个键，而应该用containsKey()方法来判断。

## 方法不同
HashTable有一个contains(Object value)，功能和containsValue(Object value)功能一样。

## 遍历方法
HashTable使用Enumeration，HashMap使用Iterator。

## 增长方式
HashTable中hash数组默认大小是11，增加的方式是 old*2+1。HashMap中hash数组的默认大小是16，而且一定是2的指数。

## 哈希值的使用不同，HashTable直接使用对象的hashCode，代码是这样的：

```
   int hash = key.hashCode();

　　int index = (hash & 0x7FFFFFFF) % tab.length;

　　而HashMap重新计算hash值，而且用与代替求模：

　　int hash = hash(k);

　　int i = indexFor(hash, table.length);

　　static int hash(Object x) {

　　int h = x.hashCode();

　　h += ~(h << 9);

　　h ^= (h >>> 14);

　　h += (h << 4);

　　h ^= (h >>> 10);

　　return h;

　　}

　　static int indexFor(int h, int length) {

　　return h & (length-1);

   }

```
