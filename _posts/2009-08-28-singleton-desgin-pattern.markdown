---
layout: post
title: "设计模式学习笔记(1)——单例模式"
date: 2009-08-28 16:48
comments: true
categories: Design Pattern
---
####目的：
保证一个类仅有一个实例，并为该实例提供也个安全的全局访问点

####用途：
在很多时候，系统中需要某个类只有一个实例，例如连接数据库时的驱动程序注册，加载属性文件等。

实现的核心是要有一个私有的构造方法，静态实例变量，以及返回该静态变量的公有静态方法作为全局访问接口。

一个最基本的实现：

	public class Singleton { 
	    private static Singleton instance = null; // 静态实例变量
	
	    private Singleton() { // 私有的构造方法 
	    }
	
	    public static Singleton getInstance() { // 公有的全局访问接口 
	       if (instance == null) { // 检查是否已经实例化，以保证只有一个实例 
	            instance = new Singleton(); 
	        } 
	        return instance; 
	    } 
	}

这种实现在单线程环境下是没有什么问题的，但是在多线程环境下就会出现问题，例如现在有两个线程A，B，当A执行完红色的判断语句后切换到线程B，假设此时instance尚未实例化，B执行判断，条件为真，进行实例化instance，返回，当再次切换到线程A时，继续上一次的执行，实例化instance，这是系统中就存在了两个实例。

最简单的改进方法就是在对getInstance方法进行同步限制，也就是

	public static synchronized Singleton getInstance() {…}

但是这样synchronized的效率很差，存在更好的改进方法

####1.直接加载

通常使用的都是晚加载（Lazy mode），需要时在实例化，但是，有时能够确定系统中肯定要存在一个该类的实例，直接加载是一种简单的实现方式

	public class SingletonEagerInstantiation { 
	    private static SingletonEagerInstantiation instance = new SingletonEagerInstantiation(); //直接静态实例化
	
	    private SingletonEagerInstantiation() { 
	    }
	
	    public static SingletonEagerInstantiation getInstance() { 
	        return instance;//直接返回 
	    } 
	}

####2.双重检查

刚才使用synchronized关键字时，出现了效率差的现象，是因为无论instance是否已经实例化了都要进行同步，synchronized效率很差，而使用双重检查，先判断instance是否已经实例化，如果为空在进行同步，这样就提高了效率，而且这种实现方式也提供了晚加载的能力，但是注意volatile关键字限制了该种实现方式只能用于jdk1.5以上版本

	public class SingletonDoubleCheck { 
	    private static volatile SingletonDoubleCheck instance = null; 
	    private SingletonDoubleCheck() { 
	    } 
	    public static SingletonDoubleCheck getInstance() { 
	        if (instance == null) {//先判断 
	            synchronized (SingletonDoubleCheck.class) {//如果没有实例化，同步 
	                if (instance == null) {//再次判断 
	                    instance = new SingletonDoubleCheck();//实例化 
	                } 
	            } 
	        } 
	        return instance; 
	    } 
	}
