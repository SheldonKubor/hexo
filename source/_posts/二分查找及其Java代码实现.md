---
title: 二分查找及其Java代码实现
date: 2019-08-31 14:38:11
tags: 算法
---
假设我们在词典中查找一个k开头的单词，我们会怎么做呢？
1. 从词典第一页开始一页一页的翻页，然后直到翻到k开头的单词。
2. 直接翻页到词典大概中间的位置，然后根据词典a-z排列规律，判断翻到的页在k之前，还是之后，然后继续翻页。

其实这就是一个查找问题，上面第二种方法就是 `二分查找`

我们再举一个例子：
我自己随便想一个 `1-100` 之间的数字，然后让你来猜，你每次猜测之后我都会告诉你，猜大了还是猜小了。（假设我心中想的数字是60）

1. 第一种方式：假设你从 `1` 开始依次往上猜，要猜到 `60` 你得猜60次才能猜到。这种方式就是 `简单查找`。（当然聪明如你肯定不会以这种方式来猜）
2. 第二种方式：从 `50` 开始，小了，排除`1-50`，排除了一半数字；接下来你猜 `75`，大了，又排除了一半数字`75-100`；接下来你猜 `63`，大了；依次类推，你每次猜测的都是中间的数字，从而每次都将余下的数字排除一半，无论我心中的所想的数字是几，你在7步之内就能猜到。（大家想想为什么是7次？给个提示，2的6次方是64,2的7次方是128。）

上述第二种方式就是 `二分查找` 。

一般而言，对于包含n个元素的列表，用二分查找最多需要 `logn` 步（log以2为底），用简单查找最多需要 `n` 步

接下来我们看看如何编写二分查找的Java代码，有两种方式，一种是利用循环，另一种是利用递归（递归是什么此篇文章就不展开讲了，之后再讲。）

另外需要注意的是，二分查找只能应用在排好顺序的数组或列表上。
```java

/**
* BinSearch
*/
public class BinSearch {
    public static void main(String[] args) {
        int[] arr = {1,4,8,16,45,48,78};
        int index = recursionBinarySearch(arr,48,0,arr.length-1);
        int index2 = commonBinarySearch(arr,48);
        System.out.println(index);
        System.out.println(index2);
    }
    //递归
    public static int recursionBinarySearch(int[] arr,int key,int low,int high){
        
        if(key < arr[low] || key > arr[high] || low > high){
            return -1;              
        }
        
        int middle = (low + high) / 2;          //初始中间位置
        if(arr[middle] > key){
            //比关键字大则关键字在左区域
            return recursionBinarySearch(arr, key,low,middle-1);
        }else if(arr[middle] < key){
            //比关键字小则关键字在右区域
            return recursionBinarySearch(arr, key,middle+1,high);
        }else {
            return middle;
        }
    }
    //循环
    public static int commonBinarySearch(int[] arr,int key){
        int low = 0;
        int high = arr.length - 1;
        int middle = 0;         //定义middle
        
        if(key < arr[low] || key > arr[high] || low > high){
            return -1;              
        }
        
        while(low <= high){
            middle = (low + high) / 2;
            if(arr[middle] > key){
                //比关键字大则关键字在左区域
                high = middle - 1;
            }else if(arr[middle] < key){
                //比关键字小则关键字在右区域
                low = middle + 1;
            }else{
                return middle;
            }
        }
        return -1;      //最后仍然没有找到，则返回-1
    }
}


```
