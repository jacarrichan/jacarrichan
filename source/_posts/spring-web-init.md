title: spring web的初始化
date: 2015-01-25 18:37:36
category: spring
tags: [springMVC,spring]
---


 主要是想探究springMVC中引用ContextLoaderListener中配置的context:property-placeholder，在初始化springMVC的时候会提示“ Could not resolve placeholder 'jdbc_url' in string value "${jdbc_url}"”的原因

<!--more-->

根据servlet的规范可知初始化顺序：context-param -> listener -> filter -> servlet

# ContextLoaderListener的初始化
```
ContextLoaderListener(ContextLoader).initWebApplicationContext(ServletContext) line: 306	 ##下面会贴代码
ContextLoaderListener.contextInitialized(ServletContextEvent) line: 112	
```

## WebApplicationContext的初始化

```
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			//放入ServletContext中，等待DispatcherServlet初始化时取出来
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```


## DispatcherServlet的初始化

```
XmlWebApplicationContext.loadBeanDefinitions(XmlBeanDefinitionReader) line: 123	 ##下面会贴代码
XmlWebApplicationContext.loadBeanDefinitions(DefaultListableBeanFactory) line: 94	
XmlWebApplicationContext(AbstractRefreshableApplicationContext).refreshBeanFactory() line: 130	
XmlWebApplicationContext(AbstractApplicationContext).obtainFreshBeanFactory() line: 537	
XmlWebApplicationContext(AbstractApplicationContext).refresh() line: 451	 ##下面会贴代码
DispatcherServlet(FrameworkServlet).configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext) line: 651	
DispatcherServlet(FrameworkServlet).createWebApplicationContext(ApplicationContext) line: 602	
DispatcherServlet(FrameworkServlet).createWebApplicationContext(WebApplicationContext) line: 665	
DispatcherServlet(FrameworkServlet).initWebApplicationContext() line: 521	 ##会使用ContextLoaderListener放入的ApplicationContext
DispatcherServlet(FrameworkServlet).initServletBean() line: 462	
DispatcherServlet(HttpServletBean).init() line: 136	
DispatcherServlet(GenericServlet).init(ServletConfig) line: 244	
ServletHolder.initServlet() line: 532	
```


XmlWebApplicationContext从web.xml中DispatcherServlet的init参数contextConfigLocation得到configLocations。


XmlWebApplicationContext.loadBeanDefinitions方法

```
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}
```

XmlWebApplicationContext的refresh()方法

```
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();//从springMVC配置文件加载beanFactory

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				//BeanFactoryPostProcessor的实现类会在这里被处理，比如PropertyPlaceholderConfigurer，处理完成后就会被抛弃掉，不会放进ApplicationContext中
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory); 


				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);    

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
	}
```

# 结论
ApplicationContext初始化完成后，会把其余的配置信息抛弃掉

