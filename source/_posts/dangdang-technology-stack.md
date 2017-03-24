title: 当当网的技术堆栈
date: 2014-08-24 19:02:45
category: Exception
tags: [springMVC,tomcat,dangdang.com]
---

今日偶然间发现了当当网的一个[异常信息](http://e.dangdang.com/ebook/fastpay.do?productId=12324#)，推测出采用了springMVC、springSecurity和Apache  tomcat， 根据当当网的主页看，视乎门户采用的是ASP.NET的技术.
<!--more-->
```
System Runtime Error: 
productId=12324的书不存在


Back 
Administrator click here to get the detail.

java.lang.RuntimeException: productId=12324的书不存在
	at com.dangdang.cms.service.PayService.generateFastpayOrder(PayService.java:80)
	at com.dangdang.cms.web.EbookController.fastpay(EbookController.java:652)
	at sun.reflect.GeneratedMethodAccessor182.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
	at java.lang.reflect.Method.invoke(Method.java:597)
	at org.springframework.web.bind.annotation.support.HandlerMethodInvoker.invokeHandlerMethod(HandlerMethodInvoker.java:176)
	at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.invokeHandlerMethod(AnnotationMethodHandlerAdapter.java:436)
	at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.handle(AnnotationMethodHandlerAdapter.java:424)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:790)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:719)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:669)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:574)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:621)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:722)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:304)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:210)
	at com.dangdang.cms.mainsite.service.filter.CheckUserLoginStatusFilter.doFilterInternal(CheckUserLoginStatusFilter.java:96)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:76)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:243)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:210)
	at org.springframework.web.multipart.support.MultipartFilter.doFilterInternal(MultipartFilter.java:113)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:76)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:243)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:210)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:369)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.invoke(FilterSecurityInterceptor.java:109)
	at org.springframework.security.web.access.intercept.FilterSecurityInterceptor.doFilter(FilterSecurityInterceptor.java:83)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:97)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:100)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.authentication.AnonymousAuthenticationFilter.doFilter(AnonymousAuthenticationFilter.java:78)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.authentication.rememberme.RememberMeAuthenticationFilter.doFilter(RememberMeAuthenticationFilter.java:112)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter.doFilter(SecurityContextHolderAwareRequestFilter.java:54)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.savedrequest.RequestCacheAwareFilter.doFilter(RequestCacheAwareFilter.java:35)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:187)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:105)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.context.SecurityContextPersistenceFilter.doFilter(SecurityContextPersistenceFilter.java:79)
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:381)
	at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:168)
	at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:237)
	at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:167)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:243)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:210)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:88)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:76)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:243)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:210)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:240)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:164)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:462)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:164)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:100)
	at org.apache.catalina.valves.AccessLogValve.invoke(AccessLogValve.java:562)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:118)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:395)
	at org.apache.coyote.http11.Http11Processor.process(Http11Processor.java:250)
	at org.apache.coyote.http11.Http11Protocol$Http11ConnectionHandler.process(Http11Protocol.java:188)
	at org.apache.coyote.http11.Http11Protocol$Http11ConnectionHandler.process(Http11Protocol.java:166)
	at org.apache.tomcat.util.net.JIoEndpoint$SocketProcessor.run(JIoEndpoint.java:302)
	at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
	at java.lang.Thread.run(Thread.java:662)
```




