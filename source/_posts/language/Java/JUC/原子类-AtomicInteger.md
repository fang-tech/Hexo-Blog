---
title: JUC-原子类-AtomicInteger类详解
tag: ["Java", "JUC", "原子类"]
categories: ["Java", "JUC", "原子类"]
---

# 原子类-AtomicInteger类详解

说实话, 也没什么好额外讲的, 整个原子类家族, 就是一族封装了一些类CAS操作的类, 提供了我们为某个变量进行原子操作的能力, 让我们可以在多线程环境下, 对某个变量进行原子操作, 而不需要加锁.

下面简单说明一下类的结构

## 核心字段

```java
    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long VALUE
        = U.objectFieldOffset(AtomicInteger.class, "value");

    private volatile int value;
```

- Unsafe就是提供了操作系统层面的API来执行CAS操作类
- VALUE是使用了Unsafe获取到的AtomicInteger类中的value字段的属性在类中偏移量, 用于CAS操作
- value就是我们这个Integer类的值, **这里的value是保证可见性的, 所以如果使用Atomic是默认保证多线程之间的可见性, 不需要额外加volatile**


## 核心方法

因为比较简单, 大多数方法就是让CAS操作更易用, 所以我只列举出来核心方法, 不提供源码了

- get() : 获取原子类的值
- set(int newValue) : 设置原子类的值

核心方法一般分成getAndSet和setAndGet两种, 前者返回的是set前的值, 后者返回的是set后的值

### getAndSet类

`getAndSet(int newValue)` : 获取原子类的值, 并设置为newValue

`getAndIncrement()` : 获取原子类的值, 并将其加1

`getAndDecrement()` : 获取原子类的值, 并将其减1

`getAndAdd(int delta)` : 获取原子类的值, 并将其加上delta

### setAndGet类

`setAndGet(int newValue)` : 设置原子类的值为newValue, 并返回newValue

`incrementAndGet()` : 将原子类的值加1, 并返回加1后的值

`decrementAndGet()` : 将原子类的值减1, 并返回减1后的值

`addAndGet(int delta)` : 将原子类的值加上delta, 并返回加上delta后的值

### Compare And Exchange类 (JDK9)

正宗的CAS操作


```java
    public final int compareAndExchangeAcquire(int expectedValue, int newValue) {
        return U.compareAndExchangeIntAcquire(this, VALUE, expectedValue, newValue);
    }
```
> 这个CAS是有内存屏障保障不会被重排序的, 任何内存操作不会跨越该操作进行重排序


```java
    public final int compareAndExchangeAcquire(int expectedValue, int newValue) {
        return U.compareAndExchangeIntAcquire(this, VALUE, expectedValue, newValue);
    }
```

> 这个CAS只保证该操作之后的读写操作不会被重排序到该操作之前, 但该操作之前的读写操作可以被重排序到该操作之后


