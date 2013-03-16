---
layout: post
title: "Jetty ClassLoader解析"
date: 2012-02-14 00:26
comments: true
categories: JVM
---
##什么是类加载器？
类加载器（ClassLoader）指将类加载到虚拟机中的代码模块，所有的类必须通过加载器被加载到JVM中。JVM规范将累加在的过程外置于JVM实现，让应用程序自己决定如何获取所需类，这种机制位类层次划分，热加载，模块化奠定了基础。
##类加载器分类
类加载器主要分为：

* 启动类加载器(Bootstrap ClassLoader)：主要负责加载<JAVA_HOME>\lib目录中或者-Xbootclasspath中指定的，并且被虚拟机识别的类库加载到VM中。这个加载器是JVM自身的一部分，用本地代码实现的(openjdk中源码位于hotspot/src/share/vm/classfile/classLoader.cpp中)，无法直接被java代码引用。
* 扩展类加载器(Extension ClassLoader)：主要负责加载jdk扩展类库<JAVA_HOME>\lib\ext或者java.ext.dirs系统属性指定的目录中jar文件，由sun.misc.Launcher$ExtClassLoader实现
* 系统类加载器(System ClassLoader)：用于加载CLASSPATH中指定的类，由sun.misc.Launcher$AppClassLoader实现。该类即ClassLoader.getSystemLoader()的返回值，是应用程序默认的类加载器。
* 自定义加载器(User ClassLoader)：用户可以自定义自己的类加载器

##双亲委托模型
JDK中要求所有的自定义ClassLoader 必须扩展自抽象类java.lang.ClassLoader。该类的文档说名中有一个段关于delegate model的描述：

>The ClassLoader class uses a `delegation model` to search for classes and resources. Each instance of ClassLoader has an associated parent class loader. When requested to find a class or resource, a ClassLoader instance will delegate the search for the class or resource to its parent class loader before attempting to find the class or resource itself. The virtual machine's built-in class loader, called the "bootstrap class loader", does not itself have a parent but may serve as the parent of a ClassLoader instance.

简单来说这个delegate model要求除了Bootstrap ClassLoader之外，其余ClassLoader都需要关联一个parent ClassLoader（这种关联方式采用的时组合而非继承），在执行加载class时，首先委托给parent ClassLoader加载，只有当parent ClassLoader无法加载时，再由自身加载。各加载器的关联关系如下：

			Bootstrap ClassLoader
					 |
			Extension ClassLoader
					 |
			System ClassLoader
				/ 			\
		User1 ClassLoader	User2 ClassLoader
Bootstrap ClassLoader是最根层的加载器，用户自定义加载器建议使用系统类加载器作为parent。
这种委托模型的好处显而易见，它维护了类加载器之间的层次优先级关系。使所有的类加载优先由parent加载，这保证了java基础类库中的加载只会有一份。以java.lang.Object为例，委托模式保证了这个类最终只会由BootstrapClassLoader来加载，以此来保证所有环境中只有同一个类。否则由各加载器自由发挥，当用户自己定义各同名的java.lang.Object类时，系统会出现多分Objec类，最根基的行为出现混乱。（当然，你还是可以自定义出一个同名的java.lang.Object，并且顺利通过编译，但是它正常情况下永远不会被加载）。

***这个委托模型并非jvm的强制规范，只是jdk中建议的一种模式，有时会发现不遵守这种模式的行为却能产生奇妙的效果，如热部署，OSGI等。***



##Jetty中的ClassLoader
jetty，tomcat等web容器通常都会对classloader做扩展，因为一个正常的容器至少要保证其内部运行的多个webapp之间：私有的类库不受影响，并且公有的类库可以共享。这正好发挥classloader的层级划分优势。
jetty中有一个org.mortbay.jetty.webapp.WebAppClassLoader，负责加载一个webapp context中的应用类，WebAppClassLoader以系统类加载器作为parent，用于加载系统类。不过servlet规范使得web容器的classloader比正常的classloader委托模型稍稍复杂，servlet规范要求：

1. WEB-INF/lib 和 WEB-INF/classes优先于父容器中的类加载，比如WEB-INF/classes下有个XYZ类，CLASSPATH下也有个XYZ类，jetty中优先加载的是WEB-INF/classes下的，这与正常的父加载器优先相反。
2. 系统类比如java.lang.String不遵循第一条， WEB-INF/classes或WEB-INF/lib下的类不能替换系统类。不过规范中没有明确规定哪些是系统类，jetty中的实现是按照类的全路径名判断。
3. Server的实现类不被应用中的类引用，即Server的实现类不能被人和应用类加载器加载。不过，同样的，规范里没有明确规定哪些是Server的实现类，jetty中同样是按照类的全路径名判断。

为了处理上述三个问题，jetty的应用类加载器(org.mortbay.jetty.webapp.WebAppClassLoader)做了些特殊处理。
###WebAppClassLoader的实现
首先看WebAppClassLoader的实现，WebAppClassLoader的构造器中有如下代码：
	
	super(new URL[]{},parent!=null?parent
                :(Thread.currentThread().getContextClassLoader()!=null?Thread.currentThread().getContextClassLoader()
                        :(WebAppClassLoader.class.getClassLoader()!=null?WebAppClassLoader.class.getClassLoader()
                                :ClassLoader.getSystemClassLoader())));

