### SpringBoot AutoConfigure 流程

自动配置的入口  是@SpringBootApplication 里面的@EnableAutoConfiguration注解

在@EnableAutoConfiguration注解里面import了AutoConfigurationImportSelector这个类

在SpringBoot refresh IOC容器调用BeanFactory的后置处理器ConfigurationClassPostProcessor的时候会调用AutoConfigurationImportSelector

的selectImports方法得到所有需要自动配置的类

```java
	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}

```

这个方法是自动配置的核心过程

再看其中的getCandidateConfigurations方法

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

这里有一个SpringFactoriesLoader，这本质是Spring将自动配置的类导入IOC容器的核心工具类，下面我会详细讲

SpringFactoryLoader有以下四个方法

```java
/**
*  如果是第一次调用，这个方法会将类路径下所有META-INF/spring.factories里的键值对与 classLoader绑*  保存下来保存在SpringFactoriesLoader 中的名为cache的Map里面
*/
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader)

/**
* 从cache里面查找factoryClass键对应的类全路径，有那么将其加载到内存中，注意没有实例化
*/
public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader);

/**
* 查找指定factoryClass的类对应的所有类全路径
*/
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader);


/**
*  将指定类实例化
*/
private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader)
```

现在再来理解 getCandidateConfigurations中的这段代码

```java
List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
```

getSpringFactoriesLoaderFactoryClass() 返回的是EnableAutoConfiguration.class 那么会将SpringBootAutoConfigure这个jar包下META-INF/spring.factories 里面EnableAutoConfiguration类全路径键对应的全部类全路径值全部加载到SpringLoaderFactory cache里面

### Spring Boot 内置Web容器创建启动流程

这里以Tomcat容器为例

内置Web容器的创建的入口是 ServletWebServerApplicationContext 的onRefresh()方法

其中调用了 createWebServer(); 方法

```java
	private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
            // 在这里创建了WebServer
			this.webServer = factory.getWebServer(getSelfInitializer());
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context",
						ex);
			}
		}
		initPropertySources();
	}
```

```java
// 创建了Tomcat容器	
@Override
	public WebServer getWebServer(ServletContextInitializer... initializers) {
		Tomcat tomcat = new Tomcat();
		File baseDir = (this.baseDirectory != null ? this.baseDirectory
				: createTempDir("tomcat"));
		tomcat.setBaseDir(baseDir.getAbsolutePath());
		Connector connector = new Connector(this.protocol);
		tomcat.getService().addConnector(connector);
		customizeConnector(connector);
		tomcat.setConnector(connector);
		tomcat.getHost().setAutoDeploy(false);
		configureEngine(tomcat.getEngine());
		for (Connector additionalConnector : this.additionalTomcatConnectors) {
			tomcat.getService().addConnector(additionalConnector);
		}
        // 在这里准备创建应用上下文
		prepareContext(tomcat.getHost(), initializers);
		return getTomcatWebServer(tomcat);
	}
```



```java
protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
   File documentRoot = getValidDocumentRoot();
    // 这个就是创建了应用上下文 它是继承于StandartContext
    // 下面都是在给它设置属性值，比如就在这里设置了它的ClassLoader
   TomcatEmbeddedContext context = new TomcatEmbeddedContext();
   if (documentRoot != null) {
      context.setResources(new LoaderHidingResourceRoot(context));
   }
   context.setName(getContextPath());
   context.setDisplayName(getDisplayName());
   context.setPath(getContextPath());
   File docBase = (documentRoot != null ? documentRoot
         : createTempDir("tomcat-docbase"));
   context.setDocBase(docBase.getAbsolutePath());
   context.addLifecycleListener(new FixContextListener());
   context.setParentClassLoader(
         this.resourceLoader != null ? this.resourceLoader.getClassLoader()
               : ClassUtils.getDefaultClassLoader());
   resetDefaultLocaleMapping(context);
   addLocaleMappings(context);
   context.setUseRelativeRedirects(false);
   configureTldSkipPatterns(context);
   WebappLoader loader = new WebappLoader(context.getParentClassLoader());
   loader.setLoaderClass(TomcatEmbeddedWebappClassLoader.class.getName());
   loader.setDelegate(true);
   context.setLoader(loader);
   if (isRegisterDefaultServlet()) {
      addDefaultServlet(context);
   }
   if (shouldRegisterJspServlet()) {
      addJspServlet(context);
      addJasperInitializer(context);
   }
   context.addLifecycleListener(new StaticResourceConfigurer(context));
   ServletContextInitializer[] initializersToUse = mergeInitializers(initializers);
   host.addChild(context);
   configureContext(context, initializersToUse);
   postProcessContext(context);
}
```

上面的步骤就将WebServer创建完成，并且已经将该应用对应的上下文已经准备好

内置Web容器启动入口在 ServletWebServerApplicationContext 的finishRefresh()

```java
	private WebServer startWebServer() {
		WebServer webServer = this.webServer;
		if (webServer != null) {
			webServer.start();
		}
		return webServer;
	}
```

```java
@Override
public void start() throws WebServerException {
   synchronized (this.monitor) {
      if (this.started) {
         return;
      }
      try {
         addPreviouslyRemovedConnectors();
         Connector connector = this.tomcat.getConnector();
         if (connector != null && this.autoStart) {
             // 重点关注这个方法
            performDeferredLoadOnStartup();
         }
         checkThatConnectorsHaveStarted();
         this.started = true;
         TomcatWebServer.logger
               .info("Tomcat started on port(s): " + getPortsDescription(true)
                     + " with context path '" + getContextPath() + "'");
      }
      catch (ConnectorStartFailedException ex) {
         stopSilently();
         throw ex;
      }
      catch (Exception ex) {
         throw new WebServerException("Unable to start embedded Tomcat server",
               ex);
      }
      finally {
         Context context = findContext();
         ContextBindings.unbindClassLoader(context, context.getNamingToken(),
               getClass().getClassLoader());
      }
   }
}
```

```java
private void performDeferredLoadOnStartup() {
   try {
       // 取得内置tomcat 虚拟主机的应用上下文， 也就是刚刚创建的TomcatEmbeddedContext
      for (Container child : this.tomcat.getHost().findChildren()) {
         if (child instanceof TomcatEmbeddedContext) {
             //这里就是加载Context下的Servlet
            ((TomcatEmbeddedContext) child).deferredLoadOnStartup();
         }
      }
   }
   catch (Exception ex) {
      TomcatWebServer.logger.error("Cannot start connector: ", ex);
      throw new WebServerException("Unable to start embedded Tomcat connectors",
            ex);
   }
}
```

#### 

### Spring IOC

#### Spring IOC概述

1. 依赖反转：把控制权从具体的业务对象手中转交到平台或者框架中，Spring IOC就是实现这个模式的载体

2. 为什么要使用Spring IOC

   因为在面向对象程序设计中，一个对象会包含其它的对象，如果合伙对象的引用或依赖关系的管理有具体对象来完成，会导致代码的高度耦合和可测试性降低；如果把对象的管理和对象的注入都	交给Spring IOC来处理，那么解耦代码的同时还提高了可测试性

#### GenericApplicationContext的类图

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftluvkk5idj210j0bh765.jpg)

#### IOC容器的初始化过程

