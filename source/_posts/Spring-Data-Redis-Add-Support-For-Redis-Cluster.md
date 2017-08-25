title: 让spring-data-redis支持redis cluster
date: 2016-02-26 16:19:08
category:
tags: [Spring,Redis,Proxy]
---



最近把使用了spring service cache的项目从单点redis迁移到集群环境的redis时，出现了下面的异常

```
Caused by: redis.clients.jedis.exceptions.JedisMovedDataException: MOVED 3688 192.168.164.45:7000
	at redis.clients.jedis.Protocol.processError(Protocol.java:108)
	at redis.clients.jedis.Protocol.process(Protocol.java:151)
	at redis.clients.jedis.Protocol.read(Protocol.java:205)
	at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:297)
	at redis.clients.jedis.Connection.getIntegerReply(Connection.java:222)
	at redis.clients.jedis.BinaryJedis.exists(BinaryJedis.java:181)
	at org.springframework.data.redis.connection.jedis.JedisConnection.exists(JedisConnection.java:779)
	... 54 more
```

出现异常的代码块如下：

```
public class JedisConnection extends AbstractRedisConnection {
	public Boolean exists(byte[] key) {
		try {
			if (isPipelined()) {
				pipeline(new JedisResult(pipeline.exists(key)));
				return null;
			}
			if (isQueueing()) {
				transaction(new JedisResult(transaction.exists(key)));
				return null;
			}
			return jedis.exists(key);
		} catch (Exception ex) {
			throw convertJedisAccessException(ex);
		}
	}
}
```

查阅spring-data-redis官网和stackoverflow网站后，发现spring-data-redis现在确实不支持redis cluster。

https://jira.spring.io/browse/DATAREDIS-315
https://jira.spring.io/browse/DATAREDIS-468


##  发现redis cluster不支持传统redis的一些命令

根据spring-data-redis官网提示翻看了支持redis cluster的客户端jedis网站和其源码，发现有新增用于支持redis cluster的类：JedisCluster、JedisClusterCommand、JedisClusterConnectionHandler。便想着能不能直接将JedisCluster替换JedisConnection里面的Jedis。根据类之间的继承关系发现不能替换，原因是JedisCluste不是Jedis的子类.且研究两者各自的继承关系发现，Jedis支持的命令比JedisCluster多。那么更进一步想，如果在运行时把外部调用redis cluster不支持的命令给屏蔽掉，是不是就可以支持redis cluster了？这个先放在这里。


Jedis & JedisCluster 各自的继承关系如下：
> Jedis extends BinaryJedis implements JedisCommands, MultiKeyCommands,AdvancedJedisCommands, ScriptingCommands, BasicCommands, ClusterCommands, SentinelCommands
> JedisCluster implements JedisCommands, BasicCommands, Closeable 

##  jedis处理node跳转
继续研究支持redis cluster的及各类发现，JedisCluster是这样处理node跳转的，当出现JedisMovedDataException异常时，获取异常里面的提示的主机地址和端口地址，根据这一信息构造并使用新的redis client实例去操作redis node ，这样就可以了。

```
public abstract class JedisClusterCommand<T> {
 private T runWithRetries(String key, int redirections, boolean tryRandomNode, boolean asking) {
    if (redirections <= 0) {
      throw new JedisClusterMaxRedirectionsException("Too many Cluster redirections?");
    }

    Jedis connection = null;
    try {

      if (asking) {
        // TODO: Pipeline asking with the original command to make it
        // faster....
        connection = askConnection.get();
        connection.asking();

        // if asking success, reset asking flag
        asking = false;
      } else {
        if (tryRandomNode) {
          connection = connectionHandler.getConnection();
        } else {
          connection = connectionHandler.getConnectionFromSlot(JedisClusterCRC16.getSlot(key));
        }
      }

      return execute(connection);
    } catch (JedisConnectionException jce) {
      if (tryRandomNode) {
        // maybe all connection is down
        throw jce;
      }

      // release current connection before recursion
      releaseConnection(connection);
      connection = null;

      // retry with random connection
      return runWithRetries(key, redirections - 1, true, asking);
    } catch (JedisRedirectionException jre) {
      // release current connection before recursion or renewing
      releaseConnection(connection);
      connection = null;

      if (jre instanceof JedisAskDataException) {
        asking = true;
        askConnection.set(this.connectionHandler.getConnectionFromNode(jre.getTargetNode()));
      } else if (jre instanceof JedisMovedDataException) {
        // it rebuilds cluster's slot cache
        // recommended by Redis cluster specification
        this.connectionHandler.renewSlotCache();
      } else {
        throw new JedisClusterException(jre);
      }

      return runWithRetries(key, redirections - 1, false, asking);
    } finally {
      releaseConnection(connection);
    }

  }

}
```



