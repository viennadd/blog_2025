--- 
title: Leetcode - Median of Two Sorted Arrays
date: 2016-04-07 23:19:05
authors: [alexchen]
tags: [leetcode, algorithm]
categories: [刷题]

---

对这题想到一个方法，既然是两个 Sorted Arrays, 用 Merge Sort 类似的归并方法组合两个数组就可以了, 根据总长度的奇偶抽取第 N 大的数值出来就完成啦。但这样做的话运行时间是 O(m+n / 2), 题目要求 O(log(m+n))

第一个想法就是二分, 参考了几个博文之后, 先准备一个 Kth 函数用于寻找两个 sorted array 的第 K 小的数, 然后中位数就很容易了, 反而一开始追求二分中位数似乎会有不少 corner cases

那先把问题换成两个有序数组的第 K 小的数值

有以下步骤
* 有序 vector v1 和 v2, 求第 K 小
* 二分思想是每次从 v1 和 v2 排除一半的可选元素
* 有 x, y > 0 && x + y == k, 
* x, y 的取值也是有讲究, x = len(v1) / (len(v1) + len(v2)) * k, 这个意思是 len(v1) 占总 size 的百分比, 再乘以 k 的话就可以确保 x 的取值不会超出 v1 的数组范围, 这里简化了, 实际代码需要注意一些边界问题
* 如果 **v1[x] < v2[y]**, 则 v1[0 .. x] 的元素都不可能是第 K 小, 他们都肯定比第 K 的数值要小了, 我们则可以排除掉这些数值, 
* **v1[x] > v2[y]** 的话都是同样的思想
* 调整 v1, v2 的长度, 下表值, k 的值等, 然后重复这些取值然后比较的步骤,
* 调整完 size 和 下标后, 检查一下 k 是否下降到 1, 第 1 小的值是 min(v1[0], v2[0]), 还有如果其中一个 vector 的 size 下降到 0, 那可以直接返回另一个 vector 的第 k 个值
* 如果是 **v1[x] == v2[y]** 的话, x + y == k, v1[x] == v2[y], 那这个数值已经是第 K 小了, 这里是一个 base case

大家可以 Leetcode 测试一下自己的代码 [https://leetcode.com/problems/median-of-two-sorted-arrays/](https://leetcode.com/problems/median-of-two-sorted-arrays/)

下面是我提交的代码
<!-- truncate -->

```cpp
#ifndef LEETCODE_MEDIAN_OF_TWO_SORTED_ARRAYS_H
#define LEETCODE_MEDIAN_OF_TWO_SORTED_ARRAYS_H

#include <vector>
#include <algorithm>
#include <cassert>

using namespace std;

class Solution {
public:
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {

        if (nums1.size() == 0)
            return median(nums2);
        if (nums2.size() == 0)
            return median(nums1);

        auto total_size = nums1.size() + nums2.size();
        if (total_size % 2)
        {
            return findKtnElement(nums1, nums2, total_size / 2 + 1);
        }
        else
        {
            return (findKtnElement(nums1, nums2, total_size / 2) +
                    findKtnElement(nums1, nums2, total_size / 2 + 1)) / 2.0;
        }
    }

    double median(vector<int> &v)
    {
        if (v.size() % 2)
        {
            return v.at(v.size() / 2);
        }
        else
        {
            return (v.at(v.size() / 2 - 1) +
                    v.at(v.size() / 2)) / 2.0;
        }
    }


    int& min(int &a, int &b)
    {
        return a < b ? a : b;
    }

    int& findKtnElement(vector<int> &v1, vector<int> &v2, int k)
    {
        auto v1_len = v1.size();
        auto v2_len = v2.size();
        if (v1_len == 0)
            return v2.at(k);
        if (v2_len == 0)
            return v1.at(k);

        if (k == 1)
            return min(v1.at(0), v2.at(0));

        assert(v1_len + v2_len >= k);

        auto v1_begin = v1.begin();
        auto v2_begin = v2.begin();

        while (true) {

            int i, j;
            i = v1_len / (v1_len + v2_len + 0.0) * (k - 1);
            j = k - i - 2;

            auto a = *(v1_begin + i);
            auto b = *(v2_begin + j);
            if (a < b) {
                /* 丢弃 v1 的前 i + 1 (0 base) 个 */
                v1_begin += i + 1;
                v1_len -= i + 1;
                k -= i + 1;
            } else if (b < a) {
                v2_begin += j + 1;
                v2_len -= j + 1;
                k -= j + 1;
            } else {
                /* 相等的话就不用介意了, 随意返回一个都是第 k 小 */
                return *(v1_begin + i);
            }

            if (v1_len == 0)
                return v2.at(k - 1);
            if (v2_len == 0)
                return v1.at(k - 1);

            if (k == 1)
                return min(*v1_begin, *v2_begin);
        }

    }


};

#endif //LEETCODE_MEDIAN_OF_TWO_SORTED_ARRAYS_H

```

