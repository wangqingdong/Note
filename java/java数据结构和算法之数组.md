---
title 数组
category java数据结构与算法
---

# 数组

上篇我们简单介绍了数据结构和算法的概念。本篇介绍数据结构的鼻祖——数组，可以说数组几乎能表示一切的数据结构，在每一门编程语言中，数组都是重要的数据结构，当然每种语言对数组的实现和处理也不相同，但是本质是都是用来存放数据的的结构，这里以Java语言为例，来详细介绍Java语言中数组的用法。

## 一、简介

在java中，数组是用来存放同一种数据类型的集合，注意只能存放同一种数据类型（Object类型数组除外）。

### 1.1、数组的声明

**第一种方式**
```java
//数据类型[] 数组名称=new 数据类型[数组长度];
int [] arr = new int[3];
arr[0]=1;
arr[1]=2;
arr[2]=3;
```
这里[]可以放在数组名称的前面，也可以放在数组名称的后面，推荐放在数组名称前面，这样看上去 数据类型 [] 表示的很明显是一个数组类型，而放在数组名称后面，则不是那么直观。

**第二种方式：**

```java
//数据类型[] 数组名称={数组元素1，数组元素2，...};
int[] arr={1,2,3};
```
这种方式声明数组的同时直接给定了数组的元素，数组的大小由给定的数组元素个数决定。

### 1.2、访问数组元素以及给数组元素赋值

数组是存在下标索引的，通过下标可以获取指定位置的元素，数组小标是从0开始的，也就是说下标0对应的就是数组中第1个元素，可以很方便的对数组中的元素进行存取操作。

前面数组的声明的两种方式，我们在声明数组的同时，也进行了初始化赋值。

```java
//访问arr的第一个元素
System.out.println(arr[0]);
```
上面的myArray 数组，我们只能赋值三个元素，也就是下标从0到2，如果你访问 myArray[3] ，那么会报数组下标越界异常。

### 1.3、数组遍历
数组有个 length 属性，是记录数组的长度的，我们可以利用length属性来遍历数组。
```java
int[] arr={1,2,3};
for(int i=0;i<arr.length;i++){
    System.out.println(arr[i]);
}
```

## 二、用类封装数组实现数据结构
上一篇介绍了一个数据结构必须具有一下基本功能：
- 如何插入一条新的数据项
- 如何寻找某一特定的数据项
- 如何删除某一特定的数据项
- 如何迭代的访问各个数据项，以便进行显示或其他操作
  
而我们知道了数组的简单用法，现在用类的思想封装一个数组，实现上面的四个基本功能：

ps:假设操作人是不会添加重复元素的，这里没有考虑重复元素，如果添加重复元素了，后面的查找，删除，修改等操作只会对第一次出现的元素有效。