表明WebAppClassLoader还是按照正常的范式设置parent classloader
然后看重要的loadclass方法实现：

    @Override
    protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        Class<?> c= findLoadedClass(name);
        ClassNotFoundException ex= null;
        boolean tried_parent= false;
        
        boolean system_class=_context.isSystemClass(name);
        boolean server_class=_context.isServerClass(name);
        
        if (system_class && server_class)
        {
            return null;
        }
        
        if (c == null && _parent!=null && (_context.isParentLoaderPriority() || system_class) && !server_class)
        {
            tried_parent= true;
            try
            {
                c= _parent.loadClass(name);
                if (LOG.isDebugEnabled())
                    LOG.debug("loaded " + c);
            }
            catch (ClassNotFoundException e)
            {
                ex= e;
            }
        }

        if (c == null)
        {
            try
            {
                c= this.findClass(name);
            }
            catch (ClassNotFoundException e)
            {
                ex= e;
            }
        }

        if (c == null && _parent!=null && !tried_parent && !server_class )
            c= _parent.loadClass(name);

        if (c == null)
            throw ex;

        if (resolve)
            resolveClass(c);

        if (LOG.isDebugEnabled())
            LOG.debug("loaded " + c+ " from "+c.getClassLoader());
        
        return c;
    }
loadclass按照：

1. findLoadedClass(name)-检查类是否已经加载
2. 判断该类是否为系统类或server类
3. 如果该类未加载且父加载器不为空且设置了父加载器优先或类类为系统类，且该类不是server类，则尝试使用父加载器加载该类
4. 如果不是父加载器优先或者父加载器未加载到该类，使用WebAppClassLoader加载该类
5. 如果是不是父加载器优先，并且WebAppClassLoader未加载到该类，尝试使用父加载器加载该类
6. 找到则返回，否则抛出ClassNotFoundException

###ClassLoader Priority
上述过程涉及一个加载器优先级的概念，这也是针对前述第一条规范中WEB-INF/lib和WEB-INF/classes类优先的处理。jetty中父加载器优先的配置项可以通过环境变量

	org.eclipse.jetty.server.webapp.parentLoaderPriority=false(默认)/true来设置

也可以通过

	org.eclipse.jetty.webapp.WebAppContext.setParentLoaderPriority(boolean)方法来设置
优于该配置默认是false，因此在load class过程中优先使用WebAppClassLoader加载WEB-INF/lib和WEB-INF/classes中的类。
当将该配置项设为true时需要确认类加载顺序没有问题。

###设置系统类
规范2中约定系统类不能被应用类覆盖，但是没有明确规定哪些时系统类，jetty中以类的package路径名来区分，当类的package路径名位包含于	

	  public final static String[] __dftSystemClasses =
	    {
	        "java.",                            
	        "javax.",                           
	        "org.xml.",                         
	        "org.w3c.",                         
	        "org.apache.commons.logging.",      
	        "org.eclipse.jetty.continuation.",  
	        "org.eclipse.jetty.jndi.",          
	        "org.eclipse.jetty.plus.jaas.",     
	        "org.eclipse.jetty.websocket.WebSocket", 
	        "org.eclipse.jetty.websocket.WebSocketFactory", 
	        "org.eclipse.jetty.servlet.DefaultServlet" 
	    } ;
时，会被认为是系统类。（该定义位于[WebAppContext@github](https://github.com/eclipse/jetty.project/blob/master/jetty-webapp/src/main/java/org/eclipse/jetty/webapp/WebAppContext.java)中）

因此，我们可以通过 org.eclipse.jetty.webapp.WebAppContext.setSystemClasses(String Array)或者org.eclipse.jetty.webapp.WebAppContext.addSystemClass(String)来设置系统类。
再次提醒，系统类是对多有应用都可见。
###设置Server类
规范3中约定Server类不对任何应用可见。jetty同样是用package路径名来区分哪些是Server类。Server类包括：

	public final static String[] __dftServerClasses =
    {
        "-org.eclipse.jetty.continuation.", 
        "-org.eclipse.jetty.jndi.",         
        "-org.eclipse.jetty.plus.jaas.",    
        "-org.eclipse.jetty.websocket.WebSocket", 
        "-org.eclipse.jetty.websocket.WebSocketFactory", 
        "-org.eclipse.jetty.servlet.DefaultServlet", 
        "-org.eclipse.jetty.servlet.listener.", 
        "org.eclipse.jetty."                
    } ;

我们可以通过， org.eclipse.jetty.webapp.WebAppContext.setServerClasses(String Array) 或org.eclipse.jetty.webapp.WebAppContext.addServerClass(String)方法设置Server类。
注意，Server类是对所有应用都不可见的，但是WEB-INF/lib下的类可以替换Server类。
###自定义WebApp ClassLoader
当默认的WebAppClassLoader不能满足需求时，可以自定义WebApp ClassLoader，不过jetty建议自定义的classloader要扩展于默认的WebAppClassLoader实现。具体请参考[jetty手册](http://wiki.eclipse.org/Jetty/Reference)
	
