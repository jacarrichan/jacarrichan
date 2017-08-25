title: 简单讨论spring代理对事务的影响
date: 2017-01-06 19:32:42
tags: [spring,AOP,事务,动态代理,CGlib]
---







一般情况下，只要项目中配置没有问题，Controller调用service功能的时候，事务都是没有问题，但是当servcie里面使用this调用当前类的other方法时，事务配置就失效了。然后很多人就开始纳闷了。

# 代码演化

首先看下最开始手动管理事务的代码:

```
import java.sql.*;

public class TestTransaction {
	private static Connection conn = null;

	public static void doBusi(String[] args) throws SQLException {
		Statement stmt = null;
		conn.setAutoCommit(false);
		stmt = conn.createStatement();
		stmt.addBatch("insert into student values (1, 'xiaolan', '很优秀！')");
		stmt.addBatch("insert into student values (2, 'xiaocao', '很优秀！')");
		stmt.executeBatch();
		conn.commit();
		conn.setAutoCommit(true);
		conn.rollback();
		conn.setAutoCommit(true);
	}
}

```
>  为便于分析，关闭连接的代码我已经删掉

然后我们将main方法拆开，拆成spring帮我们管理事务后的写法

```
import java.sql.*;

public class TestTransaction {
	private static Connection conn = null;
	private static Statement stmt;

	public void start() throws SQLException {
		Statement stmt = null;
		conn.setAutoCommit(false);
		stmt = conn.createStatement();
	}

	public void doBusi(String[] args) throws SQLException {
		//start();
		stmt.addBatch("insert into student values (1, 'xiaolan', '很优秀！')");
		stmt.addBatch("insert into student values (2, 'xiaocao', '很优秀！')");
		//commit();
	}

	public void commit() throws SQLException {
		stmt.executeBatch();
		conn.commit();
		conn.setAutoCommit(true);
	}
}

```
一眼都能看明白，spring帮我们管理事务后，start和commit这两个方法是多余的，而在实际运行过程中又是必须要这两个方法存在，不然数据不能成功提交到数据库(auto commit 为true就抛开不谈了)

下面来分析spring是如何帮我们引入start和commit这两个方法

# 代码织入

代码的织入一般可在三个时期完成：编译期织入、装载期织入、运行时织入。前两种用的比较少，我只简单提一下，用得比较多的运行时织入后面会重点探讨。

## 编译期织入、装载期织入

start 和commit这两个方法是在编译器(javac)或者classloader加载的时候引入的，你可以简单理解 doBusi两行注释的代码被打开了，当然start和commit这两个方法可以放在别的类，没必要在你的业务类也给整进来。  

## 运行时织入

最常见的有JDK动态代理和CGlib，
如果你的业务类实现了接口，那就默认用前者，否者就只能用后者。如果你没有实现接口还定义成final，那运行时织入也无能为力了。

看下面的一段话时，你需要弄清楚什么是继承，什么是组合。

假设，Controller类名是UserController，Service接口是UserService，其实现类名是UserServiceImpl，其实现类Proxy的类名是UserServiceImplProxy。那么运行时会有如下的关系。
1. 使用JDK的动态代理：UserService有两个实现类，分别是UserServiceImpl和UserServiceImplProxy，UserServiceImplProxy组合了UserServiceImpl，你可以认为UserServiceImplProxy有一个属性是UserServiceImpl。
1. 使用CGlib的动态代理: UserServiceImplProxy继承了UserServiceImpl。

不管使用的何种动态代理，在代码运行中调用链为：{UserController→[UserServiceImplProxy.doBusi→(UserServiceImpl.doBusi)]};
把这个调用链用缩进来看吧，这样清晰些：

表面上是这样的
```
{
	UserController → [

			UserServiceImpl.doBusi();

		)
	]
}
```

实际上运行时是这样的：

```
{
	UserController → [
		UserServiceImplProxy.doBusi → (
			//.....  打开事务
			UserServiceImpl.doBusi();
			//...   关闭和提交事务
		)
	]
}
```