- 这里就会涉及到一个方法refresh，下面是它的源码：

  ```java
  	    public void refresh() throws BeansException, IllegalStateException {
          Object var1 = this.startupShutdownMonitor;
          synchronized(this.startupShutdownMonitor) {
              //初始化IOC容器之前的准备工作
              this.prepareRefresh();
              //得到一个BeanFactory,并且调用refreshBeanFactory(),这个方法会进行Bean的定位 、载入和
              //注册；但是在我看AnnotationConfigApplicationContext源码的时候，我没有发现它的载入和注
              //册
              ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
              //给BeanFactory设置一些属性
              this.prepareBeanFactory(beanFactory);
              try {
                  //设置BeanFactory的后置处理
                  this.postProcessBeanFactory(beanFactory);
                  //调用BeanFactory的后置处理器，这些处理器是在IOC容器中通过Bean注册的
                  this.invokeBeanFactoryPostProcessors(beanFactory);
                  //注册Bean的后处理器，在Bena的创建过程中调用
                  this.registerBeanPostProcessors(beanFactory);
                  //对上下文中的消息源进行初始化
                  this.initMessageSource();
                  //初始化上下文中的事件机制
                  this.initApplicationEventMulticaster();
                  //初始化其它特殊的Bean
                  this.onRefresh();
                  //检查监听Bean并且将这些Bean向容器注册
                  this.registerListeners();
                  //实例化所有的单件
                  this.finishBeanFactoryInitialization(beanFactory);
                  //发布容器事件，结束refresh过程
                  this.finishRefresh();
              } catch (BeansException var9) {
                  if (this.logger.isWarnEnabled()) {
                      this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                  }
                  //为防止Bean资源占用，在异常处理中销毁已经创建的单件Bean
                  this.destroyBeans();
                  //重置'active'标志
                  this.cancelRefresh(var9);
                  throw var9;
              } finally {
                  this.resetCommonCaches();
              }
  
          }
      }
  ```

  这个方法涉及到了BeanDefinition的定位、载入和注册三个基本过程

  1. 载入过程

     Resource

     Resource 是一个对加载对象的描述，被加载对象里面封装了Bean的定义；这里使用了策略模式，不同Resource 可以使用不同的Resource的实现

     ![](https://user-gold-cdn.xitu.io/2018/3/22/1624b9dd0a2d5dee?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

     ResourceLoader

     ResourceLoader 也有不同的实现，所以也是使用了策略模式，在不同情况下使用不同的ResourceLoader ;下面是AbstractApplicationContext 实现的ResourcePatternResolver的方法，实现了加载Resource

     ```java
     @Override
     	public Resource[] getResources(String locationPattern) throws IOException {
     		return this.resourcePatternResolver.getResources(locationPattern);
     	}
     ```

     ![](https://user-gold-cdn.xitu.io/2018/3/22/1624ba7bd56e8ed7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

     BeanDefinition

     BeanDefinition 在XMl中对应了<bean></bean>里面的内容，`<bean>`元素标签拥有`class`、`scope`、`lazy-init`等配置属性，`BeanDefinition`则提供了相应的`beanClass`、`scope`、`lazyInit`属性，封装了对该对象实例化的时候的处理

     2. 实例化

        ```java
        try {
        				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        				checkMergedBeanDefinition(mbd, beanName, args);
        
        				// Guarantee initialization of beans that the current bean depends on.
        				String[] dependsOn = mbd.getDependsOn();
        				if (dependsOn != null) {
        					for (String dep : dependsOn) {
        						if (isDependent(beanName, dep)) {
        							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
        									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
        						}
        						registerDependentBean(dep, beanName);
        						try {
        							getBean(dep);
        						}
        						catch (NoSuchBeanDefinitionException ex) {
        							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
        									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
        						}
        					}
        				}
        ```

        1. 得到beanName 对应的RootBeanDefinition
        2. 取得RootBeanDefinition 对应的所有依赖
        3. 检查是否有自循环依赖
        4. 递归将RootBeanDefinition 依赖对象实例化到IOC 容器中

     3. 通过 populate 方法自动注入需要自动注入的对象

  #### IOC容器的依赖注入

- 涉及到的源码

  ```java
  // AbstractBeanFactory  
  public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
          return this.doGetBean(name, requiredType, (Object[])null, false);
      }
  
      protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
          String beanName = this.transformedBeanName(name);
          //从缓存中得到对象，避免重复创建
          Object sharedInstance = this.getSingleton(beanName);
          Object bean;
          if (sharedInstance != null && args == null) {
              if (this.logger.isDebugEnabled()) {
                  if (this.isSingletonCurrentlyInCreation(beanName)) {
                      this.logger.debug("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
                  } else {
                      this.logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                  }
              }
              //因为IOC中有FactoryBean和普通Bean两种，所以这里就是如果这个Bean是FactoryBean，
              //就取得它的真实对象
              bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
              //下面就是从父级容器里面得到Bean
          } else {
              if (this.isPrototypeCurrentlyInCreation(beanName)) {
                  throw new BeanCurrentlyInCreationException(beanName);
              }
  
              BeanFactory parentBeanFactory = this.getParentBeanFactory();
              if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                  String nameToLookup = this.originalBeanName(name);
                  if (parentBeanFactory instanceof AbstractBeanFactory) {
                      return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
                  }
  
                  if (args != null) {
                      return parentBeanFactory.getBean(nameToLookup, args);
                  }
  
                  return parentBeanFactory.getBean(nameToLookup, requiredType);
              }
  
              if (!typeCheckOnly) {
                  this.markBeanAsCreated(beanName);
              }
              //处理依赖，创建Bean
              try {
                  RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
                  this.checkMergedBeanDefinition(mbd, beanName, args);
                  String[] dependsOn = mbd.getDependsOn();
                  String[] var11;
                  if (dependsOn != null) {
                      var11 = dependsOn;
                      int var12 = dependsOn.length;
  
                      for(int var13 = 0; var13 < var12; ++var13) {
                          String dep = var11[var13];
                          if (this.isDependent(beanName, dep)) {
                              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                          }
  
                          this.registerDependentBean(dep, beanName);
  
                          try {
                              this.getBean(dep);
                          } catch (NoSuchBeanDefinitionException var24) {
                              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "'" + beanName + "' depends on missing bean '" + dep + "'", var24);
                          }
                      }
                  }
  
                  if (mbd.isSingleton()) {
                      sharedInstance = this.getSingleton(beanName, () -> {
                          try {
                              return this.createBean(beanName, mbd, args);
                          } catch (BeansException var5) {
                              this.destroySingleton(beanName);
                              throw var5;
                          }
                      });
                      bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                  } else if (mbd.isPrototype()) {
                      var11 = null;
  
                      Object prototypeInstance;
                      try {
                          this.beforePrototypeCreation(beanName);
                          prototypeInstance = this.createBean(beanName, mbd, args);
                      } finally {
                          this.afterPrototypeCreation(beanName);
                      }
  
                      bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                  } else {
                      String scopeName = mbd.getScope();
                      Scope scope = (Scope)this.scopes.get(scopeName);
                      if (scope == null) {
                          throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                      }
  
                      try {
                          Object scopedInstance = scope.get(beanName, () -> {
                              this.beforePrototypeCreation(beanName);
  
                              Object var4;
                              try {
                                  var4 = this.createBean(beanName, mbd, args);
                              } finally {
                                  this.afterPrototypeCreation(beanName);
                              }
  
                              return var4;
                          });
                          bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                      } catch (IllegalStateException var23) {
                          throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var23);
                      }
                  }
              } catch (BeansException var26) {
                  this.cleanupAfterBeanCreationFailure(beanName);
                  throw var26;
              }
          }
  
          if (requiredType != null && !requiredType.isInstance(bean)) {
              try {
                  T convertedBean = this.getTypeConverter().convertIfNecessary(bean, requiredType);
                  if (convertedBean == null) {
                      throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                  } else {
                      return convertedBean;
                  }
              } catch (TypeMismatchException var25) {
                  if (this.logger.isDebugEnabled()) {
                      this.logger.debug("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var25);
                  }
  
                  throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
              }
          } else {
              return bean;
          }
      }
  ```

  doGetBean的流程图:

  ![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftlwn5l3cij20db0lywfp.jpg)

  ```java
   protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
          BeanWrapper instanceWrapper = null;
          if (mbd.isSingleton()) {
              instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
          }
  
          if (instanceWrapper == null) {
              instanceWrapper = this.createBeanInstance(beanName, mbd, args);
          }
  
          Object bean = instanceWrapper.getWrappedInstance();
          Class<?> beanType = instanceWrapper.getWrappedClass();
          if (beanType != NullBean.class) {
              mbd.resolvedTargetType = beanType;
          }
  
          Object var7 = mbd.postProcessingLock;
          synchronized(mbd.postProcessingLock) {
              if (!mbd.postProcessed) {
                  try {
                      this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                  } catch (Throwable var17) {
                      throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                  }
  
                  mbd.postProcessed = true;
              }
          }
  
          boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
          if (earlySingletonExposure) {
              if (this.logger.isDebugEnabled()) {
                  this.logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
              }
  
              this.addSingletonFactory(beanName, () -> {
                  return this.getEarlyBeanReference(beanName, mbd, bean);
              });
          }
  
          Object exposedObject = bean;
  
          try {
              this.populateBean(beanName, mbd, instanceWrapper);
              exposedObject = this.initializeBean(beanName, exposedObject, mbd);
          } catch (Throwable var18) {
              if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                  throw (BeanCreationException)var18;
              }
  
              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
          }
  
          if (earlySingletonExposure) {
              Object earlySingletonReference = this.getSingleton(beanName, false);
              if (earlySingletonReference != null) {
                  if (exposedObject == bean) {
                      exposedObject = earlySingletonReference;
                  } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                      String[] dependentBeans = this.getDependentBeans(beanName);
                      Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                      String[] var12 = dependentBeans;
                      int var13 = dependentBeans.length;
  
                      for(int var14 = 0; var14 < var13; ++var14) {
                          String dependentBean = var12[var14];
                          if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                              actualDependentBeans.add(dependentBean);
                          }
                      }
  
                      if (!actualDependentBeans.isEmpty()) {
                          throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                      }
                  }
              }
          }
  
          try {
              this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
              return exposedObject;
          } catch (BeanDefinitionValidationException var16) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
          }
      }
  ```

  这里有个BeanWapper，这个BeanWapper的作用是管理Bean的属性

#### 容器相关特性的设计与实现

##### ApplicationContext和Bean的初始化和销毁

ApplicationContext初始化:prepareBeanFactory()

ApplicationContext的销毁：doClose();

容器的实现是通过IOC管理Bean生命周期来实现的：

initailizeBean；doClose；destroy

##### lazy-init属性和预实例化

##### FactoryBean的实现

其实在我看来FactoryBean其实就是Spring 帮我们封装一个简化的工厂方法，我们可以把这个FactoryBean放在IOC容器里面，然后通过这个工厂类生成我们想要的类

主要涉及到的源码：

```java
protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
        if (BeanFactoryUtils.isFactoryDereference(name)) {
            if (beanInstance instanceof NullBean) {
                return beanInstance;
            }

            if (!(beanInstance instanceof FactoryBean)) {
                throw new BeanIsNotAFactoryException(this.transformedBeanName(name), beanInstance.getClass());
            }
        }

        if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
            Object object = null;
            if (mbd == null) {
                object = this.getCachedObjectForFactoryBean(beanName);
            }

            if (object == null) {
                FactoryBean<?> factory = (FactoryBean)beanInstance;
                if (mbd == null && this.containsBeanDefinition(beanName)) {
                    mbd = this.getMergedLocalBeanDefinition(beanName);
                }

                boolean synthetic = mbd != null && mbd.isSynthetic();
                object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
            }

            return object;
        } else {
            return beanInstance;
        }
    }
```

```java
    protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        if (factory.isSingleton() && this.containsSingleton(beanName)) {
            synchronized(this.getSingletonMutex()) {
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    object = this.doGetObjectFromFactoryBean(factory, beanName);
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    } else {
                        if (shouldPostProcess) {
                            if (this.isSingletonCurrentlyInCreation(beanName)) {
                                return object;
                            }

                            this.beforeSingletonCreation(beanName);

                            try {
                                object = this.postProcessObjectFromFactoryBean(object, beanName);
                            } catch (Throwable var14) {
                                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var14);
                            } finally {
                                this.afterSingletonCreation(beanName);
                            }
                        }

                        if (this.containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object);
                        }
                    }
                }

                return object;
            }
        } else {
            Object object = this.doGetObjectFromFactoryBean(factory, beanName);
            if (shouldPostProcess) {
                try {
                    object = this.postProcessObjectFromFactoryBean(object, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", var17);
                }
            }

            return object;
        }
    }
```

上面两个方法解释了从FactoryBean取得真实对象

##### BeanPostProcessor的实现

这个我们看源码的initializeBean方法就行

```java
    protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(() -> {
                this.invokeAwareMethods(beanName, bean);
                return null;
            }, this.getAccessControlContext());
        } else {
            this.invokeAwareMethods(beanName, bean);
        }
        //得到的是一个BeanWapper
        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            //这里就会一次调用所有Bean创建的前置处理器
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            //初始化Bean
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            //调用所有Bean创建的后置处理器
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
        return wrappedBean;
    }
```

这个方法会在getBean ->doGetBean() ->createBean-> doCreateBean中调用		

##### autowiring 的实现

```java
                if (mbd.getResolvedAutowireMode() == 1 || mbd.getResolvedAutowireMode() == 2) {
                    MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
                    if (mbd.getResolvedAutowireMode() == 1) {
                        this.autowireByName(beanName, mbd, bw, newPvs);
                    }

                    if (mbd.getResolvedAutowireMode() == 2) {
                        this.autowireByType(beanName, mbd, bw, newPvs);
                    }

                    pvs = newPvs;
                }
```

这是自动装配的实现，在AbstractAutowireCapableBeanFactory的populateBean方法中，这个populateBean 会在调用doCreateBean中调用

#### Bean的生命周期流程

![](https://images2017.cnblogs.com/blog/584866/201710/584866-20171026101746488-698711766.png)

在这个方法里面，有许多用户自定义

这个方法的调用链是：getBean -> doGetBean ->doCreateBean ->initializeBean

```java
//AbstractAutowireCapableBeanFactory
	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
            // 实现有关Aware接口的方法
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
        // 调用Bean的前置处理器
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
            // 调用init方法，如果class 实现了InitializingBean接口，还会调用afterPropertiesSet方法；这两种方式都可以达到初始化实例的目的
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            //调用Bean的后置处理器
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

#### 循环依赖问题

构造器注入不能解决循环依赖，如果Bean的作用域为Prototype 也是不能解决循环依赖的，因为Spring中没有提前暴露对象的引用，从下面的代码中体现

```
// AbstractBeanFactory
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
     if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
	 }
}
```

- 怎么检测Prototype产生循环依赖的

  ```java
  	protected boolean isPrototypeCurrentlyInCreation(String beanName) {
  		Object curVal = this.prototypesCurrentlyInCreation.get();
  		return (curVal != null &&
  				(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
  	}
  
  ```

  prototypesCurrentlyInCreation 是一个ThreadLocal，避免添加的竞争

在SpringBoot中不存在构造器注入所以循环依赖可以解决

下面就讲一下是怎么解决的循环依赖

- Spring中，使用了三级缓存解决循环依赖

```java
// 单例对象的cache
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

//单例对象工厂的cache 
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

//提前暴光的单例对象的Cache，检测是否存在循环依赖
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

下面就是具体的解决代码：

这段代码来自DefaultSingletonBeanRegistry

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 首先从一级缓存里面获取
   Object singletonObject = this.singletonObjects.get(beanName);
    // 一级缓存取不到，并且正在创建，说明存在循环依赖
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
          // 从二级缓存获取
         singletonObject = this.earlySingletonObjects.get(beanName);
          // 二级缓存还没有并且允许提前获得其引用
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
                // 将存在循环依赖的对象放到earlySingletonObjects中
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

这段代码来自AbstractBeanFactory

```java
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
// 判断是否存在循环依赖
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
   if (logger.isDebugEnabled()) {
      logger.debug("Eagerly caching bean '" + beearlySingletonObjectsanName +
            "' to allow for resolving potential circular references");
   }
    // 提供存在循环依赖对象的对象工厂
   addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

- 其实这里存在一个问题：为什么singletonObjects 使用的是ConcurrentHashMap 而二级和三级缓存只使用了HashMap

  因为对于earlySingletonObjects、singletonFactories的操作我们需要保证它们的一致性，也就是如果添加到earlySingletonObjects 元素就要相应的删除singletonFactories中的一个元素（你可以在源码下发现这两个操作总是配对的出现），所以必须用sychronized来保证这个操作的原子性；同时sychronized 也就保证了两个hashMap在 并发下的安全

#### FactoryBean实现

这个我们了解很少，但是它的作用其实很大，它的主要思想就如它的名字一样 相当于向IOC容器里面放了一个工厂方法用于生成实例；比如说Spring自己实现的AOP就是用这个技术来生成代理对象

下面是AbstractBeanFactory 的doGetBean方法的部分

~~~java
String beanName = this.transform
// 从IOC容器中取可能存在的实例；但是需要注意的是这里取出来得实例有可能是FactoryBean
        Object sharedInstance = this.getSingleton(beanName);
        Object bean;
        if (sharedInstance != null && args == null) {
            if (this.logger.isTraceEnabled()) {
                if (this.isSingletonCurrentlyInCreation(beanName)) {
                    this.logger.trace("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
                } else {
                    this.logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            // 所以这个方法解决了不管是取出来的是什么对象，我们都可以得到我们想要的实例
            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        }
        ``````
        ``````
        ``````
~~~

getObjectForBeanInstance 方法

```java
protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }

        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(this.transformedBeanName(name), beanInstance.getClass());
        }
    }

    if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
        Object object = null;
        if (mbd == null) {
            // FactoryBean生成的Bean也有缓存
            object = this.getCachedObjectForFactoryBean(beanName);
        }

        if (object == null) {
            FactoryBean<?> factory = (FactoryBean)beanInstance;
            if (mbd == null && this.containsBeanDefinition(beanName)) {
                mbd = this.getMergedLocalBeanDefinition(beanName);
            }

            boolean synthetic = mbd != null && mbd.isSynthetic();
            // 从FactoryBean 构造Bean并放进其缓存
            object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
        }

        return object;
    } else {
        // 如果不是FactoryBean直接返回原对象
        return beanInstance;
    }
}
```

#### Bean的作用域

下面是体现Bean作用域的代码

```java
	//AbstractBeanFactory
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
			......
			if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}
		......
			}
