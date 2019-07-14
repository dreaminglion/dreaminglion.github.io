---
title: algorithm sort
---

java 选择、冒泡、快速排序，二分查找。

<!-- more -->

## 起泡排序

把一个数组的元素，安装从小到大的顺序排列。

```bash
/**
  * 冒泡排序
  * @param s
  */  
public void blisterSort(int[] s) {
  int n = s.length;
  for (int i = 0; i < n; i++) {
      /**
       * ① 7 个数比较6次，得到最大结果 所以 比较 n-1 次
       * ② 第二次剩下6个数比较，第三次剩下5个数比较 所以 n-i
       */
      for (int j = 0; j < n - 1 - i; j++) {
          // i bigger than i+1
          if (s[j] > s[j+1]) {
              int temp = s[j];
              s[j] = s[j + 1];
              s[j + 1] = temp;
          }
      }
  }          
}

// 数据源
public static int[] num = {9, 2, 6, 7};

// 调用
Sorter sorter = new Sorter();
sorter.blisterSort(num);
for (int i = 0; i < num.length; i++) {
    System.out.print(num[i]);
}        
```

解析：每执行一轮，得到一个数组最大的元素。数组剩下需要排序的元素为n-1。而且7个元素，只需要比较6次就可以得到最大值，所以流程如下：
for (int j = 0; j < n - 1 - 0; j++)
for (int j = 0; j < n - 1 - 1; j++)
for (int j = 0; j < n - 1 - 2; j++)
for (int j = 0; j < n - 1 - n; j++)

所以得到排序结果需要比较  
for (int i = 0; i < n; i++){for (int j = 0; j < n - 1 - 0; j++){}}

![](https://raw.githubusercontent.com/dreaminglion/dreaminglion.github.io/master/images/algorithm-blister-sort.png)

## 选择排序

```
for (int i = 0; i < arr.length - 1; i++) {
   for (int j = i + 1; j < arr.length; j++) {
      if (arr[i] > arr[j]) {
        int tem = arr[i];
        arr[i] = arr[j];
        arr[j] = tem;
      }
   }
}
```

## 快速排序

```
public class QuickSort {

   public static void main(String[] args) {
      int arr[] = { 34, 35, 18, 92, 46, 28, 55, 73, 64 };
      quick(arr, 0, arr.length - 1);
      for (int i : arr) {
        System.out.print(i + "\t");
      }
   }

   public static void quick(int arr[], int start, int end) {
      int i = start;
      int j = end;
      int temp = arr[start];
      while (i < j) {
        // 从后往前比较
        for (; j > i; j--) {
           if (arr[j] < temp) {
              arr[i] = arr[j];
              i++;
              break;
           }
        }
        for (; i < j; i++) {
           if (arr[i] > temp) {
              arr[j] = arr[i];
              j--;
              break;
           }
        }
        arr[i] = temp;
        if (i > start + 1) {
           quick(arr, start, i - 1);
        }
        if (j < end - 1) {
           quick(arr, j + 1, end);
        }
      }
   }
}
```

## 二分查找

二分查找法（折半查找法），前提：数组必须是有序的。

```
public static int halfSearch(int arr[],int key)
{
   int max,min,mid;
   min=0;max=arr.length-1;mid=(max+min)/2;
   while(key!=arr[mid])
   {
      if(key>arr[mid])
        min=mid+1;
      else
        max=mid-1;
      if(max<min)
        return -1;
      mid=(max+min)/2;
   }
   return mid;
}
```
