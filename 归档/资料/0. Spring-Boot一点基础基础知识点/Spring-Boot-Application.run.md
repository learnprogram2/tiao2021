SpringApplication:
0. constant:
	三個ApplicationContext的全限定名.
	benner的地址和 配置banner的属性名字, 下面还有个实例的bannerMode(console/log/none), 还有一个banner用来输出banner_stream.
	awtHeadless: 不知道这个是什么
	从logFactory里面拿log
1. fields:
	1. source:
		resourceLoader: 
		primarySource: 一个set, 装着主类
		sources: 一个LinkedHashSet???为什么用这个.
		mainApplicationClass: 主类.
		logStartupInfo??? boolean
		addCommandLineProperties: boolean
		addConversionService: boolean;
		
		
SpringApplication.run()里面:
1. new SpringApplication(mainClass);
	0. 从classpath里的class检测当前是什么webApplication.
	1. 设置Initializers: 从spring.factories里面拿
	org.springframework.context.ApplicationContextInitializer 是什么??
		Spring初始化的时候在refresh之前调用. 用来初始化(initialize), 比如: 注册property源或者
	2. 设置ApplicationListener: 从spring.factories里面拿, 放在一个list里面
		第二次restart启动找到了15个
	3. 根据运行栈, 往上找到mainclass. 设置好主类.
2. .run(args);
	1. 开一个秒表: stopWatch
	2. 从参数里拿到然后设置Headless (??? 是干什么的)
	3. 设置SpringApplicationRunListeners. 发送starting事件.
		几个start监听器:  (???从哪里拿的)
			RestartApplicationListener: 这个会把系统重启一遍.
				Restart会开启一个线程, 然后restart用这个线程, 然后把当前线程关掉. 开始restart RestartLauncher这个线程.
					(???第一个线程干了点什么??????????)
				再重启的时候因为已经设置了starter就不会再跳到另一个线程了, 会跳过去接着运行后面的.
			LoggingApplicationListener: 找啊找logger.(???logger系统是什么)
			...几个不认识的.
	4. 把args保存成ApplicationArguments: 解析commandLineArgs, 存起来args.
	5. prepareEnvironment, 使用listeners和上一步的applicationArguments准备好一个configurableEnvironment(存着PropertySources(操作必须在ApplicationContext.refresh之前))
		1. 创建一个standardServletEnvironment(就是configurableEnvironment)
		2. 用args配置上面的configurableEnvironment
			双锁创建一个单例conversionService, 然后设置到cE里(感觉是类型转换的工具)
			把args里的参数设置到cE里, 从args里拿一些激活的参数文件放进cE里.
		3. 把cE里面的PropertySources取出来, 包装一下在设置到里面的第一个. key=configurationProperties.
		4. listener 发放environmentPrepared事件.
			也会重新调用一边........
		5. 把SpringApplication这个绑定到cE里面(使用的Binder)
		6. 又把cE里面的PropertySources变成一个value, 放在里面的第一个, key=configurationProperties.
			这里的每个cE都是当前线程的 cE, 虽然每个参数都一样组成的都一样.
	6. configureIgnoreBeanInfo(cE); 
		设置一下System里面的'spring.bean.ignore'这个参数, System参数里没有就设成cE的PropertySources里面的(默认true)
	7. 打印一下banner(如果可以)
	8. createApplicationContext(); 根据应用类型创建一个ApplicationContext(不设置默认annotationConfigApplicationContext)
	9. getSpringFacotriesInstances(SpringBootExceptionReporter.class): 
		从 springFactories文件里拿出名字, 并实例化ExceptionReport
	10. prepareContext(applicationContext, cE, listeners, applicationArgs, banner)
		1. 把cE放到context里
		2. postProcessApplicationContext: 执行和context相关的 postProcess (???postProcess是什么)
		3. applyInitializers(context): 把context交给所有的initializer, 进行初始化
			每个initializer都有自己的逻辑. (???initializer是什么?)
		4. listener发送contextPrepared事件
		5. 下面是Add boot specific singleton beans
			从context里拿到BeanFactory. 注册applicationArguments和banner, 设置允许bean覆盖.
		6. load sources. 在context里加载????哪些类?
		7. listener发送contextLoaded事件.
	// 下面就乱了.
	11. refreshContext: spring开始加载bean了
	12. 停掉stopWatch
	13. listeners发送start事件
	14. callRunners
	15. listeners发送running事件
		
		
		
		
		
		
		
		
		
		
		
		
		
		