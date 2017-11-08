---
title: HeadFirst设计模式笔记2单例模式和spring单例
date: 2017-08-11 00:23:50
tags:
categories: 学
---

像线程池，缓存，注册表对象，日志对象，设备驱动程序对象等，如果存在多个实例，就很容易出问题：行为异常，资源使用过量，产生不一致结果
单例模式就是保证对象只存在一个实例
静态变量static也可以做到，但是因为static在类加载的时候就创建分配内存了，还没跑对象涉及的代码就已经在耗费系统资源

首先来个最简单的单例思路

<!--more-->
```java
public class Single(){
    private static Single singleInstance;
    private Single(){}
    public Single getInstance(){
         if(singleInstance == null){
                singleInstance = new Single();
            }
         return singleInstance
     }
}
```
就是用静态变量记录Single的唯一实例，然后把构造器声明为私有，只能Single内部自己调用，然后公共的获取实例的方法给外部用

__________________________
但是如果一开始没有实例，然后两个线程同时进去getInstace就会拿到两个不一样实例，所以这段代码是线程不安全的
简单的给getInstace方法加synchronized关键字可以解决问题
但是如果已经有实例了，那以后每次进去都是不需要同步的，所以也就第一次需要同步，那这个synchronized就很费事了
最正确的方式是双重检查，就是先看有没实例，有就不需要同步直接返回该实例，没有就同步一下
```java
public class Single {
	private volatile static Single singleInstance;
	private Single() {}
	public static Single getInstance() {
		if (singleInstance == null) {
			synchronized (Single.class) {
				if (singleInstance == null) {
					singleInstance = new Single();
				}
			}
		}
		return singleInstance;
	}
}
```
volatile关键字理解参考这里：
http://blog.csdn.net/guyuealian/article/details/52525724

注意：
使用多个类加载器时，单例失效，
_____________________________________
都说spring也是单例模式，于是再扩展下
最开头那种叫饿汉单例（不管有没有new一个用了再说），最后那种是懒汉单例（有就不new了）
上面两种都是不能被继承的
因为子类中的构造函数要调用父类的默认构造函数super()来完成自己的构造函数，而这里的父类构造函数私有化了子类用不了
于是有第三种：单例注册表
```java
import java.util.HashMap;

public class RegSingle {
	private static HashMap registry = new HashMap();
	// 静态块，在类被加载时自动执行
	static {
		RegSingle rs = new RegSingle();
		registry.put(rs.getClass().getName(), rs);
	}
	// 受保护的默认构造函数，子类继承则可以调用
	protected RegSingle() {}
	public static RegSingle getInstance(String name) {
		if (name == null) {
			name = "RegSingle";
		}
		if (registry.get(name) == null) {
			try {
				registry.put(name, Class.forName(name).newInstance());
			} catch (Exception ex) {
				ex.printStackTrace();
			}
		}
		return (RegSingle)registry.get(name);
	}
}
```
所以说springIOC也就是这么管理bean的hashMap，然后再注入进去的
更具体点的实现自己写一个IOC百度下也很多，不赘述
还有挺重要的spring怎么用ThreadLocal解决同步问题，什么时候过一遍