可以看到这样的织入级别是方法级别的织入，看上去是在我们自己写的UserServiceImpl.doBusi()外面包裹了事务相关的代码，我们称之为增强。 
 那么最开始提出使用this调用后事务不生效的原因就在这里，因为Controller表面上调用UserServiceImpl代码，实际上是调用了被增强了事务相关的UserServiceImplProxy.doBusi()。而在UserServiceImpl.doBusi()里面使用this调用的方法是没有被增强的，那么针对该方法自己的事务就不生效了的。



下面又要开始上代码了

```
public class Main {

    public static void main(String[] args) {
        System.out.println("默认对接口使用JDK的动态代理");
        ProxyFactory interfaceFactory = new ProxyFactory(new SimplePojo());
        interfaceFactory.addInterface(Pojo.class);
        interfaceFactory.addAdvice(new RetryAdvice());
        interfaceFactory.setExposeProxy(true);
        Pojo pojo = (Pojo) interfaceFactory.getProxy();
        // this is a method call on the proxy!
        pojo.foo();
        System.err.println("==============================================");
        System.out.println("强制使用CGLIB动态代理");
        interfaceFactory.setOptimize(true);
        ((Pojo) interfaceFactory.getProxy()).foo();
    }
}

```

```

import org.springframework.aop.framework.AopContext;

public class SimplePojo implements Pojo {

    public void foo() {
        System.err.println("1------------------------- start ");
        System.err.println("马上能看到RetryAdvice的打印");
        Pojo proxy = ((Pojo) AopContext.currentProxy());
        proxy.bar();
        System.err.println("1------------------------- end ");
        System.err.println("\r\n");
        System.err.println("2------------------------- start ");
        Pojo current = this;
        current.bar();
        System.err.println("不能看到RetryAdvice的打印，因为没有调用代理");
        System.err.println("2------------------------- end ");

        System.err.println("代理类： " + proxy.getClass());
        System.err.println("this类： " + current.getClass());
        System.err.println("代理对象是不是当前类的实例-------------------------  " + (current.getClass().isInstance(proxy)));
        System.err.println("代理对象是不是当前类父类的实例-------------------------  " + (current.getClass().getSuperclass().isInstance(proxy)));

    }

    public void bar() {
    }
}
```
说明下： 上述代码copy 自spring官网文档  [Understanding AOP proxies](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#aop-understanding-aop-proxies "Understanding AOP proxies" )

打印结果可能如下（JVM执行时会优化代码，所以不同机器上结果会不一样）

```

默认对接口使用JDK的动态代理
Before advice
1------------------------- start 
Before advice
马上能看到RetryAdvice的打印
1------------------------- end 


2------------------------- start 
不能看到RetryAdvice的打印，因为没有调用代理
2------------------------- end 
代理类： class com.sun.proxy.$Proxy0
this类： class SimplePojo
代理对象是不是当前类的实例-------------------------  false
代理对象是不是当前类父类的实例-------------------------  true


==============================================
强制使用CGLIB动态代理
Before advice
1------------------------- start 
Before advice
马上能看到RetryAdvice的打印
1------------------------- end 


2------------------------- start 
不能看到RetryAdvice的打印，因为没有调用代理
2------------------------- end 
代理类： class SimplePojo$$EnhancerBySpringCGLIB$$974071c9
this类： class SimplePojo
代理对象是不是当前类的实例-------------------------  true
代理对象是不是当前类父类的实例-------------------------  true

```




有必要说明下打印结果，使用JDK动态代理时，代理对象不是当前类的实例；使用CGlib动态代理时，代理对象是当前类的实例；这两种不同的现象说明了前者不是用的继承，而后者是用的继承；



 关于保留运行时由JVM生成的动态代理类，可以在JVM启动时指定启动参数保留在硬盘中，我们可以拿到后反编译，看它是如何利用组合这个模式的。这个下次说~~~ 肚子饿了，下班吃饭~~ 
