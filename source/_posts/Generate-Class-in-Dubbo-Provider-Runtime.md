title: 在Dubbo Provider运行时生成pojo的class
date: 2016-02-26 16:19:08
category:
tags:  [Dubbo,hessian]
---

 问题简述：

  没有在provider中引入consumer会传过来的子类，导致provider找不到子类后会反序列化为父类实例，同时子类里面扩展的属性都会丢失，导致mybatis里面的动态sql不能访问子类属性。

  <!-- more -->
具体场景：

  consumer-A,调用consumer-B，consumer-B调用provider-B，其中consumer-B即充当consumer角色、也充当provider角色，consumer-A和consumer-B都引用了consumer-B-api，调用时传递的也是consumer-B-api里面的类，所以当consumer-B调用provider-B时需要将consumer-B-api里面类的实例手动转换为provider-B-api里面类的实例，嫌麻烦，所以就没转。consumer-B直接透传了consumer-A发来的参数给provider-B。


如果在provider-B中能自己生成consumer-B-api里面的类并且供mybatis访问，代码就变得简单了。


## 代码场景

### 接口及抽象父类定义如下

```
public interface BaseService<T>{
    public List<T>  findList(BaseQuery query);
}

public interface CatService extends BaseService<Cat>{}

public abstract class BaseServiceImpl<T> extends BaseService<T>{
    public List<T>  findList(BaseQuery query){
       return  getMapper().selectListByQuery(query);    
    }
}
```

### provider中某个具体的业务类定义如下

```
public class CatService extends BaseService<Cat>{
    @Autowire
    private CatMapper catMapper;
    //provider中没有引入CatQuery
    public CatMapper getMapper(){
        return this.catMapper;
    }
}
```

---------------------------------------------
### Consumer中单元测试类如下

```
public class   CatServiceTest{
    @Autowire
    private CatService service
        
   @Test
   public void testFindList(){
    BaseQuery query = new CatQuery();
    List<Cat> catList = service.findList(query);
   }
}
```

## 暴露的问题
执行上述单元测试后，可以看到provider会有如下警告提示，并且mybatis中也访问不到子类中的属性。

```
二月 26, 2016 4:35:07 下午 com.alibaba.com.caucho.hessian.io.SerializerFactory getDeserializer
警告: Hessian/Burlap: 'CatQuery' is an unknown class in sun.misc.Launcher$AppClassLoader@56e88e24:
java.lang.ClassNotFoundException: CatQuery
```



同时可以看到dubbo使用的是Hessian来反序列化参数的，当找不到CatQuery后，就会使用父类BaseQuery来反序列化，这样子类所扩展的属性就丢失了。

## 分析代码

将断点打在打印警告的地方，查看调用链如下：

```
Daemon Thread [New I/O server worker #1-2] (Suspended (breakpoint at line 528 in SerializerFactory))	
	Hessian2SerializerFactory(SerializerFactory).getDeserializer(String) line: 528	
	Hessian2SerializerFactory(SerializerFactory).getObjectDeserializer(String) line: 436	
	Hessian2SerializerFactory(SerializerFactory).getObjectDeserializer(String, Class) line: 413	
	Hessian2Input.readObjectInstance(Class, Hessian2Input$ObjectDefinition) line: 2065	
	Hessian2Input.readObject(Class) line: 1592	
	Hessian2Input.readObject(Class) line: 1576	
	Hessian2ObjectInput.readObject(Class<T>) line: 94	
	DecodeableRpcInvocation.decode(Channel, InputStream) line: 109	
	DecodeableRpcInvocation.decode() line: 71	
	DubboCodec.decodeBody(Channel, InputStream, byte[]) line: 137	
	DubboCodec(ExchangeCodec).decode(Channel, InputStream, int, byte[]) line: 128	
	DubboCodec(ExchangeCodec).decode(Channel, InputStream) line: 87	
	DubboCountCodec.decode(Channel, InputStream) line: 49	
	NettyCodecAdapter$InternalDecoder.messageReceived(ChannelHandlerContext, MessageEvent) line: 135	
	NettyCodecAdapter$InternalDecoder(SimpleChannelUpstreamHandler).handleUpstream(ChannelHandlerContext, ChannelEvent) line: 80	
	DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline$DefaultChannelHandlerContext, ChannelEvent) line: 564	
	DefaultChannelPipeline.sendUpstream(ChannelEvent) line: 559	
	Channels.fireMessageReceived(Channel, Object, SocketAddress) line: 274	
	Channels.fireMessageReceived(Channel, Object) line: 261	
	NioWorker.read(SelectionKey) line: 349	
	NioWorker.processSelectedKeys(Set<SelectionKey>) line: 280	
	NioWorker.run() line: 200	
	ThreadRenamingRunnable.run() line: 108	
	DeadLockProofWorker$1.run() line: 44	
	ThreadPoolExecutor.runWorker(ThreadPoolExecutor$Worker) line: 1145	
	ThreadPoolExecutor$Worker.run() line: 615	
	Thread.run() line: 744	
```

