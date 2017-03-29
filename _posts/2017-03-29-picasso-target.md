---
layout: post
title: Picasso Target第一次不调用onBitmapLoaded问题
categories: Picasso
description: 修复Picasso Target问题
keywords: Picasso
---

## 需求

最近项目中要做一个类似朋友圈发表图片的功能，做完之后发现点击图片查看高清图片的时候第一次不调用onBitmapLoaded。然后多试了几次发现只有一张图片的时候就没有这个问题，很是怪异。初步感觉应该是ViewPager预加载导致的问题，当然，大家都知道ViewPager想不设置预加载也是个复杂的工程，因为ViewPager源码里把DEFAULT_OFFSCREEN_PAGES默认为了1，所以你怎么设setOffscreenPageLimit都解决不了的,也许copy一份能解决。扯远了...那怎么办呢，漫长的Debug之路。一步步跟下去发现
```java
Target target = getTarget();
    if (target != null) {
      target.onBitmapLoaded(result, from);
      if (result.isRecycled()) {
        throw new IllegalStateException("Target callback must not recycle bitmap!");
      }
    }
```
result也有，就要走onBitmapLoaded这一步了，但是...getTarget()为空。那就看看getTarget()是个什么鬼吧
```java
T getTarget() {
    return target == null ? null : target.get();
  }
```
object.get()是不是很熟悉，是的，就是WeakReference
```java
 this.target =
        target == null ? null : new RequestWeakReference<T>(this, target, picasso.referenceQueue);
```
该死的原来是被回收了。知道被回收了就好办了，让它持有一个强引用对象啊，使用imageView.setTag(target)完美解决		

## 实现

**关键：**来个强引用抱一下，先不让回收。

