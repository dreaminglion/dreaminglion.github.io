---
title: algorithm recursion
---

java递归算法的简单实例，与运行分析。

<!-- more -->

## 线性递归

定义：线性递归是最简单的递归形式，这类方法的每个实例只能递归地调用自己至多一次。比如，有 的时候，某些算法问题的输入可以分解为第一个(或最后一个)元素以及剩余的部分，而且剩余的 部分又具有与原输入相同的结构，此时就可以利用线性递归。

```bash
// 数组倒置
public void reverseArray(int[] list, int s, int e) {
        if (s < e) {
            int temp = list[e];
            list[e] = list[s];
            list[s] = temp;

            reverseArray(list, s + 1, e - 1);
        }
}

// 数据源
public static int[] num = {9, 2, 6, 7};

// 调用
Recursion recursion = new Recursion();
recursion.reverseArray(num,0,num.length - 1);
for (int i = 0; i < num.length; i++) {
      System.out.print(num[i]);
}
```

解析分析，打印结果：

s - 0  e - 3
s - 1  e - 2
s - 2  e - 1
7629

通过s + 1, e - 1，从数组的两头取元素进行交换，当s < e时候结束递归。

## 尾递归改造非递归

原因：利用递归可以编写出简洁而优美的算法。然而，这些优点并非没有代价，为此 计算机通常需要使用更多的空间、需要花费额外的时间以跟踪递归调用的过程。在对运行速度要求 极高、对存储空间需要精打细算时，往往需要将算法的递归版本改写成非递归的版本。

定义：请注意，即使递归调用语句出现在方法体的最后一行，也不见得就是尾递归。严格地说，只有 当该方法的任何一次执行(当然，除平凡的递归基外)都以这一递归调用结束时，才属于尾递归。

```
// 数组倒置
public void iterativeReverseArray(int[] list, int s, int e){
        while (s < e){
            int temp = list[e];
            list[e] = list[s];
            list[s] = temp;

            s++;
            e--;
        }
}
```

## 二分递归

- 定义：有的时候，算法需要将一个大问题分解为两个子问题，然后分别通过递归调用来求解，这种情 况称作二分递归(Binary recursion )。
- 思路：以居中的元素为界，将数组一分为二;递归地对这两个子数组分别求和，它们相加就得到 了原数组的总和。

```bash
//算法结构
public int binarySum(int[] list, int i,int n){
        System.out.println("i - " + i + "  n - " + n);
        if (n <=1) {
            return list[i];
        } else {
            return binarySum(list,i,n/2) + binarySum(list,i + n/2,n/2);
        }
}

// 数据源
public static int[] num = {9, 2, 6, 7};

// 调用
Recursion recursion = new Recursion();
int m = recursion.binarySum(num, 0, num.length);
System.out.print("m - " + m);
```

解析分析：使用System.out.println("i - " + i + "  n - " + n);看函数执行了多少次，以及对应的参数。

i - 0  n - 4  
i - 0  n - 2  
i - 0  n - 1
i - 1  n - 1  
i - 2  n - 2  
i - 2  n - 1
i - 3  n - 1

binarySum(num,0,4);
binarySum(num,0,2) + binarySum(num,2,2);
binarySum(num,0,1) + binarySum(num,1,1);
binarySum(num,2,1) + binarySum(num,3,1);
当n=1时，i={0，1，2，3} return num[0] + num[1] + num[2] + num[3];

## 多分支递归

看着名字，你就知道这个非常、非常滴麻烦啦~