Hessian2SerializerFactory(SerializerFactory).getDeserializer(String)部分代码如下：

```
      try {
	Class cl = Class.forName(type, false, _loader);
	deserializer = getDeserializer(cl);
      } catch (Exception e) {
	log.warning("Hessian/Burlap: '" + type + "' is an unknown class in " + _loader + ":\n" + e);
	log.log(Level.FINER, e.toString(), e);
      }
```

###  分析Hessian2SerializerFactory的引入
查看/META-INF/dubbo/internal/com.alibaba.dubbo.common.serialize.Serialization文件里面引入了Hessian2Serialization，Hessian2Serialization引用了Hessian2InputEnhance，同时Hessian2InputEnhance调用了Hessian2SerializerFactory。

CodecSupport使用com.alibaba.dubbo.common.serialize.Serialization文件的代码如下：

```
    static {
        Set<String> supportedExtensions = ExtensionLoader.getExtensionLoader(Serialization.class).getSupportedExtensions();
        for (String name : supportedExtensions) {
            Serialization serialization = ExtensionLoader.getExtensionLoader(Serialization.class).getExtension(name);
            byte idByte = serialization.getContentTypeId();
            if (ID_SERIALIZATION_MAP.containsKey(idByte)) {
                logger.error("Serialization extension " + serialization.getClass().getName()
                                 + " has duplicate id to Serialization extension "
                                 + ID_SERIALIZATION_MAP.get(idByte).getClass().getName()
                                 + ", ignore this Serialization extension");
                continue;
            }
            ID_SERIALIZATION_MAP.put(idByte, serialization);
        }
    }
```

从这个代码可以看出dubbo不允许覆盖自带的Serialization，如果能否覆盖Hessian2Serialization，那么就能引入自己的代码了。

##  大致思路
debug时看到Hessian2Input.readObjectInstance方法传入了父类class和带有子类类名及其属性的参数Hessian2Input$ObjectDefinition，那么根据这些信息就可以在provider中构造出子类class，并且加载到当前线程Classloader中就可以了。


##  解决步骤
### 引入自己的代码
上面分析了dubbo不允许覆盖自带的Serialization，查看其加载Serialization的代码是在CodecSupport的静态块中，如果在该静态块加载完成后使用自己的Hessian2SerializationEnhance覆盖ID_SERIALIZATION_MAP中Hessian2SerializerFactory，那么目的就达到了。

随之问题就变成“在何处读取CodecSupport的ID_SERIALIZATION_MAP属性”，想到利用dubbo提供的filter扩展，在filter的构造器中就可以完成这个功能。

