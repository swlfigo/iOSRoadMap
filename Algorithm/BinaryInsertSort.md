# 二分法插入排序

二分法插入排序，简称二分排序，是在插入第i个元素时，对前面的0～i-1元素进行折半，先跟他们中间的那个元素比，如果小，则对前半再进行折半，否则对后半进行折半，直到left<right，然后再把第i个元素前1位与目标位置之间的所有元素后移，再把第i个元素放在目标位置上。



(一)概念及实现

二分查找插入排序的原理：是直接插入排序的一个变种，区别是：在有序区中查找新元素插入位置时，为了减少元素比较次数提高效率，采用二分查找算法进行插入位置的确定。

 

具体如下**（实现为升序）**：

设数组为a[0…n]。

1. 将原序列分成有序区和无序区。a[0…i-1]为有序区，a[i…n] 为无序区。（i从1开始）

2. 从无序区中取出第一个元素，即a[i]，使用二分查找算法在有序区中查找要插入的位置索引j。

3. 将a[j]到a[i-1]的元素后移，并将a[i]赋值给a[j]。

4. 重复步骤2~3，直到无序区元素为0。



(二)算法复杂度

1. 时间复杂度：O(n^2)

二分查找插入位置，因为不是查找相等值，而是基于比较查插入合适的位置，所以必须查到最后一个元素才知道插入位置。

二分查找最坏时间复杂度：当2^X>=n时，查询结束，所以查询的次数就为x，而x等于log2n**（以2为底，n的对数）**。即O(log2n)

所以，二分查找排序比较次数为：x=log2n

二分查找插入排序耗时的操作有：比较 + 后移赋值。时间复杂度如下：

1)    最好情况：查找的位置是有序区的最后一位后面一位，则无须进行后移赋值操作，其比较次数为：log2n 。即O(log2n)

2)    最坏情况：查找的位置是有序区的第一个位置，则需要的比较次数为：log2n，需要的赋值操作次数为[n(n-1)/2](http://zhidao.baidu.com/link?url=D1uGyXzk3biP8YR-tKHq1_YHgZZmojMd0XzWlPxSWoYdhaTZdlRyd-FXaVqGNaYpgVHe0Lh3mMKPCwNH2E5C6q)加上 (n-1) 次。即O(n^2)

3)    渐进时间复杂度（平均时间复杂度）：O(n^2)

2. 空间复杂度：O(1)

从实现原理可知，二分查找插入排序是在原输入数组上进行后移赋值操作的（称“就地排序”），所需开辟的辅助空间跟输入数组规模无关，所以空间复杂度为：O(1)

 

(三)稳定性

二分查找排序是稳定的，不会改变相同元素的相对顺序。





```java
public static void advanceInsertSortWithBinarySearch(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int temp = arr[i];
        int low = 0, high = i - 1;
        int mid = -1;
        while (low <= high) {            
            mid = low + (high - low) / 2;            
            if (arr[mid] > temp) {               
                high = mid - 1;            
            } else { // 元素相同时，也插入在后面的位置                
                low = mid + 1;            
            }        
        }        
        for(int j = i - 1; j >= low; j--) {            
            arr[j + 1] = arr[j];        
        }        
        arr[low] = temp;    
    }
}
```



# Reference

[优化的直接插入排序（二分查找插入排序，希尔排序）](https://www.cnblogs.com/heyuquan/p/insert-sort.html)

