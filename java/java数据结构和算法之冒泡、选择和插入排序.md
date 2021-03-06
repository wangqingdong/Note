---
title 冒泡、选择、插入排序算法
category java数据结构与算法
---
# 冒泡、选择、插入排序算法
上一篇博客我们实现的数组结构是无序的，也就是纯粹按照插入顺序进行排列，那么如何进行元素排序，本篇博客我们介绍几种简单的排序算法。

## 一、冒泡排序

这个名词的由来很好理解，一般河水中的冒泡，水底刚冒出来的时候是比较小的，随着慢慢向水面浮起会逐渐增大，这物理规律不作过多解释，了解即可。

### 1.1、算法描述

冒泡算法的运作规律如下：

1. 比较相邻的元素。如果第一个比第二个大，就交换他们两个
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数（也就是第一波冒泡完成）
3. 针对所有的元素重复以上的步骤，除了最后一个
4. 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较

### 1.2、代码实现

```java
public class BubbleSort {
    public static int[] sort(int[] array){
        //这里for循环表示总共需要比较多少轮
        for(int i = 1 ; i < array.length; i++){
            //设定一个标记，若为true，则表示此次循环没有进行交换，也就是待排序列已经有序，排序已经完成。
            boolean flag = true;
            //这里for循环表示每轮比较参与的元素下标
            //对当前无序区间array[0......length-i]进行排序
            //j的范围很关键，这个范围是在逐步缩小的,因为每轮比较都会将最大的放在右边
            for(int j = 0 ; j < array.length-i ; j++){
                if(array[j]>array[j+1]){
                    int temp = array[j];
                    array[j] = array[j+1];
                    array[j+1] = temp;
                    flag = false;
                }
            }
            if(flag){
                break;
            }
        }
        return array;
    }
}
```
本来应该是8轮排序的，这里我们只进行了7轮排序，因为第7轮排序之后已经是有序数组了

### 1.3、算法分析

**冒泡排序解释**

冒泡排序是由两个for循环构成，第一个for循环的变量 i 表示总共需要多少轮比较，第二个for循环的变量 j 表示每轮参与比较的元素下标【0,1，......，length-i】，因为每轮比较都会出现一个最大值放在最右边，所以每轮比较后的元素个数都会少一个，这也是为什么 j 的范围是逐渐减小的。

**冒泡排序性能分析**

假设参与比较的数组元素个数为N,则第一轮排序有N-1次比较，第二轮有N-2次，如此类推，这种序列的求和公式为：

（N-1）+（N-2）+...+1 = N*（N-1）/2

当N的值很大时，算法比较次数约为N<sup>2</sup>/2次比较,忽略减1.

假设数据是随机的，那么每次比较可能要交换位置，可能不会交换，假设概率为50%，那么交换次数为N<sup>2</sup>/4。不过如果是最坏的情况，初始数据是逆序的，那么每次比较都要交换位置。

交换和比较次数都和N<sup>2</sup>成正比。由于常数不算大O表示法中，忽略2和4，那么冒泡排序运行都需要O(N<sup>2</sup>)时间级别。

其实无论何时，只要看见一个循环嵌套在另一个循环中，我们都可以怀疑这个算法的运行时间为O(N<sup>2</sup>)级，外层循环执行N次，内层循环对每一次外层循环都执行N次（或者几分之N次）。这就意味着大约需要执行N<sup>2</sup>次某个基本操作

## 二、选择排序

### 2.1、算法描述

选择排序是每一次从待排序的数据元素中选出最小的一个元素，存放在序列的起始位置，直到全部待排序的数据元素排完。分为三步：
1. 从待排序序列中，找到关键字最小的元素
2. 如果最小元素不是待排序序列的第一个元素，将其和第一个元素互换
3. 从余下的 N - 1 个元素中，找出关键字最小的元素，重复(1)、(2)步，直到排序结束

### 2.2、代码实现

```java
public class ChoiceSort {
    public static int[] sort(int[] array){
        //总共要经过N-1轮比较
        for(int i = 0 ; i < array.length-1 ; i++){
            int min = i;
            //每轮需要比较的次数
            for(int j = i+1 ; j < array.length ; j++){
                if(array[j]<array[min]){
                    min = j;//记录目前能找到的最小值元素的下标
                }
            }
            //将找到的最小值和i位置所在的值进行交换
            if(i != min){
                int temp = array[i];
                array[i] = array[min];
                array[min] = temp;
            }
        }
        return array;
    }
}
```

### 2.3、算法分析
选择排序和冒泡排序执行了相同次数的比较：N*(N-1)/2,但是至多只进行了N次交换。

当N值很大时，比较次数是主要的，所以和冒泡排序，用大O表示是O(N<sup>2</sup>)时间级别。但是由于选择排序交换的次数少，所以选择排序无疑是比冒泡排序快的。当N值较小时，如果交换时间比选择时间大得多，那么选择排序是相当快的。

## 三、插入排序

### 3.1、算法描述

直接插入排序基本思想是每一步将一个待排序的记录，插入到前面已经排好序的有序序列中去，直到插完所有元素为止。

插入排序还分为直接插入排序、二分插入排序、链表插入排序、希尔排序等等，这里我们只是以直接插入排序讲解，后面讲高级排序的时候会将其他的。

### 3.2、代码实现

```java
public class InsertSort {
    public static int[] sort(int[] array){
        int j;
        //从下标为1的元素开始选择合适的位置插入，因为下标为0的只有一个元素，默认是有序的
        for(int i = 1 ; i < array.length ; i++){
            int tmp = array[i];//记录要插入的数据
            j = i;
            while(j > 0 && tmp < array[j-1]){//从已经排序的序列最右边的开始比较，找到比其小的数
                array[j] = array[j-1];//向后挪动
                j--;
            }
            array[j] = tmp;//存在比其小的数，插入
        }
        return array;
    }
}
```

### 3.3、算法分析

在第一轮排序中，它最多比较一次，第二轮最多比较两次，一次类推，第N轮，最多比较N-1次。因此有 1+2+3+...+N-1 = N*（N-1）/2。

　　假设在每一轮排序发现插入点时，平均只有全体数据项的一半真的进行了比较，我们除以2得到：N*（N-1）/4。用大O表示法大致需要需要 O(N<sup>2</sup>) 时间级别。

　　复制的次数大致等于比较的次数，但是一次复制与一次交换的时间耗时不同，所以相对于随机数据，插入排序比冒泡快一倍，比选择排序略快。

　　这里需要注意的是，如果要进行逆序排列，那么每次比较和移动都会进行，这时候并不会比冒泡排序快。

## 四、总结

上面讲的三种排序，冒泡、选择、插入用大 O 表示法都需要 O(N<sup>2</sup>) 时间级别。一般不会选择冒泡排序，虽然冒泡排序书写是最简单的，但是平均性能是没有选择排序和插入排序好的。

选择排序把交换次数降低到最低，但是比较次数还是挺大的。当数据量小，并且交换数据相对于比较数据更加耗时的情况下，可以应用选择排序。

在大多数情况下，假设数据量比较小或基本有序时，插入排序是三种算法中最好的选择。

后面我们会讲解高级排序，大O表示法的时间级别将比O(N2)小。　