##  处理思路

1. spring-data-redis使用的是Jedis类，而这个Jedis又不能使用JedisCluster来替换，但是我可以在运行时把redis cluster不支持的命令给替换掉；
1. Jedis不支持node跳转的问题，那我可以学习JedisCluster中的代码，让它支持跳转；

综合上述两点：就是要在现有的代码中嵌入动过手脚的Jedis


##  嵌入代码

使用spring-data-redis时，xml中一般配置RedisCacheManager引用redisTemplate，redisTemplate引用jedisConnectionFactory，jedisConnectionFactory引用JedisPoolConfig。那么要嵌入自己的代码，就得在这三个关系中动手。
继续分析，发现jedisConnectionFactory的方法[public JedisConnection getConnection()]是用来提供Jedis实例的，那么我只要让它返回被我动过手脚的Jedis就可以了。

因此用aop拦截这个方法，这里使用ProxyFactoryBean来织入自己的代码。
织入代码片段如下：

```
public class JedisConnectionWrapperClusterInvocationHandler implements MethodInterceptor, DisposableBean {
	protected RedisConnection createRedisConnectionProxy(JedisCluster jedisCluster, int redirections) {
		Class<?>[] ifcs = ClassUtils.getAllInterfacesForClass(RedisConnection.class, getClass().getClassLoader());
		return (RedisConnection) Proxy.newProxyInstance(jedisCluster.getClass().getClassLoader(), ifcs,
				new JedisConnectionWrapper(jedisCluster, redirections));
	}

	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		Class<?> returnType = invocation.getMethod().getReturnType();
		Object returnVal = null;
		// 只拦截获取RedisConnection的getConnection方法
		if (returnType.isAssignableFrom(RedisConnection.class)) {
			returnVal = createRedisConnectionProxy(jedisCluster, redirections);
		} else {
			returnVal = invocation.proceed();
		}
		return returnVal;
	}
}
```


下面贴上如何屏蔽redis cluster不支持的命令以及node跳转的代码:

```
public class JedisConnectionWrapper implements InvocationHandler {

	private static final String CLOSE = "close";
	private static final String HASH_CODE = "hashCode";
	private static final String EQUALS = "equals";
	private static final String MULTI = "multi";
	private static final String EXEC = "exec";

	private JedisClusterConnectionHandler connectionHandler;
	static Method METHOD_GETCONNECTION = null;
	static Method METHOD_GETCONNECTIONFROMSLOT = null;
	static Field FIELD_CONNECTIONHANDLER = null;
	private ThreadLocal<Jedis> askConnection = new ThreadLocal<Jedis>();
	private int redirections;
	static {
		METHOD_GETCONNECTION = ReflectionUtils.findMethod(JedisClusterConnectionHandler.class, "getConnection");
		METHOD_GETCONNECTION.setAccessible(true);
		METHOD_GETCONNECTIONFROMSLOT = ReflectionUtils.findMethod(JedisClusterConnectionHandler.class,
				"getConnectionFromSlot", int.class);
		METHOD_GETCONNECTIONFROMSLOT.setAccessible(true);
		FIELD_CONNECTIONHANDLER = ReflectionUtils.findField(JedisCluster.class, "connectionHandler");
		FIELD_CONNECTIONHANDLER.setAccessible(true);
	}

	@SuppressWarnings("unchecked")
	public JedisConnectionWrapper(JedisCluster jedisCluster, int redirections) {
		this.redirections = redirections;
		connectionHandler = (JedisClusterConnectionHandler) ReflectionUtils.getField(FIELD_CONNECTIONHANDLER,
				jedisCluster);
		Assert.notNull(connectionHandler, "connectionHandler 获取失败");
	}

	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

		if (method.getName().equals(EQUALS)) {
			// Only consider equal when proxies are identical.
			return (proxy == args[0]);
		} else if (method.getName().equals(HASH_CODE)) {
			// Use hashCode of PersistenceManager proxy.
			return System.identityHashCode(proxy);
		} else if (method.getName().equals(CLOSE)) { // Handle close
			// method: suppress, not valid. 下文中用完了就关了，所以调这个close也没有意义，索性屏蔽掉
			return null;
		} else if (method.getName().equals(MULTI)) {
			// 集群不支持事务，所以跳过
			log.warn("集群不支持事务，所以跳过:{}", MULTI);
			return null;
		} else if (method.getName().equals(EXEC)) {
			// 集群不支持事务，所以跳过
			log.warn("集群不支持事务，所以跳过:{}", EXEC);
			return null;
		}
		// Invoke method on target RedisConnection.
		Object retVal = runWithRetries(method, args, redirections, false, false);
		return retVal;
	}

	private Object runWithRetries(Method method, Object[] args, int redirections, boolean tryRandomNode, boolean asking) {
		if (redirections <= 0) {
			throw new JedisClusterMaxRedirectionsException("Too many Cluster redirections?");
		}
		Map<String, JedisPool> map = connectionHandler.getNodes();
		Assert.notEmpty(map, "没有可用的连接资源");
		Jedis connection = null;
		// if (null == connection) {
		// connection =
		// map.entrySet().iterator().next().getValue().getResource();
		// }
		JedisConnection jconnection = null;
		JedisConnection target;
		try {
			if (asking) {
				// TODO: Pipeline asking with the original command to make it
				// faster....
				connection = askConnection.get();
				connection.asking();

				// if asking success, reset asking flag
				asking = false;
			} else {
				connection = (Jedis) METHOD_GETCONNECTION.invoke(connectionHandler);
			}
			// --------------------------
			Client client = connection.getClient();
			log.trace("redis node 当前值：" + client.getHost() + "------" + client.getPort());
			// jconnection = new JedisConnection(connection, pool, 0);// redis
			// cluster只支持0号数据库，所以这里写死
			jconnection = new JedisConnection(connection);// redis
			target = jconnection;//在这里偷偷的替换掉 jedisconnection
			Object result = method.invoke(target, args);
			String commandName = method.getName();
			log.debug("exec command:{} @{}:{}", commandName, client.getHost(), client.getPort());
			return result;
		} catch (Exception e) {
			releaseConnection(connection);
			connection = null;
			try {
				Throwable t = e.getCause();
				if (e instanceof InvocationTargetException) {// 拿出反射包裹的异常
					t = ((InvocationTargetException) e).getTargetException();
					if (t instanceof NestedRuntimeException) {// 拿出spring包裹的异常
						t = t.getCause();
					}
				}
				throw t;
			} catch (JedisRedirectionException jre) {
				// release current connection before recursion or renewing
				releaseConnection(connection);
				connection = null;
				if (jre instanceof JedisAskDataException) {
					asking = true;
					HostAndPort node = jre.getTargetNode();
					log.trace("redis node 期望值：" + node.getHost() + "---EEEE---" + node.getPort());
					askConnection.set(this.connectionHandler.getConnectionFromNode(node));
				} else if (jre instanceof JedisMovedDataException) {
					// it rebuilds cluster's slot cache
					// recommended by Redis cluster specification
					this.connectionHandler.renewSlotCache();
					asking = true;
					HostAndPort node = jre.getTargetNode();
					log.trace("redis node 期望值：" + node.getHost() + "---BBBBB--" + node.getPort());
					askConnection.set(this.connectionHandler.getConnectionFromNode(node));
				} else {
					throw new JedisClusterException(jre);
				}
				return runWithRetries(method, args, redirections - 1, false, asking);
			} catch (Throwable t) {
				e.printStackTrace();
			}
		} finally {
			releaseConnection(connection);
		}
		return null;
	}
```


>  经过上面的操作后，  业务代码不用改动，只需在xml中配置下就能支持redis  cluster了.