```

从源码我们可以看到 对于Scope的调用Spring使用了策略模式，解耦了对Scope的依赖

![](https://img-blog.csdn.net/20160417164310654?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### Spring AOP

在Spring 中其实存在两种AOP方式，一种是Spring自己实现的AOP，还有一种就是使用的AspectJ的方式；在以前用xml配置的时候用Spring自己实现AOP比较多，但是现在更喜欢注解的方式去实现AOP，注解的方式其实就是用的AspectJ的方式

AspectJ 的方式我没有细探，但是大概知道其是通过运行时代码织入的形式修改原来方法的代码来实现的

下面我就细讲一下Spring AOP实现的原理

在弄懂Spring AOP之前要去看下上文的FactoryBean的实现，因为Spring AOP是在其上实现的

Spring AOP 的入口是ProxyFactoryBean 这个类， 看了上文的FactoryBean实现 你也就知道了如果想取得被扩展了的类其实就是从IOC里面取ProxyFactoryBean ， 只不过在IOC里面会把它转化成扩展的类而已

#### Spring AOP 概述

业务逻辑的代码中不再含有针对特定领域问题代码的调用，业务逻辑同特定领域问题的关系通过切面来封装和维护，这样原本分散在整个应用程序中的变动就可以很好的管理起来

- 基础：视为待增强对象或者说目标对象；
- 切面：通常包含对于基础的增强应用；
- 配置：可以看成是一种编织，把基础和切面结合起来从而实现切面对目标对象的编织实现

在Spring AOP中有三个与上面对应：

- Advice通知：定义在切入点做什么，为切面增强提供织入接口，相当于上面的切面
- Pointcut切点：定义通知应该作用于哪个连接点，相当于上面的基础
- Advisor通知器：将切面和连接点结合起来的设计，相当于上面的配置

#### 增强对象生成的过程

首先给出AOP应用相关类的继承关系图：

![](https://ws1.sinaimg.cn/large/a67bf22fgy1fts57tjs3uj20op0dz0t2.jpg)

我们先以ProxyFactoryBean来讲解Aop

1. 配置ProxyFactoryBean

   ```xml
   <bean id="testAdvisor" class="comabc.TestAdvisor"/>
   <bean id="testAop" class="org.springframework.aop.ProxyFactoryBean">
   <property name="proxyInterfaces"><value>com.test.AbcInterface</value></property>
   <property name="target">
       <bean class="com.abc.TestTarget"/>
   </property>
   <property name="intercerptorNames">
       <list><value>testAdvisor</value></list>
   </property>
   </bean>
   ```

   - testAdvisor是定义切面
   - testAop定义ProxyFactoryBean，target属性是基础即待增强的类

2. ProxyFactoryBean**生成AopProxy代理对象**

   生成AopProxy代理对象的流程图

   ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fts84yqduyj20x50gvwfd.jpg)

   ```java
   public Object getObject() throws BeansException {
       //初始化通知器链
       this.initializeAdvisorChain();
       //这里是对singleton和prototype的类型进行区分，生成对应的proxy
       if (this.isSingleton()) {
           return this.getSingletonInstance();
       } else {
           if (this.targetName == null) {
               this.logger.warn("Using non-singleton proxies with singleton targets is often undesirable. Enable prototype proxies by setting the 'targetName' property.");
           }
   
           return this.newPrototypeInstance();
       }
   }
   
    private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
           if (!this.advisorChainInitialized) {
               if (!ObjectUtils.isEmpty(this.interceptorNames)) {
                   if (this.beanFactory == null) {
                       throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) - cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
                   }
   
                   if (this.interceptorNames[this.interceptorNames.length - 1].endsWith("*") && this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
                       throw new AopConfigException("Target required after globals");
                   }
   
                   String[] var1 = this.interceptorNames;
                   int var2 = var1.length;
                   //这里是添加Advisor链的调用，是通过interceprotNames属性进行设置的
                   for(int var3 = 0; var3 < var2; ++var3) {
                       String name = var1[var3];
                       if (this.logger.isTraceEnabled()) {
                           this.logger.trace("Configuring advisor or advice '" + name + "'");
                       }
   
                       if (name.endsWith("*")) {
                           if (!(this.beanFactory instanceof ListableBeanFactory)) {
                               throw new AopConfigException("Can only use global advisors or interceptors with a ListableBeanFactory");
                           }
   
                           this.addGlobalAdvisor((ListableBeanFactory)this.beanFactory, name.substring(0, name.length() - "*".length()));
                       } else {
                           Object advice;
                           if (!this.singleton && !this.beanFactory.isSingleton(name)) {
                               advice = new ProxyFactoryBean.PrototypePlaceholderAdvisor(name);
                           } else {
                               advice = this.beanFactory.getBean(name);
                           }
   
                           this.addAdvisorOnChainCreation(advice, name);
                       }
                   }
               }
   
               this.advisorChainInitialized = true;
           }
       }
   
   private synchronized Object getSingletonInstance() {
           if (this.singletonInstance == null) {
               this.targetSource = this.freshTargetSource();
               if (this.autodetectInterfaces && this.getProxiedInterfaces().length == 0 && !this.isProxyTargetClass()) {
                   Class<?> targetClass = this.getTargetClass();
                   if (targetClass == null) {
                       throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
                   }
   
                   this.setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
               }
   
               super.setFrozen(this.freezeProxy);
               this.singletonInstance = this.getProxy(this.createAopProxy());
           }
   
           return this.singletonInstance;
       }
   
   
       protected final synchronized AopProxy createAopProxy() {
           if (!this.active) {
               this.activate();
           }
   
           return this.getAopProxyFactory().createAopProxy(this);
       }
   
       public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
           if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
               return new JdkDynamicAopProxy(config);
           } else {
               Class<?> targetClass = config.getTargetClass();
               if (targetClass == null) {
                   throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
               } else {
                   //如果有接口使用JDK动态代理，没有接口使用CGLIB
                   return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
               }
           }
       }
   ```

3. Spring Aop拦截器调用实现

   我们就以JDK invoke方法讲解

   ```java
       @Nullable
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
           Object oldProxy = null;
           boolean setProxyContext = false;
           TargetSource targetSource = this.advised.targetSource;
           Object target = null;
   
           Boolean var9;
           try {
               if (this.equalsDefined || !AopUtils.isEqualsMethod(method)) {
                   //如果目标对象没有Object类的equals、hashCode方法
                   if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                       Integer var19 = this.hashCode();
                       return var19;
                   }
   
                   if (method.getDeclaringClass() == DecoratingProxy.class) {
                       Class var18 = AopProxyUtils.ultimateTargetClass(this.advised);
                       return var18;
                   }
   
                   Object retVal;
                   if (!this.advised.opaque && method.getDeclaringClass().isInterface() && method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                       //根据代理对象的配置(ProxyConfig)来调用服务
                       retVal = AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
                       return retVal;
                   }
   
                   if (this.advised.exposeProxy) {
                       oldProxy = AopContext.setCurrentProxy(proxy);
                       setProxyContext = true;
                   }
   
                   target = targetSource.getTarget();
                   Class<?> targetClass = target != null ? target.getClass() : null;
                   //获得经过过滤这个方法的所有拦截器
                   List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                   if (chain.isEmpty()) {
                       Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                       retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
                   } else {
                       //沿着拦截器链执行方法
                       MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                       retVal = invocation.proceed();
                   }
   
                   Class<?> returnType = method.getReturnType();
                   if (retVal != null && retVal == target && returnType != Object.class && returnType.isInstance(proxy) && !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                       retVal = proxy;
                   } else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                       throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
                   }
   
                   Object var13 = retVal;
                   return var13;
               }
   
               var9 = this.equals(args[0]);
           } finally {
               if (target != null && !targetSource.isStatic()) {
                   targetSource.releaseTarget(target);
               }
   
               if (setProxyContext) {
                   AopContext.setCurrentProxy(oldProxy);
               }
   
           }
   
           return var9;
       }
   ```

   - AOP 拦截器链的调用

     ```java
     
     public Object proceed() throws Throwable {
             if (this.currentInterceptorIndex ==this.interceptorsAndDynamicMethodMatchers.size() - 1) {
                 return this.invokeJoinpoint();
      } else {
                 Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
                 if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
                     InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
                     //如果通知器和调用的方法匹配，那么调用拦截器链的所有方法，否则继续匹配
                     return dm.methodMatcher.matches(this.method, this.targetClass, this.arguments) ? dm.interceptor.invoke(this) : this.proceed();
                 } else {
                     return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
                 }
             }
         }
     ```

- 拦截器链的产生

  ```java
   private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
          if (!this.advisorChainInitialized) {
              if (!ObjectUtils.isEmpty(this.interceptorNames)) {
                  if (this.beanFactory == null) {
                      throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) - cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
                  }
  
                  if (this.interceptorNames[this.interceptorNames.length - 1].endsWith("*") && this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
                      throw new AopConfigException("Target required after globals");
                  }
  
                  String[] var1 = this.interceptorNames;
                  int var2 = var1.length;
                  //这里是添加Advisor链的调用，是通过interceprotNames属性进行设置的
                  for(int var3 = 0; var3 < var2; ++var3) {
                      String name = var1[var3];
                      if (this.logger.isTraceEnabled()) {
                          this.logger.trace("Configuring advisor or advice '" + name + "'");
                      }
  
                      if (name.endsWith("*")) {
                          if (!(this.beanFactory instanceof ListableBeanFactory)) {
                              throw new AopConfigException("Can only use global advisors or interceptors with a ListableBeanFactory");
                          }
  
                          this.addGlobalAdvisor((ListableBeanFactory)this.beanFactory, name.substring(0, name.length() - "*".length()));
                      } else {
                          Object advice;
                          if (!this.singleton && !this.beanFactory.isSingleton(name)) {
                              advice = new ProxyFactoryBean.PrototypePlaceholderAdvisor(name);
                          } else {
                              //从IOC容器中获得通知器或者切面，因为在实现中不管是通知器还是切面都会最终
                              //变成通知器Advisor
                              advice = this.beanFactory.getBean(name);
                          }
                          //添加到通知器中
                          this.addAdvisorOnChainCreation(advice, name);
                      }
                  }
              }
              this.advisorChainInitialized = true;
          }
      }
  
  //这个完成了从所有的通知器中得到拦截器链
  public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class<?> targetClass) {
          List<Object> interceptorList = new ArrayList(config.getAdvisors().length);
          Class<?> actualClass = targetClass != null ? targetClass : method.getDeclaringClass();
          boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
          AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
      //从config中得到实现从xml中获得的所有通知器
          Advisor[] var8 = config.getAdvisors();
          int var9 = var8.length;
  
          for(int var10 = 0; var10 < var9; ++var10) {
              Advisor advisor = var8[var10];
              //拦截器链
              MethodInterceptor[] interceptors;
              if (advisor instanceof PointcutAdvisor) {
                  PointcutAdvisor pointcutAdvisor = (PointcutAdvisor)advisor;
                  if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                      //全局的注册工厂获得一个通知器的拦截器(这里是同注册工厂里面的适配器将所有的)
                      //advisor变成特定类型的拦截器(MethodBeforeAdvice、AfterReturningAdvice等等)
                      interceptors = registry.getInterceptors(advisor);
                      //如果编码实现通知器，通知器中会实现切入点，只不过我们不怎么使用硬编码的形式实现
                      //切入点，所以下面再检查是否方法是否符合切入点
                      MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                      //下面是对所有的拦截器进行过滤，剩下这个方法的拦截器
                      if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
                          if (mm.isRuntime()) {
                              MethodInterceptor[] var15 = interceptors;
                              int var16 = interceptors.length;
  
                              for(int var17 = 0; var17 < var16; ++var17) {
                                  MethodInterceptor interceptor = var15[var17];
                                  //将所有方法加入拦截器链
                                  interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                              }
                          } else {
                              interceptorList.addAll(Arrays.asList(interceptors));
                          }
                      }
                  }
              } else if (advisor instanceof IntroductionAdvisor) {
                  IntroductionAdvisor ia = (IntroductionAdvisor)advisor;
                  if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                      interceptors = registry.getInterceptors(advisor);
                      interceptorList.addAll(Arrays.asList(interceptors));
                  }
              } else {
                  Interceptor[] interceptors = registry.getInterceptors(advisor);
                  interceptorList.addAll(Arrays.asList(interceptors));
              }
          }
  
          return interceptorList;
      }
  ```

  - 方法增强的实现

    ```java
    //这个方法的意图是借助DefaultAdvisorAdapterRegistry的适配器将所有的Advisor变成特定的interceptor
    //方便在调用拦截器链时的调用
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
            List<MethodInterceptor> interceptors = new ArrayList(3);
            Advice advice = advisor.getAdvice();
            if (advice instanceof MethodInterceptor) {
                interceptors.add((MethodInterceptor)advice);
            }
    
            Iterator var4 = this.adapters.iterator();
    
        //根据所有的适配器来适配得到拦截器
            while(var4.hasNext()) {
                AdvisorAdapter adapter = (AdvisorAdapter)var4.next();
                if (adapter.supportsAdvice(advice)) {
                    interceptors.add(adapter.getInterceptor(advisor));
                }
            }
    
            if (interceptors.isEmpty()) {
                throw new UnknownAdviceTypeException(advisor.getAdvice());
            } else {
                return (MethodInterceptor[])interceptors.toArray(new MethodInterceptor[0]);
            }
        }
    ```

    ```java
    //我们就分析这个adapter，supportsAdvice用来适配判断能够通过这个适配器来得到特定的拦截器；
    //getInterceptor就是用来得到特定的拦截器
    class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {
        MethodBeforeAdviceAdapter() {
        }
    
        public boolean supportsAdvice(Advice advice) {
            return advice instanceof MethodBeforeAdvice;
        }
    
        public MethodInterceptor getInterceptor(Advisor advisor) {
            MethodBeforeAdvice advice = (MethodBeforeAdvice)advisor.getAdvice();
            return new MethodBeforeAdviceInterceptor(advice);
        }
    }
    ```

    ```java
    //这个就是一种拦截器，通过invoke方法，我们也可以知道，其实就是在调用方法之前先执行了增强的内容
    public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
        private MethodBeforeAdvice advice;
    
        public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
            Assert.notNull(advice, "Advice must not be null");
            this.advice = advice;
        }
    
        public Object invoke(MethodInvocation mi) throws Throwable {
            this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
            return mi.proceed();
        }
    }
    
    ```

#### 大概讲述一个Spring AOP实现的过程

Spring AOP 是通过拦截器的形式实现的

1. 初始化得到一个拦截器链：这里涉及到适配器的设计模式，将所有注册的Advisor 适配成对应的MethodInterceptor 加入拦截器链
2. 通过动态代理生成AOPProxy 代理对象
   1. 其中invoke方法 大致实现为 将本方法加入到拦截器链中对应位置，然后通过一个类似于FilterChain 的责任链模式，根据index递归调用拦截器中的类的对应方法

###Spring MVC

#### 基于XML Spring MVC 源码详解

##### 关于web.xml

web.xml其实是对ServletContext的参数设置也就是Tomcat的环境设置，我觉得通俗点讲就是所有Servlet的上下文环境的设置，其实它也是Tomcat和Spring项目的耦合点，也就是说Tomcat启动后会去加载这个文件里面的内容；然后也是基于此内容Spring 会去创建一个WebApplicationContext也就是IOC容器，在这里具体的是XmlWebApplicationContext，下面会具体讲怎么创建的，也基于此在web.xml中有下面的一段配置：

```xml
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value></param-value>
    </context-param>