```
public class ReplaceDubboHessianCodecFilter implements Filter {
 
    public ReplaceDubboHessianCodecFilter() {
        try {
            Field field = CodecSupport.class.getDeclaredField("ID_SERIALIZATION_MAP");
            field.setAccessible(true);
            Map<Byte, Serialization> ID_SERIALIZATION_MAP = (Map<Byte, Serialization>) field.get(null);
            Serialization ser = ID_SERIALIZATION_MAP.get(Hessian2Serialization.ID);
            Hessian2Serialization serialization=new Hessian2SerializationEnhance();
            log.warn("将替换dubbo的hessian序列化代码替换为{}",serialization.getClass());
            ID_SERIALIZATION_MAP.put(Hessian2Serialization.ID,serialization);
        }
        catch (IllegalArgumentException | IllegalAccessException | NoSuchFieldException | SecurityException e) {
            e.printStackTrace();
        }
    }


    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }
}
```

###  运行时生成pojo的class
顺着dubbo自己的思路，Hessian2SerializationEnhance引入Hessian2ObjectInputEnhance，Hessian2ObjectInputEnhance引入Hessian2SerializerFactoryEnhance。
其中Hessian2ObjectInput中readObjectInstance方法用private修饰，那就直接复制Hessian2ObjectInput源码到Hessian2ObjectInputEnhance中，在readObjectInstance方法中生成子类。代码如下：

```
  private void checkAndLoadNotexistClass(Class cl, ObjectDefinition def){
      String type = def.getType();
      String [] fieldNames = def.getFieldNames();
      ClassLoader _loader = Thread.currentThread().getContextClassLoader();
      boolean isUnknownCls = false;
      try {
          Class.forName(type, false, _loader);
      }
      catch (ClassNotFoundException e) {
          isUnknownCls = true;
      }
      if (isUnknownCls) {
          System.err.println("不存在这个类，需要运行时生成---->"+type);
         Class cls= dubboClassloader.createAndLoadClass(cl,type, Arrays.asList(fieldNames));
      }          
  }
```

使用javassist构造class并且使用ClassLoader加载的代码如下：

```
public class DubboClassloader extends ClassLoader {
    private final static String SETTER_STR = "set";
    private final static String GETTER_STR = "get";
    // type/fieldName
    private final static String fieldTemplate = "private %s %s;";


    public Class<?> createAndLoadClass(Class cl, String className, List<String> fields) {
        byte[] classBytes = DubboClassloader.createBeanClass(cl, className, fields);
        // -----------------
        ClassLoader _loader = Thread.currentThread().getContextClassLoader();
        Method method =
                ReflectionUtils.findMethod(_loader.getClass(), "defineClass", String.class, byte[].class,
                    int.class, int.class);
        method.setAccessible(true);
        Class cls = null;
        try {
            // 想办法加载到当前线程的classloader中
            method.invoke(_loader, className, classBytes, 0, classBytes.length);
            cls = Class.forName(className);
        }
        catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException
                | ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
        return cls;
    }


    /**
     * 构造class
     * 
     * @param cl
     * @param className
     * @param fields
     * @return
     */
    public static byte[] createBeanClass(Class superClass, String className, List<String> fields) {
        try {
            ClassPool cpool = ClassPool.getDefault();
            CtClass cc = cpool.makeClass(className);

            String setMethodName = null;
            String getMethodName = null;
            List<String> superFieldNames = new ArrayList<>();
            if (null != superClass) {
                Field[] superFields = superClass.getDeclaredFields();
                for (Field field : superFields) {
                    superFieldNames.add(field.getName());
                }
                cc.setSuperclass(cpool.get(superClass.getCanonicalName()));
            }
            for (String field : fields) {
                // 忽略父类已经存在的属性
                if (superFieldNames.contains(field)) {
                    continue;
                }
                setMethodName = SETTER_STR + StringUtils.capitalize(field);
                getMethodName = GETTER_STR + StringUtils.capitalize(field);

                CtField newField = CtField.make(String.format(fieldTemplate, "java.lang.Object", field), cc);
                cc.addField(newField);

                CtMethod ageSetter = CtNewMethod.setter(setMethodName, newField);
                cc.addMethod(ageSetter);
                CtMethod ageGetter = CtNewMethod.getter(getMethodName, newField);
                cc.addMethod(ageGetter);
            }

            return cc.toBytecode();
        }
        catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}
```





