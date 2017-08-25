title: 分析JDK动态代理类
date: 2017-01-09 09:55:25
tags: [动态代理,InvocationHandler ]
---


在mybatis编写面向接口的mapper时，只用写接口，不用写实现类就可以。但是接口不能new不能创建实例，那在运行时肯定类实现了mapper这个接口。而创建mapper接口的就是使用的是动态代理。
<!--more-->
下面贴一个JDK动态代理的例子，因为等下我们需要它运行时的结果。
# 动态代理 Demo

```
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class JdkProxyDemo {
	interface If {
		void originalMethod(String s);
	}

	static class Original implements If {
		public void originalMethod(String s) {
			System.out.println(s);
		}
	}

	static class Handler implements InvocationHandler {
		private final If original;

		public Handler(If original) {
			this.original = original;
		}

		public Object invoke(Object proxy, Method method, Object[] args)
				throws IllegalAccessException, IllegalArgumentException,
				InvocationTargetException {
			System.out.println("BEFORE");
			Object result = method.invoke(original, args);
			System.out.println("AFTER");
			return result;
		}
	}

	public static void main(String[] args) {
		Original original = new Original();
		Handler handler = new Handler(original);
		If f = (If) Proxy.newProxyInstance(If.class.getClassLoader(),
				new Class[] { If.class }, handler);
		f.originalMethod("Hallo");
	}
}

```

编译后使用如下命令执行

```
java  -Dsun.misc.ProxyGenerator.saveGeneratedFiles=true JdkProxyDemo
```
# 分析动态代理

眼尖的同学应该能发现多了一个文件，一般叫$Proxy0.class,这个class就是运行时由JDK生成的动态代理类。通常程序运行时，JDK不会保留动态代理class，大家如果想知道JDK是如何生成这个动态代理class，可以参阅JDK源码[ProxyGenerator.java](http://hg.openjdk.java.net/jdk7u/jdk7u/jdk/file/d6bfaec7e2c9/src/share/classes/sun/misc/ProxyGenerator.java,'ProxyGenerator'),其中该类的属性saveGeneratedFiles已经说明了如何在运行时才能保存动态代理类，。 


接下来使用反编译工具比如jd-gui反编译动态代理类,可以看到如下代码：

```
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy
  implements JdkProxyDemo.If
{
  private static Method m1;
  private static Method m0;
  private static Method m3;
  private static Method m2;

  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject)
    throws 
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final int hashCode()
    throws 
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final void originalMethod(String paramString)
    throws 
  {
    try
    {
      this.h.invoke(this, m3, new Object[] { paramString });
      return;
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final String toString()
    throws 
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m3 = Class.forName("JdkProxyDemo$If").getMethod("originalMethod", new Class[] { Class.forName("java.lang.String") });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
    }
    throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
  }
}
```

通过上述反编译后的代码可以看到如下特点

1. 动态代理类是java.lang.reflect.Proxy的子类；
1. 动态代理类是final类，所以该类不能被继承；
1. 动态代理类的方法都是final修饰的；
1. 动态代理中hashCode、equals 和 toString 方法声明的类是Object；


动态代理类实现的方法和被代理的类（javadoc称为调用处理程序）方法是调用链是这样的：

```
"main@1" prio=5 tid=0x1 nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at JdkProxyDemo$Original.originalMethod(JdkProxyDemo.java:13)
	  at sun.reflect.NativeMethodAccessorImpl.invoke0(NativeMethodAccessorImpl.java:-1)
	  at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	  at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	  at java.lang.reflect.Method.invoke(Method.java:497)
	  at JdkProxyDemo$Handler.invoke(JdkProxyDemo.java:28)
	  at $Proxy0.originalMethod(Unknown Source:-1)
	  at JdkProxyDemo.main(JdkProxyDemo.java:39)
```

注意看下每个方法中有这样的代码:[this.h],我们根据刚才的调用链可以看出this.h指向被代理的类，因为我们从这里看到JDK用到的一个设计模式：组合。

# 组合

下面贴个题外话，类似彩蛋吧：

　组合也就是设计类的时候把要组合的类的对象加入到该类中作为自己的成员变量。

### 组合的优点：

1. 当前对象只能通过所包含的那个对象去调用其方法，所以所包含的对象的内部细节对当前对象时不可见的。

1. 当前对象与包含的对象是一个低耦合关系，如果修改包含对象的类中代码不需要修改当前对象类的代码。

1. 当前对象可以在运行时动态的绑定所包含的对象。可以通过set方法给所包含对象赋值。

### 组合的缺点：

1. 容易产生过多的对象。
1. 为了能组合多个对象，必须仔细对接口进行定义。

组合比继承更具灵活性和稳定性，所以在设计的时候优先使用组合。


