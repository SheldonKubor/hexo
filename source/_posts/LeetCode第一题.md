---
title: LeetCode第一题-TwoSum
date: 2019-10-01 20:14:26
tags: [算法,LeetCode,面试题]
---

刷了无数遍的LeetCode第一题，为啥刷了无数遍呢，因为每次想提高自己算法与数据结构能力的时候我都会下定决心来LeetCode刷题，而每次刷题，都是从第一题开始...

不多扯淡，直接开题。

题目要求是这样的：
```
Given an array of integers, return indices of the two numbers such that they add up to a specific target.

You may assume that each input would have exactly one solution, and you may not use the same element twice.
```


啥意思呢，帮英文不好的同学翻译一下：

给定一个整数数组 `nums` 和一个目标值 `target`，请你在该数组中找出和为目标值的那两个整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

例如：
```
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
```

首先，一般人第一个想到的方法就是暴力破解，也就是直接循环两次数组，遍历数组中每个元素 `x`，再次遍历数组，并查找是否存在一个值与 `target - x` 相等的目标元素。

先亮出代码，我们再来讨论程序性能。

```Java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        for (int i = 0; i < nums.length; i++) {
            for (int j = i + 1; j < nums.length; j++) {
                if (nums[j] == target - nums[i]) {
                    return new int[] { i, j };
                }
            }
        }
        throw new IllegalArgumentException("No two sum solution");
    }
}
```
然后我们来讨论一下此种解决方式的代码性能，分析一段代码的性能，主要从时间复杂度，以及空间复杂度两个方面来考虑。
简单解释一下什么是时间复杂度，什么是空间复杂度。

时间复杂度指的是解决一个问题，程序所需要进行多少次操作。
空间复杂度指的是解决一个问题，所需要的额外空间。

上述算法的复杂度为：

时间复杂度：O(n<sup>2</sup>)，对于每个元素（一共`n`个），我们试图通过遍历数组的其余部分（其余的`n-1`个元素）来寻找它所对应的目标元素，这将耗费`O(n)`的时间。遍历数组中每个元素时间复杂度为O(n)，遍历数组其余部分的时间复杂度为`O(n)`（准确的来说为`O(n-1)`，但是因为`1`是常数，与n比起来可以忽略不计，所以简写为`O(n)`），因此时间复杂度为O(n<sup>2</sup>)。

空间复杂度：`O(1)`。

可以看到，此种方法的时间复杂度很高，因为这道题的本质还是在数组中查找元素，这相当于两层嵌套循环。说到查找元素，可能同学们会想到，可以用二分查找啊，二分查找不比简单的for循环速度快多了。确实，二分查找比暴力查找速度快得多，但是，二分查找只适用于有序数组，此题目中数组并不是有序的，所以不能使用二分查找。

那么，这道题有没有更加快速的解法呢？当然是有的。

上面说了，这道题的本质还是在数组中查找元素，重点在于查找，我们的目的是缩短查找的时间。那么有什么方法可以缩短查找时间呢？学过数据结构的同学应该都知道或者听说过一种数据结构--`散列表`（又叫做`哈希表`）。这种数据结构，在无冲突的情况下，查找元素的时间复杂度为`O(1)`，比暴力循环数组的`O(n)`快了很多。那么我们如何利用它来解决这个问题呢？

一个简单的实现，使用了两次迭代。在第一次迭代中，我们将每个元素的值和它的索引添加到表中。然后，在第二次迭代中，我们将检查每个元素所对应的目标元素`(target - nums[i])`是否存在于表中。注意，该目标元素不能是 `nums[i]` 本身！

```Java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            map.put(nums[i], i);
        }
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (map.containsKey(complement) && map.get(complement) != i) {
                return new int[] { i, map.get(complement) };
            }
        }
        throw new IllegalArgumentException("No two sum solution");
    }
}
```

`注：这里使用了Java中的集合类HashMap，其实HashMap本质上就是哈希表`

复杂度分析：

时间复杂度：`O(n)`，
我们把包含有 `n` 个元素的列表遍历两次。由于哈希表将查找时间缩短到 `O(1)` ，所以时间复杂度为 `O(n)`。

空间复杂度：`O(n)`，
所需的额外空间取决于哈希表中存储的元素数量，该表中存储了 `n` 个元素。

可以看到，我们将程序的时间复杂度从O(n<sup>2</sup>)降到了`O(n)`，这说明我们的程序变得更快了，但是空间复杂度从`O(1)`变成了`O(n)`，占用的空间更多了。我们使用空间换取了时间，在现在程序效率要求较高，而内存空间或者硬盘空间成本日益廉价的情况下，如果没有特殊要求，我们可以不关注程序的空间复杂度，只关心程序的时间复杂度。


上面哈希表的解法，我们使用了两遍哈希表，第一遍是把数组中的元素放到哈希表中，也就是建表；第二遍从哈希表中进行查找。
那么我们能不能使用一次哈希表就解决问题呢？

我们看一下下面的代码
```Java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (map.containsKey(complement)) {
                return new int[] { map.get(complement), i };
            }
            map.put(nums[i], i);
        }
        throw new IllegalArgumentException("No two sum solution");
    }
}
```
事实证明，我们可以一次完成。在进行迭代并将元素插入到表中的同时，我们还会回过头来检查表中是否已经存在当前元素所对应的目标元素。如果它存在，那我们已经找到了对应解，并立即将其返回。

复杂度分析：

时间复杂度：`O(n)`，
我们只遍历了包含有 `n` 个元素的列表一次。在表中进行的每次查找只花费 `O(1)` 的时间。

空间复杂度：`O(n)`，
所需的额外空间取决于哈希表中存储的元素数量，该表最多需要存储 `n` 个元素。

可以看到，这种方法的时间复杂度与空间复杂度，和两遍哈希表的方法是一样的，程序性能没有什么差别。但这种方法只用了一次for循环，看起来就很高端，看起来就要比两遍哈希表的方法效率要高（只是看起来要高，其实效率提高很有限。。），为啥还要讲最后一种解法呢？因为很多面试官就喜欢这种看起来高端的解法。（当然，只要你不用第一中方法暴力求解，面试官还是会欣赏你的。。。）