```

XmlWebApplicationContext 会将具体路径的配置加载到IOC容器中

##### 关于ContextLoaderListener

这个是具体Tomcat和Spring的耦合点，通过这个监听器Tomcat在启动完成后传递ServletContext给Spring然后完成一系列Spring项目的启动

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }

    //Tomat会在启动后通过监听器调用这个方法
    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
    }

    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}

  public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
            throw new IllegalStateException("Cannot initialize context because there is already a root application context present - check whether you have multiple ContextLoader* definitions in your web.xml!");
        } else {
            Log logger = LogFactory.getLog(ContextLoader.class);
            servletContext.log("Initializing Spring root WebApplicationContext");
            if (logger.isInfoEnabled()) {
                logger.info("Root WebApplicationContext: initialization started");
            }

            long startTime = System.currentTimeMillis();

            try {
                if (this.context == null) {
                    //ContextLoader会去读ContextLoader.properties中的Context的类型来创建具体是什么
                    //Context
                    this.context = this.createWebApplicationContext(servletContext);
                }

                if (this.context instanceof ConfigurableWebApplicationContext) {
                    ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext)this.context;
                    if (!cwac.isActive()) {
                        if (cwac.getParent() == null) {
                            //这里是NULL，也就是说着个容器已经是顶级容器
                            ApplicationContext parent = this.loadParentContext(servletContext);
                            cwac.setParent(parent);
                        }
                        //设置和初始化该ROOT IOC 容器
                        this.configureAndRefreshWebApplicationContext(cwac, servletContext);
                    }
                }

                servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
                ClassLoader ccl = Thread.currentThread().getContextClassLoader();
                if (ccl == ContextLoader.class.getClassLoader()) {
                    currentContext = this.context;
                } else if (ccl != null) {
                    currentContextPerThread.put(ccl, this.context);
                }

                if (logger.isDebugEnabled()) {
                    logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" + WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
                }

                if (logger.isInfoEnabled()) {
                    long elapsedTime = System.currentTimeMillis() - startTime;
                    logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
                }

                return this.context;
            } catch (RuntimeException var8) {
                logger.error("Context initialization failed", var8);
                servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, var8);
                throw var8;
            } catch (Error var9) {
                logger.error("Context initialization failed", var9);
                servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, var9);
                throw var9;
            }
        }
    }

```

