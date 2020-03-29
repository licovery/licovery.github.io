---
title: C++线程安全的单例模式
date: 2019-10-23 15:31:55
tags: 
- 多线程
- 多进程
- 设计模式
- 单例
categories: 编程
---

## 线程安全

多线程并行执行的程序中，对共享数据的访问或修改可能由于**执行顺序的不确定**，导致运行结果与代码设计者的原意相违背。为了保证代码原有的逻辑，在多线程的情况下可能需要用到一些同步或互斥的操作来保证线程安全。人的思考方式是单线程的，所以有时候看着很合理的代码逻辑，在多线程环境却出bug。

<!-- more -->

## 单例模式

最简单的设计模式之一，它的含义是这个类只能有**唯一的对象实例**。一般的单例对外的接口如下

```c++
//.h

class SingleInstance
{
public:
    static SingleInstance * getInstance();
private:
    static SingleInstance *pInstance;
    SingleInstance();//private禁止外部构造对象
};
```



单例模式按**实例创建的时机**不同，可以分为**懒汉式**和**饿汉式**：

- **懒汉式**：实例并不会提前创建好，只有当你**第一次**去访问的时候才会创建出唯一的实例，后面的访问会直接返回第一次创建的实例。
- **饿汉式**：**提前**创建实例，无论什么时候访问，都只能返回提前创建好的实例。

### 普通懒汉式单例实现

懒汉式的实现，先把实例指针初始化为`nullptr`，在`getInstance()`的调用过程中去检查实例是否已经被创建，如没有则创建，若已经创建好了，则直接返回。

```c++
//.cpp

SingleInstance * SingleInstance::pInstance = nullptr;

SingleInstance * SingleInstance::getInstance()
{
    if (pInstance == nullptr)//考虑多个线程都运行到这一行代码，会发生什么影响
    {
        pInstance = new SingleInstance;
    }
    return pInstance;
}

SingleInstance::SingleInstance()
{
}
```

### 饿汉式单例实现

饿汉式的实现，在定义静态数据成员的时候就new一个实例出来，这发生在所有代码运行前。

```c++
//.cpp

SingleInstance * SingleInstance::pInstance = new SingleInstance;

SingleInstance * SingleInstance::getInstance()
{
    return pInstance;
}

SingleInstance::SingleInstance()
{
}
```

## 线程安全的单例

说了那么多，那单例和线程安全有什么关系，怎么样实现的单例线程不安全呢？

### 饿汉式本身线程安全

对于**饿汉式**单例，本身就是线程安全的，因为实例的创建发生在所有代码运行前。

### 加锁的懒汉式单例

对于**普通懒汉式**的单例，在判断指针是否为空这里，如果是在多线程的情景下，可能会出现多个线程判断`if (pInstance == nullptr)`均成立，然后去创建了多个实例，这就违反了单一实例的原则，同时也会内存泄露。所以在这个地方，必须加锁用于互斥。

先简单说一下怎么加锁，C++11以后提供了多线程库的支持，使用`mutex`和`unique_lock`就可以很简单地实现加锁。

```c++
//.cpp

#include <mutex>

std::mutex m;//全局变量

void fun()
{
    std::unique_lock<std::mutex> lck(m);//从这里开始加锁,使用mutex初始化会自动加锁
    proc;
}//在lck离开作用域析构的时候会自动解锁，也可以显式地使用lck.lock或者lck.unlock来进行操作
```

那么我们来给懒汉加个锁

```c++
/.cpp

#include <mutex>

std::mutex m;//这里是全局锁，也可以把这个锁放在SingleInstance里面作为静态成员变量

SingleInstance * SingleInstance::pInstance = nullptr;

SingleInstance * SingleInstance::getInstance()
{
    if (pInstance == nullptr)//double check
    {
        std::unique_lock<std::mutex> lck(m)；
        if (pInstance == nullptr)
        {
            pInstance = new SingleInstance;
        }
        //离开lck作用域析构解锁
    }
    return pInstance;
}

SingleInstance::SingleInstance()
{
}

```

