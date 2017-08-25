title: 下载大量数据导致tomcat出现OOME
date: 2017-03-24 12:01:10
tags: [tomcat,OOME,OutOfMemoryError,poi]
---


2月21日晚下班路上，等了6分钟公交车，车还在驶往下一站的路上接到同事电话：商户端系统无法打开，全国客服群炸窝子....




## 漫无目的的查找原因

赶回办公室，开电脑，登录VPN，登录跳板机，登录nginx：发现upstream连接全部被拒绝。然后便登录到其中一条业务机，无法ps查看tomcat了，因为已经被运维童鞋重启了；有同事提议查看是否有heapdump文件，去bin目录等各种可能存在的地方找找都没有发现。top查看CPU，发现资源占用也比较正常；查看tomcat日志最后几条日志，发现都是简单的Zookeeper心跳。然后开始纠结了，到底是哪种看不见的问题导致了商户系统无法正常打开的~~ 

## 找出罪魁祸首

过了几分钟，同事在群里发出了OOM的日志,根据同事贴出的这一信息，往上翻，发现最后使用最具怀疑的对象是**报表下载**功能。因最近上线一个功能（在logback日志中打印操作用户的id）找出了当时是哪个商户（全国最大的连锁火锅店）使用系统导致的。并且该用户在系统没有响应的情况下，**连续操作此功能，在配合nginx自动重试的情况下，导致8个tomcat实例全部挂了**。

> 使用nginx获取的OOME信息如下：

```
192.168.8.12 | success | rc=0 >>
java.lang.OutOfMemoryError: Java heap space
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'loginController' defined in file [/home/.../controllers/LoginController.class]: Initialization of bean failed; nested exception is java.lang.OutOfMemoryError: Java heap space
Caused by: java.lang.OutOfMemoryError: Java heap space
java.lang.Exception: java.lang.OutOfMemoryError: Java heap space
Caused by: java.lang.OutOfMemoryError: Java heap space

192.168.3.13 | success | rc=0 >>
java.lang.OutOfMemoryError: Java heap space
java.lang.Exception: java.lang.OutOfMemoryError: Java heap space
Caused by: java.lang.OutOfMemoryError: Java heap space

192.168.100.12 | success | rc=0 >>
java.lang.OutOfMemoryError: Java heap space
java.lang.Exception: java.lang.OutOfMemoryError: Java heap space
Caused by: java.lang.OutOfMemoryError: Java heap space
....

```
## 解决办法

当时提议来个紧急发版：把下载功能屏蔽，后来经讨论认为在当前用餐高峰期进行这样操作风险太大，于是否定此方案。

既然当时从技术上解决此问题，于是想办法从业务上避免：考虑到目前没有跟这家火锅店做活动，于是让运营同事把这个火锅店商户给屏蔽了，不让其下载报表。

第二天工作日，解决方案是**把报表下载功能单独提出来放在单独的tomcat中**，然后nginx单独路由。考虑到低版本的POI使用XSSF功能耗费的内存比较大，遂**把POI升级**，然后使用SXSSF.

## 总结

### 问题原因
低版本的POI做Excel导出时容易出现OOM，并且没有做tomcat隔离，nginx自动重试导致所有tomcat出现OOM；

### 解决方案
升级POI，tomcat隔离


##  吐槽
最后要吐槽下springMVC，一个违反设计模式6大原则里面**依赖倒置原则**的抽象类：org.springframework.web.servlet.view.document.AbstractExcelView

该类中一个抽象方法的签名为

```
protected abstract void buildExcelDocument(Map<String, Object> var1, HSSFWorkbook var2, HttpServletRequest var3, HttpServletResponse var4) throws Exception;

```

槽点其实是在方法的第二个参数的类型：HSSFWorkbook，该类实现了Workbook,*依赖倒置原则*已经说明了**抽象不依赖于细节，而细节应该依赖于抽象**，然而他没有使用接口，而是一个具体的类。导致我在升级poi的过程中，需要改每个重写这个方法的地方。

不过好在spring后来意识到这个问题，在[4.2](https://github.com/spring-projects/spring-framework/blob/4.2.x/spring-webmvc/src/main/java/org/springframework/web/servlet/view/document/AbstractXlsView.java 'AbstractXlsView.java')版本中，将参数改成了接口:

```
	/**
	 * Application-provided subclasses must implement this method to populate
	 * the Excel workbook document, given the model.
	 * @param model the model Map
	 * @param workbook the Excel workbook to populate
	 * @param request in case we need locale etc. Shouldn't look at attributes.
	 * @param response in case we need to set cookies. Shouldn't write to it.
	 */
	protected abstract void buildExcelDocument(
			Map<String, Object> model, Workbook workbook, HttpServletRequest request, HttpServletResponse response)
			throws Exception;
```
