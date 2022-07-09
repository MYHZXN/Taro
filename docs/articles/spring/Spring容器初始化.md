    
    IOC容器接口
    1.BeanFactory
        Spring内部容器，不建议开发使用
        懒加载，获取Bean的时候去创建Bean
    2.ApplicationContext
        更强大，开发中常使用
        启动时便加载Bean信息并创建Bean
        
    SpringBoot启动
        SpringApplication.run(String... args)    
        //创建容器，返回ConfigurableApplicationContext
        //实际是AnnotationConfigServletWebServerApplicationContext
        //构造Context，生成注解扫描器和包扫描器
        context = this.createApplicationContext();
        
    BeanFactory
    |
    ApplicationContext 更强大
    |
    ConfigurableApplicationContext
    |
    AbstractApplicationContext
    
        refresh()
            
            //1，为refresh做好准备
            //  初始化WebEnvironment
            //      this.initPropertySources(); 
            //      this.getEnvironment().validateRequiredProperties();
            //  添加监听器
            //      this.earlyApplicationListeners = new LinkedHashSet(this.applicationListeners);
            prepareRefresh() 
            
            //2.获取ApplicationContext的容器
            //  GenericApplicationContext.DefaultListableBeanFactory
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            
            
            //3.配置容器的标准上下文
            prepareBeanFactory(beanFactory);
            
            //4.允许容器的子类执行工厂后置处理器
            // 修改应用程序上下文的内部bean工厂标准后初始化。所有bean定义将被加载,
            //但没有实例化bean。这允许特殊Bean注册后在某些应用程序上下文实现处理器等。
            postProcessBeanFactory(beanFactory);
            
            //5.调用工厂后置处理器
            invokeBeanFactoryPostProcessors(beanFactory);
            
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
            beanPostProcess.end();
            
            // Initialize message source for this context.
            initMessageSource();
            
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();
            
            // Initialize other special beans in specific context subclasses.
            onRefresh();
            
            // Check for listener beans and register them.
            registerListeners();
            
            //初始化容器，创建Bean
            finishBeanFactoryInitialization(beanFactory)
             
            //DefaultListableBeanFactory
            beanFactory.preInstantiateSingletons()
                //实例化Bean
                AbstractBeanFactory#getBean
                AbstractBeanFactory#doGetBean
                    // 检查缓存是否存在
                    // 容器有三级缓存
                    // 	1，singletonObjects
                    //	2. earlySingletonObjects
                    //	3. singletonFactories
                    // 疑问？这里为什么一开始要getSingleton        
                    //      解决循环依赖问题
                    Object sharedInstance = getSingleton(beanName);
                
                    //  如果没有缓存对象
                    //  1.singleton单例
                    //      DefaultSingletonBeanRegistry#getSingleton
                            getSingleton(beanName, () -> {
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
                            
                            resolveBeforeInstantiation
                            doCreateBean
                            
                                AbstractAutowireCapableBeanFactory
                                    //
                                    instanceWrapper = createBeanInstance(beanName, mbd, args);
                                    
                    //  2.prototype多例
                    //  3.scope
                    //      request、session
                    
            
            // Last step: publish corresponding event.
            finishRefresh();
            
            
                
                
    
    
    