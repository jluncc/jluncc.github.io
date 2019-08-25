---
title: Java 实现二分查找
author: 陈加菲
date: 2019-06-13 22:48:47
tags:
 - 算法
---

### 介绍

二分查找，又称折半查找。其查找思路为在一个 **有序** 排列中，每次拿中间位置的数字跟目标值判断，相等返回该中间值，大于则在前半段重复判断，小于则在后半段重复判断，直至找到目标值或全部判断完毕。

### 代码实现

#### 递归实现与非递归实现

``` java
public class BinarySearch {

    /**
     * 递归实现二分查找
     * @param array 待查找的有序序列
     * @param target 目标值
     * @param low 头指针
     * @param high 尾指针
     * @return 目标值下标，若没有找到目标值返回-1
     */
    public static int binarySearch(int[] array, int target, int low, int high) {
        if (array.length <= 0 || target < array[low]
                || target > array[high] || low > high) {
            return -1;
        }

        int middle = (low + high) / 2;
        if (target > array[middle]) {
            return binarySearch(array, target, middle + 1, high);
        } else if (target < array[middle]) {
            return binarySearch(array, target, low, middle - 1);
        } else {
            return middle;
        }
    }

    /**
     * 非递归实现的二分查找
     * @param array 待查找的有序序列
     * @param target 目标值
     * @return 目标值下标，若没有找到目标值返回-1
     */
    public static int binarySearchNotRecursion(int[] array, int target) {
        if (array.length <= 0) {
            return -1;
        }

        int low = 0;
        int high = array.length - 1;
        int middle = 0;

        while (low <= high) {
            middle = (low + high) / 2;
            if (target == array[middle]) {
                return middle;
            } else if (target > array[middle]) {
                low = middle + 1;
            } else {
                high = middle - 1;
            }
        }
        return -1;
    }

    public static void main(String[] args) {
        int[] array = new int[]{2, 45, 47, 99, 100};
        int target = 1;
        int low = 0;
        int high = array.length - 1;
        System.out.println("递归实现二分查找，找到的下标：" + binarySearch(array, target, low, high));
        System.out.println("非递归实现二分查找，找到的下标：" + binarySearchNotRecursion(array, target));
    }
}
```

#### 注意点

注意在 int 中两数相加，为防止溢出可以使用 **位运算**，如下示例

``` java
(a + b) >> 1 代替 (a + b) / 2
```

### 复杂度分析

#### 时间复杂度

两种情况下都为 O(log2 N)

#### 空间复杂度

非递归方式，没有使用辅助空间，因此空间复杂度是 O(1)；

递归方式，因为递归需要使用辅助空间，而递归的次数和深度都是 log2N，每次所需要的辅助空间都是常数级别的，所以空间复杂度为 O(log2N )。

### 参考文章

[Java实现二分查找-两种方式](https://blog.csdn.net/maoyuanming0806/article/details/78176957)
[Java安全防溢出的两整数平均值算法](https://blog.csdn.net/as1072966956/article/details/79982623)

**END**