```java
public class MyArray {
    //定义一个数组
    private int [] intArray;
    //定义数组的实际有效长度
    private int elems;
    //定义数组的最大长度
    private int length;
     
    //默认构造一个长度为50的数组
    public MyArray(){
        elems = 0;
        length = 50;
        intArray = new int[length];
    }
    //构造函数，初始化一个长度为length 的数组
    public MyArray(int length){
        elems = 0;
        this.length = length;
        intArray = new int[length];
    }
     
    //获取数组的有效长度
    public int getSize(){
        return elems;
    }
     
    /**
     * 遍历显示元素
     */
    public void display(){
        for(int i = 0 ; i < elems ; i++){
            System.out.print(intArray[i]+" ");
        }
        System.out.println();
    }
     
    /**
     * 添加元素
     * @param value,假设操作人是不会添加重复元素的，如果有重复元素对于后面的操作都会有影响。
     * @return添加成功返回true,添加的元素超过范围了返回false
     */
    public boolean add(int value){
        if(elems == length){
            return false;
        }else{
            intArray[elems] = value;
            elems++;
        }
        return true;
    }
     
    /**
     * 根据下标获取元素
     * @param i
     * @return查找下标值在数组下标有效范围内，返回下标所表示的元素
     * 查找下标超出数组下标有效值，提示访问下标越界
     */
    public int get(int i){
        if(i<0 || i>elems){
            System.out.println("访问下标越界");
        }
        return intArray[i];
    }
    /**
     * 查找元素
     * @param searchValue
     * @return查找的元素如果存在则返回下标值，如果不存在，返回 -1
     */
    public int find(int searchValue){
        int i ;
        for(i = 0 ; i < elems ;i++){
            if(intArray[i] == searchValue){
                break;
            }
        }
        if(i == elems){
            return -1;
        }
        return i;
    }
    /**
     * 删除元素
     * @param value
     * @return如果要删除的值不存在，直接返回 false;否则返回true，删除成功
     */
    public boolean delete(int value){
        int k = find(value);
        if(k == -1){
            return false;
        }else{
            if(k == elems-1){
                elems--;
            }else{
                for(int i = k; i< elems-1 ; i++){
                    intArray[i] = intArray[i+1];
                   
                }
                 elems--;
            }
            return true;
        }
    }
    /**
     * 修改数据
     * @param oldValue原值
     * @param newValue新值
     * @return修改成功返回true，修改失败返回false
     */
    public boolean modify(int oldValue,int newValue){
        int i = find(oldValue);
        if(i == -1){
            System.out.println("需要修改的数据不存在");
            return false;
        }else{
            intArray[i] = newValue;
            return true;
        }
    }
 
}
```
测试：

```java
public class MyArrayTest {
    public static void main(String[] args) {
        //创建自定义封装数组结构，数组大小为4
        MyArray array = new MyArray(4);
        //添加4个元素分别是1,2,3,4
        array.add(1);
        array.add(2);
        array.add(3);
        array.add(4);
        //显示数组元素
        array.display();
        //根据下标为0的元素
        int i = array.get(0);
        System.out.println(i);
        //删除4的元素
        array.delete(4);
        //将元素3修改为33
        array.modify(3, 33);
        array.display();
    }
 
}
```
结果为：
>1 2 3 4  
1  
1 2 33

## 三、分析数组的局限性

　通过上面的代码，我们发现数组是能完成一个数据结构所有的功能的，而且实现起来也不难，那数据既然能完成所有的工作，实际应用中为啥不用它来进行所有的数据存储呢？

数组的局限性分析：

1. **插入快**，对于无序数组，上面我们实现的数组就是无序的，即元素没有按照从大到小或者某个特定的顺序排列，只是按照插入的顺序排列。无序数组增加一个元素很简单，只需要在数组末尾添加元素即可，但是有序数组却不一定了，它需要在指定的位置插入。

2. **查找慢**，当然如果根据下标来查找是很快的。但是通常我们都是根据元素值来查找，给定一个元素值，对于无序数组，我们需要从数组第一个元素开始遍历，直到找到那个元素。有序数组通过特定的算法查找的速度会比无需数组快，后面我们会讲各种排序算法。

3. **删除慢**，根据元素值删除，我们要先找到该元素所处的位置，然后将元素后面的值整体向前面移动一个位置。也需要比较多的时间。

4. 数组一旦创建后，**大小就固定**了，不能动态扩展数组的元素个数。如果初始化你给一个很大的数组大小，那会白白浪费内存空间，如果给小了，后面数据个数增加了又添加不进去了。

很显然，数组虽然插入快，但是查找和删除都比较慢，而且扩展性差，所以我们一般不会用数组来存储数据，那有没有什么数据结构插入、查找、删除都很快，而且还能动态扩展存储个数大小呢，答案是有的，但是这是建立在很复杂的算法基础上，后面也会详细介绍。

## 四、总结
以上，介绍了数组的基本用法，以及用Java语言中的类实现了一个数组的数据结构，但是在分析该数据结构，发现存在很多性能问题，后面介绍别的数据结构时，看看那些数据结构是如何处理这些问题的。当然在讲解数据结构之前，会简单的介绍几种常用的排序算法。

> 参考资料   
https://www.cnblogs.com/ysocean/tag/Java%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%92%8C%E7%AE%97%E6%B3%95/