```java
   protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
        String configLocationParam;
        if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
            configLocationParam = sc.getInitParameter("contextId");
            if (configLocationParam != null) {
                wac.setId(configLocationParam);
            } else {
                wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX + ObjectUtils.getDisplayString(sc.getContextPath()));
            }
        }

        wac.setServletContext(sc);
       //通过ServletContext读取web.xml文件中设置的contextConfigLocation并把它赋值给wac
       //方便在后IOC容器的加载
        configLocationParam = sc.getInitParameter("contextConfigLocation");
        if (configLocationParam != null) {
            wac.setConfigLocation(configLocationParam);
        }

        ConfigurableEnvironment env = wac.getEnvironment();
        if (env instanceof ConfigurableWebEnvironment) {
            ((ConfigurableWebEnvironment)env).initPropertySources(sc, (ServletConfig)null);
        }

        this.customizeContext(sc, wac);
       //IOC 容器启动啦
        wac.refresh();
    }
```

##### 关于DispatchServlet

DispatchServlet因为其自身本来就是一个Servlet所以它的初始化的完成时依赖于Servlet的init方法	

下面关于它的继承关系图

![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu63tj7rxyj20ob0do74r.jpg)

```java
public final void init() throws ServletException {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Initializing servlet '" + this.getServletName() + "'");
        }

        PropertyValues pvs = new HttpServletBean.ServletConfigPropertyValues(this.getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
            try {
                BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                ResourceLoader resourceLoader = new ServletContextResourceLoader(this.getServletContext());
                bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, this.getEnvironment()));
                
                this.initBeanWrapper(bw);
                bw.setPropertyValues(pvs, true);
            } catch (BeansException var4) {
                if (this.logger.isErrorEnabled()) {
                    this.logger.error("Failed to set bean properties on servlet '" + this.getServletName() + "'", var4);
                }

                throw var4;
            }
        }
        //初始化FrameworkServlet中的属性，其中就初始化了IOC容器
        this.initServletBean();
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Servlet '" + this.getServletName() + "' configured successfully");
        }

    }
```

