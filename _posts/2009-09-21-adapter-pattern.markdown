---
layout: post
title: " 设计模式学习笔记（3）——适配器模式(Adapter Pattern)"
date: 2009-08-28 16:48
comments: true
categories: "Design Pattern"
---
一、 目的：

将一个类的接口转换成客户希望的另一个接口。Adapter 模式使原本由于接口不兼容而不能一起工作的那些类可以一起工作。

——GoF 《Design Pattern 》

二、 描述：

其实个人觉得Adapter模式在所有的模式当中算是比较简单的那种，也比较容易理解，就像我们日常生活中经常会使用USB转换器，将某种非USB接口的外设转换我们本本上提供的USB接口一样，在面向对象设计中，有时候我们不得不将将某个类的接口转换成另外一种我们需要的类型，最通常的情况可能是因为我们要想为现有的系统添加额外的一个现有的子系统，而两个系统提供的接口不同，因此，我们不得不讲其中一个转换成另一个，或者进行双向转换。

说起来好像很容易理解，那么到底应该如何应用Adapter模式呢，还是举HeadFirst中的例子吧：

假设现在我们有一个现有系统SimuDuck，提供一个鸭子的接口Duck，要求系统中的所有的类必须实现该Duck接口

![](/images/_clip_image002.gif)

（注：staruml中设置接口图形格式：右键图标->Format->seterotype->None,不过怎么把方法显示出来还么研究好）

该接口提供两个方法，fly(),飞， quack(),叫。

现在要向这个系统中天加一个由一些火鸡组成的子系统，火鸡实现的是火鸡接口Turkey：

![](/images/o_clip_image004_2_633891499875937500.gif)

该接口提供的方法功能包括,fly()飞，gobble()叫。

现在要想将实现Turkey的具体火鸡类在现有的使用Duck接口的系统中使用，就必须将Turkey转换成Duck，实现起来很简单，类图如下：

![](/images/o_adapter_duck_turkey_4.jpg)

设计一个TurkeyAdapter，该适配器实现了Duck接口，并使用一个Turkey作为构造参数，在TurkeyAdapter中实现Duck接口的方法时，实际调用的是Turkey的方法，下面时代码实现：

Duck接口：

[java] view plaincopy
public interface Duck {  
    public abstract void quack();  
    public abstract void fly();  
}  
MallardDuck具体鸭子类：

[java] view plaincopy
public class MallardDuck implements Duck {  
    public void quack(){  
        System.out.println("MallardDuck quack");  
    }  
    public void fly(){  
        System.out.println("MallardDuck fly");  
    }  
}  
注意，这里的实现quack()和fly()方法时使用简单的直接实现，实际中应该采用Strategey（策略）模式，以后再说该模式。

Turkey接口：

[java] view plaincopy
public interface Turkey {  
    public void fly();  
    public void gobble();  
}  
实现Turkey的具体WildTurkey：

[c-sharp] view plaincopy
public class WildTurkey implements Turkey {  
    public void fly(){  
        System.out.println("WildTurkey fly");  
    }  
    public void gobble(){  
        System.out.println("WildTurkey gobble");  
    }  
}  
最关键的TurkeyAdapter：

[java] view plaincopy
public class TurkeyAdapter implements Duck {  
    Turkey turkey;  
    public TurkeyAdapter(Turkey turkey){  
        this.turkey = turkey;  
    }  
    public void quack(){  
        turkey.gobble();  
    }  
    public void fly(){  
        turkey.fly();  
    }  
}  
该类拥有一个Turkey的实例变量，一个以Turkey作为参数构造方法，也就是说在构造时将要转换的Turkey对象作为构造参数传递给该适配器，当调用适配器的Duck接口的方法时，实际调用的是turkey对象的方法。

在这里，由于示例比较简单，没有什么问题，在实践中，可能会出现被转换的类不具备目标对象的某种功能，例如，如果Duck中有一个swim()方法，很明显，Turkey不能游泳，也就没有可以调用的方法，通常的做法是抛出一个异常CanNotFindMethodException。

最后是测试程序：

[java] view plaincopy
public class Client {  
    public static void main(String[] args){  
        MallardDuck duck = new MallardDuck();  
          
        WildTurkey wildTurkey = new WildTurkey();  
        Duck turkeyAdapter = new TurkeyAdapter(wildTurkey);  
          
        System.out.println("Duck...");  
        testDuck(duck);  
          
        System.out.println("TurkeyAdapter...");  
        testDuck(turkeyAdapter);  
    }  
    static void testDuck(Duck duck){  
        duck.fly();  
        duck.quack();  
    }  
}  
该测试程序的输出结果为：

[java] view plaincopy
Duck...  
MallardDuck fly  
MallardDuck quack  
TurkeyAdapter...  
WildTurkey fly  
WildTurkey gobble  