咦，有人会奇怪，为什么要判断两次`if (pInstance == nullptr)`。当初我也很疑惑，去掉最外层的判断逻辑是否正确呢，答案是正确的。这个称为**double check**，是为了提升性能。首先加锁是会对系统造成较大的性能消耗的。如果没有最外层的判断，那么每次`getInstance()`都要构造锁加锁，然后再析构锁解锁，性能影响是很大，而且没有必要。只有`pInstance==nullptr`才有申请实例的可能，才有出现线程安全问题的可能，才有加锁的必要。所以说这个**double check**就是一个套路啦，在其他用到锁的场合应该都能应用上。

**这样就完美了吗？**

`pInstance = new SingleInstance;`这行代码在执行时会进行许多工作，内存申请，对象构造，指针赋值。

```c++
pInstance = new SingleInstance;
//相当于
memory = allocate();//1内存申请
construct(memory);  //2对象构造
pInstance = memory;	//3指针赋值
```

在编译神秘的优化下，可能发生**指令的重排**，例如执行顺序为1,3,2。对单线程来说没有很大的影响，但是在多线程环境下，在3执行完，把`pInstance`置为非空后，刚好其他线程抢占了CPU，那么这个线程获取实例时就会获取到了一个没有构造好的对象，导致逻辑出错。一般的解决方法是加一个**局部变量做缓冲**。我看到有网上的资料提过，还要加**内存屏障**。 因为CPU有一级二级缓存，CPU的计算结果并不是及时更新到内存的,所以在多线程环境，不同线程间共享内存数据存在可见性问题。

```c++
//.cpp

SingleInstance * SingleInstance::getInstance()
{
    if (pInstance == nullptr)
    {
        std::unique_lock<std::mutex> lck(m)；
        if (pInstance == nullptr)
        {
            auto temp = new SingleInstance;//先用局部变量记录起来
            memory_barrier();//保证前后代码执行的顺序性
            pInstance = temp;
        }
        //离开lck作用域析构解锁
    }
    return pInstance;
}
```

这样写的话应该是最安全，但逻辑也是最复杂，性能也比较差。除了给懒汉加锁，还有没有其他的方式呢？答案是有的。

### 局部静态变量的懒汉单例（C++11支持）

不再把实例作为类的静态成员，而是把实例放到`getInstance()`函数中，作为其中的**静态局部变量**。

```c++
//.cpp

class SingleInstance
{
public:
    static SingleInstance & getInstance();
private:
    SingleInstance();
};


SingleInstance & SingleInstance::getInstance()
{
    static SingleInstance instance;
    return instance;
}

SingleInstance::SingleInstance()
{
}
```

`g++`编译的时候注意加上 `-std=c++11`的参数。C++11 保证静态局部变量的初始化过程是线程安全的。 这种方式既简洁又高效，首推这个。要注意的是之前`getInstance()`返回指针，现在返回的是**引用**，如果不是引用，返回过程就有拷贝，就会出现多实例，违背单例原则。

## 总结

单例实现分为懒汉式和饿汉式。

懒汉式以时间换空间， 适应于访问量较**小**时 ，如果要使用首推局部静态变量的懒汉单例。

饿汉式以空间换时间 ，适应于访问量较**大**时 。

## 感悟

一个简单的单例模式涉及到多线程的问题就会变得如此复杂。很多看似简单的问题背后，都会有很多深奥的知识。探索的过程固然很累，但是如果学习到新的知识，还是会感到快乐的。这就是程序员学习的动力吧！

## 参考

本文参考网络上已有资料，自己整合做的总结。

https://juejin.im/post/5d692773f265da03986c0832#heading-13

https://blog.csdn.net/ll641058431/article/details/50056597

 https://blog.csdn.net/qq_36922927/article/details/84977365#3_double_check_42 