```java
//在DispatchServlet初始化IOC容器过程中有完成了对基础设施的初始化  
protected void onRefresh(ApplicationContext context) {
        this.initStrategies(context);
    }

    protected void initStrategies(ApplicationContext context) {
        this.initMultipartResolver(context);
        this.initLocaleResolver(context);
        this.initThemeResolver(context);
        this.initHandlerMappings(context);
        this.initHandlerAdapters(context);
        this.initHandlerExceptionResolvers(context);
        this.initRequestToViewNameTranslator(context);
        this.initViewResolvers(context);
        this.initFlashMapManager(context);
    }
```

#### SpringMVC 的特性

##### 监听器

1. 基本使用方式

   1. 创建继承于ApplicationEvent的事件对象 
   2. 创建实现ApplicationListener的监听器 并且形式参数为创建的Event
   3. 发布事件

   ```java
   public class MyTestEvent extends ApplicationEvent{
       /**
        * 
        */
       private static final long serialVersionUID = 1L;
       
       private String msg ;
   
       public MyTestEvent(Object source,String msg) {
           super(source);
           this.msg = msg;
       }
   
       public String getMsg() {
           return msg;
       }
   
       public void setMsg(String msg) {
           this.msg = msg;
       }
   
   }
   
   @Component
   public class MyNoAnnotationListener implements ApplicationListener<MyTestEvent>{
   
       @Override
       public void onApplicationEvent(MyTestEvent event) {
           System.out.println("监听器：" + event.getMsg());
       }
   
   }
   
   @Component
   public class MyTestEventPubLisher {
       @Autowired
       private ApplicationContext applicationContext;
   
       // 事件发布方法
       public void pushListener(String msg) {
           applicationContext.publishEvent(new MyTestEvent(this, msg));
       }
   
   }
   
   
   ```

2. 监听器的源码实现

```java
//AbstractApplicaionContext下的
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
        Assert.notNull(event, "Event must not be null");
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Publishing event in " + this.getDisplayName() + ": " + event);
        }

        Object applicationEvent;
        if (event instanceof ApplicationEvent) {
            applicationEvent = (ApplicationEvent)event;
        } else {
            applicationEvent = new PayloadApplicationEvent(this, event);
            if (eventType == null) {
                eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
            }
        }
     //如果有事件没有处理完，就加入earlyApplicationEvents里面等待被处理
        if (this.earlyApplicationEvents != null) {
            this.earlyApplicationEvents.add(applicationEvent);
        } else {
           //真正开始处理事件
           this.getApplicationEventMulticaster().multicastEvent((ApplicationEvent)applicationEvent, eventType);
        }

        if (this.parent != null) {
            if (this.parent instanceof AbstractApplicationContext) {
                ((AbstractApplicationContext)this.parent).publishEvent(event, eventType);
            } else {
                this.parent.publishEvent(event);
            }
        }
    }

//SimpleApplicationEventMulticaster
   public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
       //这里取得了所有注册的相关类型的事件，至于怎么得到的，我这里简单的说一下；AbstractApplicationEventMulticaster.ListenerRetriever 这个类就是专门用于存储所有的监听器的，而且在AbstractApplicationEventMulticaster这个类下会缓存最近取的相关key的所有监听器，所以就可以从这个类里面得到监听器了
        Iterator var4 = this.getApplicationListeners(event, type).iterator();

        while(var4.hasNext()) {
            ApplicationListener<?> listener = (ApplicationListener)var4.next();
            //如果有线程池，使用线程池完成所有的监听器里面的内容
            Executor executor = this.getTaskExecutor();
            if (executor != null) {
                executor.execute(() -> {
                    this.invokeListener(listener, event);
                });
            } else {
                this.invokeListener(listener, event);
            }
        }

    }

```

3. 与Tomcat 声明周期监听器实现的差别

   ```java
   public interface Lifecycle {
       String BEFORE_INIT_EVENT = "before_init";
       String AFTER_INIT_EVENT = "after_init";
       String START_EVENT = "start";
       String BEFORE_START_EVENT = "before_start";
       String AFTER_START_EVENT = "after_start";
       String STOP_EVENT = "stop";
       String BEFORE_STOP_EVENT = "before_stop";
       String AFTER_STOP_EVENT = "after_stop";
       String AFTER_DESTROY_EVENT = "after_destroy";
       String BEFORE_DESTROY_EVENT = "before_destroy";
       String PERIODIC_EVENT = "periodic";
       String CONFIGURE_START_EVENT = "configure_start";
       String CONFIGURE_STOP_EVENT = "configure_stop";
   
       void addLifecycleListener(LifecycleListener var1);
   
       LifecycleListener[] findLifecycleListeners();
   
       void removeLifecycleListener(LifecycleListener var1);
   
       void init() throws LifecycleException;
   
       void start() throws LifecycleException;
   
       void stop() throws LifecycleException;
   
       void destroy() throws LifecycleException;
   
       LifecycleState getState();
   
       String getStateName();
   
       public interface SingleUse {
       }
   }
   
   ```

   从上面可以看到 关于生命周期的四个方法 init、start、stop、destroy还有四个方法运行中的不同的状态

   Spring 的监听器更像是一个完整的应用，你把Event以及和它绑定的listener 放进这个应用，那么你只需要调用就行

   ```java
    public void fireLifecycleEvent(String type, Object data) {
   
           LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
           LifecycleListener interested[] = listeners;
           for (int i = 0; i < interested.length; i++)
               interested[i].lifecycleEvent(event);
   
       }
   ```

   上面是Tomcat 调用监听器的核心方法

   这反应了另外一种 监听器获取模式 llistener自己过滤

   3. SpringApplicationRunlistener实现

      可以看下EventPublishingRunListener源码 可以发现它获取listener识别方式是通过 方法来识别

#### Spring MVC处理流程

