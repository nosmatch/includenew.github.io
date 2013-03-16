---
layout: post
title: "设计模式学习笔记（2）——装饰模式(Decorator Pattern)"
date: 2009-09-20 16:48
comments: true
categories: "Design Pattern"
---

###一．目的

装饰模式的目的是动态的为对象添加一些额外职责。就增加功能来说， Decorator 模式比继承产生的子类给为灵活。 ——GoF《Design Pattern》。

###二．描述

举HeadFirst中Starbucks的例子，星巴克中有很多种饮料，每种饮料需要有单独的计算价格cost()的功能，以及能够获得该饮料的具体描述的getDescription()功能。如果通过设置一个Beverage超类，

![Beverage](/images/o_clip_image002_2.gif)

所有其它的具体饮料都继承该超类，就会产生类爆炸现象（可以考虑有4大类饮料，HouseBlend，DarkRotast，Decaf，Espresso，4种配料milk，soy，mocha，whip，配料可以自由组合，自己算算一共有多少种具体的饮料类型吧），显然这是不行的（考虑一下某种配料的价格改变了，那么需要改变的类的数目！！）。

一种改进方法是将配料价格的计算放在超类Beverage中，

![](/images/o_clip_image004_2.gif)


这样就不用担心某种配料价格的改变了（可以通过超类来设置配料价格），但是这种设置方式还是有缺陷的，可以想象一下如果创造出一种新的配料（毕竟创新时代嘛），那么就不得不改变超类Beverage，在其中添加新配料的方法，改变现有的代码是OO里极其避讳的事情，因此需要更好的方法来处理类似的问题，于是，就有了Decorator Pattern。

Decorator Pattern的目的就是要动态的想对象中添加功能，针对此例来说，我们可以将配料当作额外的功能，饮料大类当作目标对象，用配料来装饰目标对象，例如，想要一个加鲜奶（whip）摩卡（mocha）的DarkRoast，那么，首先选择目标对象，DarkRoast，然后用添加mocha，（用mocha装饰），形成一个加mocha的DarkRoast，然后添加whip（用whip装饰），产生最终的饮料，但是在这里有一点很重要，如何计算价钱，毕竟人家要赚钱么~设计中很容易，因为所有的目标对象类，以及配料类都是继承自超类Beverage，（不过注意这里，继承并不是为了获取父类功能，只是为了类型的统一），均有一个cost()方法，调用最后包装好的目标对象的cost()方法得到的就是全部的价钱。好像有点乱哈，先看看类图吧：

![](/images/o_clip_image006_2.gif)


每个目标对象类（Espresso等）都继承自超类Beverage，实现超类提供一个抽象方法cost()，配料装饰类继承自一个抽象类CondimentDecorator类，该抽相类同样继承自超类Beverage，并且覆盖了超类中的getDescription()方法，使其成为抽象方法，强制具体的配料类必须实现该方法。并且每个具体的配料类中都拥有一个Beverage实例对象，作为其要包装的目标对象。

###三．代码实现：

Beverage 类 ：
	
	public abstract class Beverage {  
	    String description="Unknown Beverage";  
	    public abstract double cost();  
	    public String getDescription() {  
	        return description;  
	    }  
	}  

具体目标对象类：

DarkRoast类：

	
	public class DarkRoast extends Beverage {  
	    public DarkRoast(){  
	        super.description = "DarkRoast";  
	    }  
	    public double cost() {  
	        return 0.89;//事实上应该提供一个price属性，然后提供该属性的setter，getter方法  
	    }  
	}  

这个代码实例是按照HeadFirst上实现的，事实上，个人觉得在实际应用中应该提供一个price属性，并为该属性提供setter和getter方法，用于设置该饮料的价格，而不是直接将价格硬编码到类中。

这个累的实现很简单，就是在构造时设置description为对应的饮料描述。其余的目标类与这个类的实现类似，只不过对应的描述，价格不同而已。

抽象配料装饰类：

CondimentDecorator类：

	
	public abstract class CondimentDecorator extends Beverage {  
	    public abstract String getDescription();  
	}  

该抽象类强制具体实现的配料类要提供getDescription方法，这也是HeadFirst中的实现，但是我个人觉得，应该把子类中的beverage也放到该类中，毕竟是共性的东西。

具体配料类的实现：

Mocha类：

	
	public class Mocha extends CondimentDecorator {  
	    private Beverage beverage;  
	      
	    public Mocha(Beverage beverage) {  
	        this.beverage = beverage;  
	    }  
	    public double cost() {  
	        return beverage.cost()+0.25;  
	    }  
	    @Override  
	    public String getDescription() {  
	        // TODO Auto-generated method stub  
	        return beverage.getDescription()+", Mocha";  
	    }  
	}  

具体配料类的构造方法中有一个参数，也就是目标对象，通过构造方法为属性赋值，计算价格时调用目标对象的cost()方法，然后加上配料的价格，实现抽象配料类中的getDescription()方法，调用目标对象的getDescription()，并添加相应的配料描述。同样在这里，我觉得应该为配料提供一个价格属性，price，而不是直接将价格硬编码到类中。其余的配料实现类与该类的实现类似。

测试类：

	
	public class StarbuzzTest {  
	    public static void main(String[] args){  
	        Beverage beverage = new Espresso();  
	        System.out.println(beverage.getDescription()+" $"+beverage.cost());  
	          
	        Beverage beverage2 = new DarkRoast();  
	        beverage2 = new Mocha(beverage2);  
	        beverage2 = new Whip(beverage2);  
	        System.out.println(beverage2.getDescription()+" $"+beverage2.cost());  
	          
	        Beverage beverage3 = new HouseBlend();  
	        beverage3 = new Soy(beverage3);  
	        beverage3 = new Mocha(beverage3);  
	        beverage3 = new Mocha(beverage3);  
	        System.out.println(beverage3.getDescription()+" $"+beverage3.cost());  
	    }  
	}  
	























测试类中用于产生目标饮料类型对象的语句，可以采用工厂方法，或者抽象工厂等模式来实现，这里只是为了简化，尊重HeadFirst原书实现。

测试结果：

	
	Espresso $1.05  
	DarkRoast, Mocha, Whip $1.39  
	HouseBlend, Soy, Mocha, Mocha $2.69  





###四．总结

装饰模式的作用就是为目标对象动态添加职责。它提供了一种比继承更加自由的扩展功能的实现。

当然，装饰模式中也用到了继承，但是这里的继承只是为了实现类型的统一，因为需要要使目标对象与装饰对象具有相同的超类。而不是想通常所用继承为了扩展超类的功能。

该模式的参与者包括：

Component：定义一个对象接口，可以给这些对象动态添加职责。

ConcreteComponent：定义一个对象，可以个这个对象添加职责。

Decorator：拥有一个目标对象Component的引用，并且该Decorator实现了目标Component接口。

ConcreteDecorator：负责向目标对象添加职责的具体Decorator实现。

通用的装饰模式类图：

![](/images/o_clip_image008_2.gif)

装饰模式的优点：

1. 比静态集成更灵活。

2. 避免在层次结构高层的类有太多的特征。

——GoF 《Design Pattern 》

适用性：

1. 在不影响其他对象的情况下，以动态的，透明的方式给单个对象添加职责。

2. 处理可撤销的职责。

3. 当不能采用生成子类的方法进行扩充时。一种情况是，可能有大量独立的扩展，为支持每一种组合，将产生大量子类（例如上面的类爆炸），使得子类呈现爆炸式增长。另一种就是类定义被隐藏，也就是说不允许生成子类（java中的final类）。