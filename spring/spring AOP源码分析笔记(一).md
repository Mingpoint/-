 spring AOP源码分析笔记(一)
======================

#####  1. AnnotationAwareAspectJAutoProxyCreator 对象创建过程

#### 2.代理对象的创建过程


首先看AnnotationAwareAspectJAutoProxyCreator 类图：

![alt](https://github.com/Mingpoint/notes/tree/master/imgs/AnnotationAwareAspectJAutoProxyCreator.png)



由上图可以看出，AnnotationAwareAspectJAutoProxyCreator实现了 SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware。所以可以重点关注下面的方法

`

    BeanFactoryAware.setBeanFactory(BeanFactory beanFactory);
    //BeanPostProcessor后置处理器
    SmartInstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(Class<?> beanClass, String beanName);
    SmartInstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(Object bean, String beanName);
    
    public class MainConfigAopTest {
        @Test
          public void test01 () {
              //启动IOC 容器，初始化bean
            AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfigAop.class);
            MathCalculator bean = ctx.getBean(MathCalculator.class);
            bean.div(2,4);
        }
    }
    
`

启动IOC 容器 源码

`

    public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) { 
        this();   
        register(annotatedClasses);
        //刷新IOC容器，其实也就是实例化单实例bean
        refresh();
    }

`

下面是 refresh 方法片段

`

    public void refresh() throws BeansException, IllegalStateException {
        ....
        // 注册创建BeanPostProcessors的后置处理器,AnnotationAwareAspectJAutoProxyCreator就是
        //在这里调用getBean方法进行初始化创建
        registerBeanPostProcessors(beanFactory);
        ..
        ..
        // 实例化非延迟加载的单实例bean
        finishBeanFactoryInitialization(beanFactory);
    
        // Last step: publish corresponding event.
        finishRefresh();
    }
    
`

进入registerBeanPostProcessors(beanFactory),对调用到PostProcessorRegistrationDelegate.registerBeanPostProcessors

`

    public static void registerBeanPostProcessors(
    			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    			// 获取所有的BeanPostProcessor后置处理器
    		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    		....
    		....
    		// BeanPostProcessor后置处理器 implement PriorityOrdered,Ordered.
    		for (String ppName : postProcessorNames) {
    			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
    		    	....
    			    ....
    			}
    			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
    				orderedPostProcessorNames.add(ppName);
    			}
    			else {
    				nonOrderedPostProcessorNames.add(ppName);
    			}
    		}
            ....
            ....
    
    		// BeanPostProcessor后置处理器 implement Ordered,由类图可以看出AnnotationAwareAspectJAutoProxyCreator
    		// implement Ordered 
    		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
    		for (String ppName : orderedPostProcessorNames) {
    		//调用beanFactory.getBean(ppName, BeanPostProcessor.class),创建AnnotationAwareAspectJAutoProxyCreator
    			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    			orderedPostProcessors.add(pp);
    			if (pp instanceof MergedBeanDefinitionPostProcessor) {
    				internalPostProcessors.add(pp);
    			}
    		}
    		sortPostProcessors(orderedPostProcessors, beanFactory);
    		//注册AnnotationAwareAspectJAutoProxyCreator到beanFactory上
    		registerBeanPostProcessors(beanFactory, orderedPostProcessors);
    		...
    		...
    	}
    	
`
正真在创建bean的流程
beanFactory.getBean->doGetBean->createBean->doCreateBean

`

    protected <T> T doGetBean(
 			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
 			throws BeansException {
 
        .....
        .....
        //缓存中查询该bean是否实例化，由于AnnotationAwareAspectJAutoProxyCreator第一次创建肯定为null
 		Object sharedInstance = getSingleton(beanName);
 		if (sharedInstance != null && args == null) {
 			....
 			....
 			....
 		}
 
 		else {
 			.....
 			.....
 			.....
 			.....

                Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        beforePrototypeCreation(beanName);
                        try {
                        //创建bean
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    }
                });
 				
 		...
 		...
 		...
 		return (T) bean;
 	}
 	
 		protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    		....
    		....
    
    		try {
    			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    			//BeanPostProcessors 后置处理器处理bean是否需要创建代理对象, 
    			//bean创建之前会有一个拦截调用AbstractAutoProxyCreator.postProcessBeforeInstantiation()
    			//然后通过AspectJAwareAdvisorAutoProxyCreator.shouldSkip 反射生成List<Advisor>，List<Advisor>就是我们要
    			//logAspecj 拦截通知方法的一个封装
    			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    			if (bean != null) {
    				return bean;
    			}
    		}
    		catch (Throwable ex) {
    			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
    					"BeanPostProcessor before instantiation of bean failed", ex);
    		}
    		//创建bean
    		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    		if (logger.isDebugEnabled()) {
    			logger.debug("Finished creating instance of bean '" + beanName + "'");
    		}
    		return beanInstance;
    	}
    	
    	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        			throws BeanCreationException {
        	......
        	......
        	
        		Object exposedObject = bean;
        		try {
        		//给bean的属性赋值
        			populateBean(beanName, mbd, instanceWrapper);
        			if (exposedObject != null) {
        			//初始化bean,主要是BeanPostProcessors的处理，以及实现了Awre接口的出，以及Lazy，init()方法的处理
        				exposedObject = initializeBean(beanName, exposedObject, mbd);
        			}
        		}
        		catch (Throwable ex) {
        			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
        				throw (BeanCreationException) ex;
        			}
        			else {
        				throw new BeanCreationException(
        						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        			}
        		}
            ......
            ......
        		return exposedObject;
        	}
        	
`

   initializeBean方法调用
    
`
        	
        	protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
            		if (System.getSecurityManager() != null) {
            			AccessController.doPrivileged(new PrivilegedAction<Object>() {
            				@Override
            				public Object run() {
            				//处理Aware接口的方法回调
            					invokeAwareMethods(beanName, bean);
            					return null;
            				}
            			}, getAccessControlContext());
            		}
            		else {
            		//处理Aware接口的方法回调
            			invokeAwareMethods(beanName, bean);
            		}
            
            		Object wrappedBean = bean;
            		if (mbd == null || !mbd.isSynthetic()) {
            		//应用后置处理器，注意此处处理的是postProcessBeforeInitialization
            		//与InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 不一样
            			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
            		}
            
            		try {
            		//执行自定义的初始化方法
            			invokeInitMethods(beanName, wrappedBean, mbd);
            		}
            		catch (Throwable ex) {
            			throw new BeanCreationException(
            					(mbd != null ? mbd.getResourceDescription() : null),
            					beanName, "Invocation of init method failed", ex);
            		}
            
            		if (mbd == null || !mbd.isSynthetic()) {
                        //应用后置处理器，注意此处处理的是 postProcessAfterInitialization
                        //与InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation 不一样
                        //这里我们需要关注的是MathCalculator bean的创建
                        //调用AbstractAutoProxyCreator.postProcessAfterInitialization 创建了MathCalculator的代理对象
            			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
            		}
            		return wrappedBean;
            	}

 	`
 	
调用AbstractAutoProxyCreator.postProcessAfterInitialization 创建MathCalculator的代理对象，
    postProcessAfterInitialization->wrapIfNecessary
    
    `	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
     		....
     		//  创建代理对象
     		// 去找出Advice 和 Advisors，这里和上面resolveBeforeInstantiation 这里是一样的，就是找到List<Advice>
     		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
     		if (specificInterceptors != DO_NOT_PROXY) {
     			this.advisedBeans.put(cacheKey, Boolean.TRUE);
     			//创建代理对象
     			Object proxy = createProxy(
     					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
     			this.proxyTypes.put(cacheKey, proxy.getClass());
     			return proxy;
     		}
     
     		this.advisedBeans.put(cacheKey, Boolean.FALSE);
     		return bean;
     	}
    `
    
    
 createProxy->getProxy->createAopProxy
    
    
 `
 
       public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
            if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
              .....
              .....
                //Cglib代理
                return new ObjenesisCglibAopProxy(config);
            }
            else {
                //jdk代理
                return new JdkDynamicAopProxy(config);
            }
        }
    
 `





以上就是AOP的初始化过程，以及代理对象的创建过程
总结：
  * @EnableAspectJAutoProxy 开启AOP功能
  * @EnableAspectJAutoProxy 会给容器中注册一个组件 AnnotationAwareAspectJAutoProxyCreator
  * AnnotationAwareAspectJAutoProxyCreator是一个后置处理器,实现InstantiationAwareBeanPostProcessor，BeanPostProcessor接口
    需要注意 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation、
            InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation 方法，这个是在resolveBeforeInstantiation
            方法时候调用
    需要注意 BeanPostProcessor.postProcessBeforeInitialization、BeanPostProcessor.postProcessAfterInitialization 方法，
    这个是在initializeBean调用
    以及注意这个几个方法的调用时机
    

 	
 	
 	