- 概述其处理流程

  - 图片大致了解其处理过程

    ![](https://segmentfault.com/img/bV58lC?w=788&h=375)

  - 结合源码描述其过程

    1. 首先用户发送请求，`DispatcherServlet`实现了`Servlet`接口，整个请求处理流：`HttpServlet.service -> FrameworkServlet.doGet -> FrameworkServlet.processRequest -> DispatcherServlet.doService -> DispatcherServlet.doDispatch`。 `doDispatch 开始正式进入MVC 分发处理`

    2. 从HandlerMapping中 获取HandlerExecutionChain（包含了 handler 和 拦截器）

       ```java
       protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
               if (this.handlerMappings != null) {
                   Iterator var2 = this.handlerMappings.iterator();
       
                   while(var2.hasNext()) {
                       HandlerMapping hm = (HandlerMapping)var2.next();
                       if (this.logger.isTraceEnabled()) {
                           this.logger.trace("Testing handler map [" + hm + "] in               DispatcherServlet with name '" + this.getServletName() + "'");
                   }
       
                       HandlerExecutionChain handler = hm.getHandler(request);
                       if (handler != null) {
                           return handler;
                       }
               }
        }
       ```

       HandlerMapping 实现可以参考[这篇文章](https://segmentfault.com/a/1190000014114278)

    3. 由于前面的HandlerMapping 的不同，所以映射的结果有不同也就是处理方式不同；所以我们需要不同HandlerAdapter 来处理这些由不同HandlerMapping得到的handler，所以这里很明显使用了适配器模式，将处理方式和对处理方式的调用进行调用

       ```java
           protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
               if (this.handlerAdapters != null) {
                   Iterator var2 = this.handlerAdapters.iterator();
       
                   while(var2.hasNext()) {
                       HandlerAdapter ha = (HandlerAdapter)var2.next();
                       if (this.logger.isTraceEnabled()) {
                           this.logger.trace("Testing handler adapter [" + ha + "]");
                       }
       
                       if (ha.supports(handler)) {
                           return ha;
                       }
                   }
               }
       
               throw new ServletException("No adapter for handler [" + handler + "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
       }
       ```

       HandlerAdapter 实现的分析可以参考[这篇文章](https://segmentfault.com/a/1190000014171117)

    4. 通过HandlerAdapter 调用具体的某种Handler来 处理请求并返回ModelAndView

    5. 视图解析，遍历`DispatcherServlet`的ViewResolver列表，获取对应的View对象，入口方法`DispatcherServlet.processDispatchResult`

    6. 渲染，调用5中获取的View的render方法，完成对Model数据的渲染。

    7. DispatcherServlet 将6中渲染后的数据返回响应给用户，到此一个流程结束。

### 分析Spring Boot启动源码

```java
    public ConfigurableApplicationContext run(String... args) {
        //用来监听只能运行一个Spring boot
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        //这个是Spring Boot专有的监听器
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        Collection exceptionReporters;
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            Banner printedBanner = this.printBanner(environment);
            //初始化IOC容器实例
            context = this.createApplicationContext();
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            //这里面完成了BeanDefinition的定位、载入、注册
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //进行IOC容器的初始化
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            listeners.started(context);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
```

```java
//这个方法比较有趣，在初始化完IOC容器后，注册了一个这个应用的shutdownhook
private void refreshContext(ConfigurableApplicationContext context) {
        this.refresh(context);
        if (this.registerShutdownHook) {
            try {
                context.registerShutdownHook();
            } catch (AccessControlException var3) {
                ;
            }
        }
    }
```

#### Spring 内置Tomcat启动源码解读

```java
    protected void onRefresh() {
        super.onRefresh();

        try {
            this.createWebServer();
        } catch (Throwable var2) {
            throw new ApplicationContextException("Unable to start web server", var2);
        }
    }
```

首先在初始化IOC容器的时候，通过onRefresh方法，调用creatWebServer创建指定的Web容器并启动

```java
 private void createWebServer() {
        WebServer webServer = this.webServer;
        ServletContext servletContext = this.getServletContext();
        if (webServer == null && servletContext == null) {
            ServletWebServerFactory factory = this.getWebServerFactory();
            this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
        } else if (servletContext != null) {
            try {
                this.getSelfInitializer().onStartup(servletContext);
            } catch (ServletException var4) {
                throw new ApplicationContextException("Cannot initialize servlet context", var4);
            }
        }

        this.initPropertySources();
    }

    public WebServer getWebServer(ServletContextInitializer... initializers) {
        Tomcat tomcat = new Tomcat();
        File baseDir = this.baseDirectory != null ? this.baseDirectory : this.createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        Connector connector = new Connector(this.protocol);
        tomcat.getService().addConnector(connector);
        this.customizeConnector(connector);
        tomcat.setConnector(connector);
        tomcat.getHost().setAutoDeploy(false);
        this.configureEngine(tomcat.getEngine());
        Iterator var5 = this.additionalTomcatConnectors.iterator();

        while(var5.hasNext()) {
            Connector additionalConnector = (Connector)var5.next();
            tomcat.getService().addConnector(additionalConnector);
        }

        this.prepareContext(tomcat.getHost(), initializers);
        return this.getTomcatWebServer(tomcat);
    }

    protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
        return new TomcatWebServer(tomcat, this.getPort() >= 0);
    }

    public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
        this.monitor = new Object();
        this.serviceConnectors = new HashMap();
        Assert.notNull(tomcat, "Tomcat Server must not be null");
        this.tomcat = tomcat;
        this.autoStart = autoStart;
        this.initialize();
    }

    private void initialize() throws WebServerException {
        logger.info("Tomcat initialized with port(s): " + this.getPortsDescription(false));
        Object var1 = this.monitor;
        synchronized(this.monitor) {
            try {
                this.addInstanceIdToEngineName();
                Context context = this.findContext();
                context.addLifecycleListener((event) -> {
                    if (context.equals(event.getSource()) && "start".equals(event.getType())) {
                        this.removeServiceConnectors();
                    }

                });
                this.tomcat.start();
                this.rethrowDeferredStartupExceptions();

                try {
                    ContextBindings.bindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
                } catch (NamingException var5) {
                    ;
                }

                this.startDaemonAwaitThread();
            } catch (Exception var6) {
                this.stopSilently();
                throw new WebServerException("Unable to start embedded Tomcat", var6);
            }

        }
    }
```

通过上面一步一步我们很容易发现Tomcat是怎么启动的

#### 关于Tomcat源码

- omcat的架构图

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftwlh8jmi0j20h90brq3m.jpg)

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftwlcws80xj20es09wt90.jpg)

Container的组成

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftxkshzdpvj21fc14wgmw.jpg)

- 一些名词

  1. Server：服务器，一个Tomcat只能有一个Server，可以看做就是Tomcat，用于控制Tomcat的生命周期

  2. Service：服务，有了Service就可以对外提供服务了，一个Server可以拥有多个Service

  3. Connector：连接器，表示接受请求的端点，并返回回复；Servlet容器处理请求，是需要Connector进行调度和控制的，Connector是Tomcat处理请求的主干，因此Connector的配置和使用对Tomcat的性能有着重要的影响 

  4. Engine：引擎，Engine下可以配置多个虚拟主机Virtual Host，每个虚拟主机都有一个域名，当Engine获得一个请求时，它把该请求匹配到某个Host上，然后把该请求交给该Host来处理，Engine有一个默认虚拟主机，当请求无法匹配到任何一个Host上的时候，将交给该默认Host来处理

  5. Host：虚拟主机，每个虚拟主机和某个网络域名Domain Name相匹配 ，每个虚拟主机下都可以部署(deploy)一个或者多个Web App ,每个Web App对应于一个Context，有一个Context path，当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理，匹配的方法是“最长匹配”，所以一个path==""的Context将成为该Host的默认Context，所有无法和其它Context的路径名匹配的请求都将最终和该默认Context匹配 

  6. Context ：一个Context对应于一个Web Application，一个Web Application由一个或者多个Servlet组成，Context在创建的时候将根据配置文件CATALINA_HOME/conf/web.xml和WEBAPP_HOME/WEB-INF/web.xml载入Servlet类，当Context获得请求时，将在自己的映射表(mapping table)中寻找相匹配的Servlet类，如果找到，则执行该类，获得请求的回应，并返回。 一个Host可以有多个Context，就相当于多个Web Application；其中TomcatEmbeddedContext就相当于这个

  7. Wrapper：最底层的容器，是对 Servlet 的封装，负责 Servlet 实例的创 建、执行和销毁 

     需要注意的是	Engine、Host、Context、Wrapper都是实现Container接口的类，所以里面的结构就像第三张图一样一个容器嵌套一个容器

#### Spring Boot源码之内置Servlet容器

创建WebServer并且启动

```java
  protected void onRefresh() {
        super.onRefresh();

        try {
            this.createWebServer();
        } catch (Throwable var2) {
            throw new ApplicationContextException("Unable to start web server", var2);
        }
    }


private void createWebServer() {
        WebServer webServer = this.webServer;
        ServletContext servletContext = this.getServletContext();
        if (webServer == null && servletContext == null) {
            ServletWebServerFactory factory = this.getWebServerFactory();
            //参数是一些ServletContextInitializer，Servlet容器启动的时候会遍历这些ServletContextInitializer，并调用onStartup方法
            this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
        } else if (servletContext != null) {
            try {
                this.getSelfInitializer().onStartup(servletContext);
            } catch (ServletException var4) {
                throw new ApplicationContextException("Cannot initialize servlet context", var4);
            }
        }

        this.initPropertySources();
    }
```

