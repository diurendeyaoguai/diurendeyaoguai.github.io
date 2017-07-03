---
layout: post
title: Kotlin初探，空安全
categories: Kotlin
description: Kotlin空安全
keywords: Kotlin
---

## 一，var val区别 

var 变量  val 常量（java中的final）

## 二，空安全的问题
在 Kotlin 中，类型系统区分一个引用可以容纳 null （可空引用）还是不能容纳（非空引用）。 例如，String 类型的常规变量不能容纳 null：
```java
var a:String = "a"     a = null //会编译不通过
```
如果需要允许为空，需要在类型后面加? 应该这样写
```java
var b:String? = "b"  b = null//这样就不会报错了
```
如果调用
```java
 val length = a.length 
```
一点问题都没有，因为a绝对不会为null
但是如果调用b.length编译会不通过，因为b可能为空。但是我们还是会必须访问b的变量的吧，在实际开发中
那么，将有以下几种方法做到
**1，手动判断**
```java
val length = if(b!=null) b.length else -1;
```
或者
```java
if (b != null && b.length > 0) {
    print("String of length ${b.length}")
} else {
    print("Empty string")
}
```
请注意，这只适用于 b 是不可变的情况（即在检查和使用之间没有修改过的局部变量 ，或者不可覆盖并且有幕后字段的 val 成员），因为否则可能会发生在检查之后 b 又变为 null 的情况。

**2，操作符?.**   

b?.length 如果b为null那直接返回null，如果b不为null则返回length
kotlin文档里有这么个案例特别有意思：
安全调用在链式调用中很有用。例如，如果一个员工 Bob 可能会（或者不会）分配给一个部门， 并且可能有另外一个员工是该部门的负责人，那么获取 Bob 所在部门负责人（如果有的话）的名字，我们写作：
bob?.department?.head?.name
如果任意一个属性（环节）为空，这个链式调用就会返回 null。

如果要只对非空值执行某个操作，安全调用操作符可以与 let 一起使用：
```java
var b:String? = "b"
b?.let{ print(b)}
```
**操作符Elvis（猫王？哈哈）    ?:**

可以说kotlin的操作符一上来就给我干懵了
```java
val l = b?.length?:-1 //这句话的意思跟val l = if(b!=null) b.length else -1一个意思
```
?:意思其实是如果前面的表达式为null则结果为右侧的值，否则返回表达式的值

**操作符!!**

没错这个操作符是两个!
```java
val l = b!!.length  
```    
b!!如果b为null那么直接抛出nullpointexception，如果b不为null那么直接返回b.length（这tm不是又回到java了吗）

可空类型的集合

如果你有一个可空类型元素的集合，并且想要过滤非空元素，你可以使用 filterNotNull 来实现。
```java
val nullableList: List<Int?> = listOf(1, 2, null, 4)
val intList: List<Int> = nullableList.filterNotNull()
```
 