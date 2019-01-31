---
layout: post
category: java
comments: true
title: 说一说java.util.Arrays$ArrayList
tags: ArrayList
---
* content
{:toc}

`java.util.Arrays$ArrayList`(下文：`Arrays$ArrayList`)是`java.util.Arrays`的私有静态内部类，他实现的接口，继承的父类几乎和`java.util.ArrayList`（下文：`ArrayList`）相同，既然是私有的，那么平常应该是我们少关注的地方。本文尝试对比一两个他们之间的不同点。

## 使用场景对比
构造拥有三个字符串的`List`，
`ArrayList`
```java
List<String> lb = new ArrayList<>(3);
        lb.add("a");
        lb.add("b");
        lb.add("c");
```
`Arrays$ArrayList`
```java
List<String> la = Arrays.asList("a", "b", "c");
//源码
public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
```
两者都满足了需求，后者看起来比前者简洁，除此之外两者还有什么不同呢。

## 增加删除操作对比
支持的操作，
```java
lb.add("d")
lb.remove("d")
```
不支持的操作，这将会抛出异常`java.lang.UnsupportedOperationException`
```java
la.add("d")
la.remove("d")
```
可见`Arrays$ArrayList`不允许增加也不允许删除。




## 具体的add方法
在`Arrays$ArrayList`类，
首先并没有override父类的`add`方法,所以这个方法来自他的父类`AbstractList`；
看`AbstractList`中的`add`方法
```java
    public boolean add(E e) {
        add(size(), e);
        return true;
    }
    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }    
```
最终调用的方法抛出了异常`UnsupportedOperationException`。
相比较`ArrayList`
```java
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }    
```
这两个方法都在`ArrayList`中实现了，或者扩容然后在尾部插入，或者扩容、移动数组元素，然后插入到指定的下标位置。
## 具体的remove方法
`Arrays$ArrayList`的`remove(Object)`方法继承自`AbstractList`的父类`AbstractCollection`，其中
```java
    public boolean remove(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext()) {
                if (it.next()==null) {
                    it.remove();
                    return true;
                }
            }
        } else {
            while (it.hasNext()) {
                if (o.equals(it.next())) {
                    it.remove();
                    return true;
                }
            }
        }
        return false;
    }
```
这里使用了迭代器的方式进行删除，看这个方法的注释

![clipboard.png](/images/2900097773-5ac38376995e4_articlex.png)

如果这个迭代器没有实现`remove`方法的话，那么这整个方法也将要抛出`UnsupportedOperationException`异常的。
在`AbstractCollection`中`iterator`是一个抽象方法，之于`Arrays$ArrayList`，这个方法实现的位置还是在`AbstractList`，
```java
    public Iterator<E> iterator() {
        return new Itr();
    }
    private class Itr implements Iterator<E> {
        //...省略
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                AbstractList.this.remove(lastRet);
                if (lastRet < cursor)
                    cursor--;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException e) {
                throw new ConcurrentModificationException();
            }
        }    
    }
```
到这里我们发现`AbstractList`实现了`AbstractCollection`的`iterator`方法，而且返回的迭代器也实现了`remove`方法，不是上文提到的注释那种情况。但是为什么删除动作还是不允许的呢？
具体这个迭代器的`remove`方法，
```java
AbstractList.this.remove(lastRet);
```
可见迭代器最终也是调用容器类的`remove`方法的，那么`Arrays$ArrayList`没有实现`remove`方法，而`AbstractList`的`remove`方法，如下
```java
    public E remove(int index) {
        throw new UnsupportedOperationException();
    }
```
因此，即使在`AbstractList`中使用迭代器进行删除操作，但由于`Arrays$ArrayList`没有实现`remove`且继承的`remove`抛出`UnsupportedOperationException`异常，最终在`Arrays$ArrayList`是不允许删除操作的。

值得注意的是，`AbstractList`的迭代器是否要调用`remove(int)`方法是由要删除的目标是否数组的元素决定的，如果不存在这样的元素，则下面的代码并不会出现异常，
```java
List<String> la = Arrays.asList("a", "b", "c");
la.remove("e");
```