ServletContextInitializer的其中一个具体内容，目的是添加Servlet、Filter、Listener，包括那些自定义的，它们最终都会通过适配器变为ServletRegistrationBean

```java

private void selfInitialize(ServletContext servletContext) throws ServletException {
        this.prepareWebApplicationContext(servletContext);
        //得到 IOC容器
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        ServletWebServerApplicationContext.ExistingWebApplicationScopes existingScopes = new ServletWebServerApplicationContext.ExistingWebApplicationScopes(beanFactory);
        WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.getServletContext());
        existingScopes.restore();
        //将ServletContext注册到IOC容器
        WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.getServletContext());
        //得到ServletContextInitializerBean遍历器
        Iterator var4 = this.getServletContextInitializerBeans().iterator();

        while(var4.hasNext()) {
            //遍历执行所有ServletContextInitializer
            ServletContextInitializer beans = (ServletContextInitializer)var4.next();
            beans.onStartup(servletContext);
        }

    }
```



```java
//IOC容器初始化过程
protected void finishRefresh() {
        super.finishRefresh();
        WebServer webServer = this.startWebServer();
        if (webServer != null) {
            this.publishEvent(new ServletWebServerInitializedEvent(webServer, this));
        }

    }

//这里的启动WebServer其实就是注册所有的
    private WebServer startWebServer() {
        WebServer webServer = this.webServer;
        if (webServer != null) {
            webServer.start();
        }

        return webServer;
    }
//这里以Tomcat启动为例
   public void start() throws WebServerException {
        Object var1 = this.monitor;
        synchronized(this.monitor) {
            if (!this.started) {
                boolean var10 = false;

                try {
                    var10 = true;
                    this.addPreviouslyRemovedConnectors();
                    Connector var2 = this.tomcat.getConnector();
                    if (var2 != null && this.autoStart) {
                        this.performDeferredLoadOnStartup();
                    }

                    this.checkThatConnectorsHaveStarted();
                    this.started = true;
                    logger.info("Tomcat started on port(s): " + this.getPortsDescription(true) + " with context path '" + this.getContextPath() + "'");
                    var10 = false;
                } catch (ConnectorStartFailedException var11) {
                    this.stopSilently();
                    throw var11;
                } catch (Exception var12) {
                    throw new WebServerException("Unable to start embedded Tomcat server", var12);
                } finally {
                    if (var10) {
                        Context context = this.findContext();
                        ContextBindings.unbindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
                    }
                }

                Context context = this.findContext();
                ContextBindings.unbindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
            }
        }
    }

    private void performDeferredLoadOnStartup() {
        try {
            //得到该虚拟主机下的所有Context也就是WebApplication上下文
            Container[] var1 = this.tomcat.getHost().findChildren();
            int var2 = var1.length;

            for(int var3 = 0; var3 < var2; ++var3) {
                Container child = var1[var3];
                if (child instanceof TomcatEmbeddedContext) {
                    ((TomcatEmbeddedContext)child).deferredLoadOnStartup();
                }
            }

        } catch (Exception var5) {
            logger.error("Cannot start connector: ", var5);
            throw new WebServerException("Unable to start embedded Tomcat connectors", var5);
        }
    }

    public void deferredLoadOnStartup() {
        ClassLoader classLoader = this.getLoader().getClassLoader();
        ClassLoader existingLoader = null;
        if (classLoader != null) {
            existingLoader = ClassUtils.overrideThreadContextClassLoader(classLoader);
        }

        if (this.overrideLoadOnStart) {
            //加载该上下文下的所有Servlet
            super.loadOnStartup(this.findChildren());
        }

        if (existingLoader != null) {
            ClassUtils.overrideThreadContextClassLoader(existingLoader);
        }

    }

    public boolean loadOnStartup(Container[] children) {
        TreeMap<Integer, ArrayList<Wrapper>> map = new TreeMap();

        for(int i = 0; i < children.length; ++i) {
            Wrapper wrapper = (Wrapper)children[i];
            int loadOnStartup = wrapper.getLoadOnStartup();
            if (loadOnStartup >= 0) {
                Integer key = loadOnStartup;
                ArrayList<Wrapper> list = (ArrayList)map.get(key);
                if (list == null) {
                    list = new ArrayList();
                    map.put(key, list);
                }

                list.add(wrapper);
            }
        }

        Iterator i$ = map.values().iterator();

        while(i$.hasNext()) {
            ArrayList<Wrapper> list = (ArrayList)i$.next();
            Iterator i$ = list.iterator();

            while(i$.hasNext()) {
                Wrapper wrapper = (Wrapper)i$.next();

                try {
                    wrapper.load();
                } catch (ServletException var8) {
                    this.getLogger().error(sm.getString("standardContext.loadOnStartup.loadException", new Object[]{this.getName(), wrapper.getName()}), StandardWrapper.getRootCause(var8));
                    if (this.getComputedFailCtxIfServletStartFails()) {
                        return false;
                    }
                }
            }
        }

        return true;
    }s
```

### Spring 事务管理

#### Spring 事务传播行为

| 事务传播行为类型          | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | 如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择。 |
| PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |
| PROPAGATION_MANDATORY     | 使用当前的事务，如果当前没有事务，就抛出异常。               |
| PROPAGATION_REQUIRES_NEW  | 新建事务，如果当前存在事务，把当前事务挂起。                 |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。   |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛出异常。             |
| PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |

#### SpringBoot 如何自动配置事务相关类

在上一节中讲了 原来Spring用ProxyFactoryBean实现的AOP动态代理，在SpringBoot中已经弃用这种方式 结合了自动配置的方式

![](https://s2.ax1x.com/2019/02/02/k8dnD1.png)

1. 首先使用SpringBoot 的自动配置类 TransactionAutoConfiguration 设置了transactionManager
2. 在启用动态代理的同时用EnableTransactionManagement 注解启用了事务管理
3. 启用事务管理就是加载自动生成代理的类(AutoProxyRegistrar, 这是Spring给用户开的后门用于编程式动态添加bean)和向IOC容器里面添加了关于事务的拦截器
4. 通过代理生成器 结合IOC容器给Bean初始化时打开的口，动态生成了代理增强实例

#### Spring 事务管理的实现

其实Spring的事务管理 就是依赖于 TransactionInterceptor 这个类

因为它是实现了 MethodIntercepor 的拦截器类，所以调用被代理对象的方法时会调用其invoke方法

```java
	public Object invoke(MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
```

在invokeWithinTransaction 方法中就实现了 事务的新建、提交和回滚的操作

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// 如果当前没有事务就新建一个事务
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// 这里就是MethodInterceptor的精华，会先去递归调用所有拦截器链中的所有方法
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// 发生异常 在这个方面里面进行回滚
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
					try {
						return invocation.proceedWithInvocation();
					}
					catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException) {
								throw (RuntimeException) ex;
							}
							else {
								throw new ThrowableHolderException(ex);
							}
						}
						else {
							// A normal return value: will lead to a commit.
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});

				// Check result state: It might indicate a Throwable to rethrow.
				if (throwableHolder.throwable != null) {
					throw throwableHolder.throwable;
				}
				return result;
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}
		}
	}

```

#### 关于MethodInterceptor 和 其它的BeforeMethodInterceptor 等等区别

在MethodIntercepot 注释上面有下面这样一句话

```java
 * Intercepts calls on an interface on its way to the target. These
 * are nested "on top" of the target.
```

再 根据 ReflectiveMethodInvocation类里面的process 方法  下面这句代码

```java
if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
```

但是注意的是 在MethodInterceptor中invoke方法要回调 JoinPoint的proceed方法

### 相关面试题

http://www.importnew.com/15851.html#ioc_di

#### 其中值得学习的类

- ResolveableType

  这个类是为了解决获取泛型实际类型

  [这里具体使用方法](http://jinnianshilongnian.iteye.com/blog/1993608)

### 相关链接

- Spring Boot源码分析

  [https://fangjian0423.github.io/2017/05/22/springboot-embedded-servlet-container/](https://fangjian0423.github.io/2017/05/22/springboot-embedded-servlet-container/)

- Tomcat

  [https://juejin.im/post/5af27c34f265da0b78687e14](https://juejin.im/post/5af27c34f265da0b78687e14)

  [http://www.importnew.com/27309.html](http://www.importnew.com/27309.html)

  [https://juejin.im/post/58eb5fdda0bb9f00692a78fc](https://juejin.im/post/58eb5fdda0bb9f00692a78fc)

  

  











