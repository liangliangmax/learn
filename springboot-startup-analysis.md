# SpringApplication.run 启动全链路源码分析

  源码版本：Spring Boot 4.0.x  |  源码目录：/home/liang/workspace/spring-boot/core/spring-boot/

  分析范围：从 `SpringApplication.run(Class, String...) 第一行到容器启动完成返回`

>
  **图例：**
  · 阶段/入口
  · 核心步骤
  · 子方法
  · 更深层方法
  · 细节实现
  · 底层操作
  · 事件发布

## 入口：静态 run 方法

> **【阶段】**

🚀
### 调用链

```
SpringApplication.run(Class primarySource, String... args)           // L1353
  └─ run(new Class[]{primarySource}, args)                           // L1354
      └─ run(Class[] primarySources, String[] args)                  // L1364
          └─ new SpringApplication(primarySources).run(args)            // L1365
              ├─ [1] new SpringApplication(primarySources)  → 构造阶段
              └─ [2] .run(args)                            → 启动阶段
```

静态方法 `run(Class, String...) 只是包装，核心逻辑在实例方法 `run(String...) 中。

## 阶段一：构造 SpringApplication 实例

> **【阶段】**

🏗️
### SpringApplication(ResourceLoader, Class...)  L274-284

**1.**`保存 primarySources L277`

this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources))

保存传入的主配置类（如 MyApplication.class），后续在 `getAllSources() 中作为 BeanDefinition 的来源。`

**2.**`WebApplicationType.deduce() L278 / WebApplicationType.java L63-71`

推断 Web 应用类型（SERVLET / REACTIVE / NONE）：

- 先查 META-INF/spring.factories 中的 WebApplicationType.Deducer 实现，有则调用 deduceWebApplicationType()

- 没有 Deducer → isServletApplication()：检查 classpath 是否同时存在 jakarta.servlet.Servlet 和 org.springframework.web.context.ConfigurableWebApplicationContext

- 两个类都在 → **SERVLET**；否则 → **NONE**

结果存入 this.properties.webApplicationType，后续决定创建哪种 ApplicationContext 和 Environment。

**3.**`getSpringFactoriesInstances(BootstrapRegistryInitializer.class) L279-280`

通过 SpringFactoriesLoader.forDefaultResourceLocation().load(type) 从 META-INF/spring.factories 加载所有 BootstrapRegistryInitializer 实现。

这些初始化器将在 `createBootstrapContext() 中被调用，用于在 ApplicationContext 创建前注册早期实例。`

**4.**`setInitializers(getSpringFactoriesInstances(ApplicationContextInitializer.class)) L281`

同样从 spring.factories 加载，存入 this.initializers 列表。

这些初始化器将在 `prepareContext() 的 `applyInitializers() 中被遍历调用，对 ApplicationContext 做早期定制。

**5.**`setListeners(getSpringFactoriesInstances(ApplicationListener.class)) L282`

从 spring.factories 加载所有 ApplicationListener 实现，存入 this.listeners 列表。

这些监听器将在 `contextLoaded() 中被注册到 ApplicationContext，监听各种应用事件。`

**6.**`deduceMainApplicationClass() L283 / L286-296`

使用 StackWalker.getInstance(RETAIN_CLASS_REFERENCE) 遍历调用栈：

stack.filter(frame → "main".equals(frame.getMethodName())).findFirst().map(frame → frame.getDeclaringClass())

找到调用 main() 方法的类，存入 this.mainApplicationClass，用于日志输出和 Banner 显示。

SpringFactoriesLoader 是 Spring Boot SPI 机制的核心：它从所有 jar 的 META-INF/spring.factories 文件中读取接口→实现类列表，反射实例化。构造阶段连续调了 3 次（BootstrapRegistryInitializer、ApplicationContextInitializer、ApplicationListener），把 Spring Boot 自动配置的"骨架"搭起来了。

## 阶段二：run(String... args) 核心启动流程

方法位置：`SpringApplication.run(String... args) L304-342`

这是整个 Spring Boot 启动的核心方法，共 20 个步骤，分为 6 个阶段。

### 阶段 A：启动准备（步骤 1-6）

> **【阶段】**

**1.**`Startup.create() L305 / L1771-1776`

创建启动计时器：

- 检查 classpath 是否有 jdk.crac.management.CRaCMXBean 和 org.crac.management.CRaCMXBean

- 有 → CoordinatedRestoreAtCheckpointStartup（支持 CRaC 检查点恢复，startTime() 优先用 restoreTime）

- 无 → StandardStartup（startTime = System.currentTimeMillis() 记录启动开始时间）

**2.**`shutdownHook.enableShutdownHookAddition() L306-308 / SpringApplicationShutdownHook.java L74-76`

ApplicationProperties.registerShutdownHook 默认 true。

此方法只设置 shutdownHookAdditionEnabled = true 标记，**并不立即注册** Runtime shutdownHook。

真正的 Runtime.getRuntime().addShutdownHook() 延迟到 `refreshContext() 中的 `registerApplicationContext() 时执行。

**3.**`createBootstrapContext() L309 / L344-348`

创建 DefaultBootstrapContext（引导上下文）：

- new DefaultBootstrapContext() — 内部维护 Map<Class, InstanceSupplier> 和 Map<Class, Object>，以及一个 SimpleApplicationEventMulticaster

- 遍历所有 BootstrapRegistryInitializer，调用 initializer.initialize(bootstrapContext)

每个 initializer 可以通过 bootstrapContext.register(type, supplier) 注册类型→实例供应器

BootstrapContext 的作用：在 ApplicationContext 创建前，提供一个轻量级容器，存放配置文件加载器、早期 Bean 实例等。在 `prepareContext() 完成后通过 bootstrapContext.close(context) 关闭。`

**4.**`configureHeadlessProperty() L311 / L448-451`

System.setProperty("java.awt.headless", "true")

设置无头模式，防止 AWT/Swing 弹窗（服务器环境不需要 GUI）。

默认 this.headless = true，可通过 spring.main.headless=false 覆盖。

**5.**`getRunListeners(args) L312 / L453-465`

获取运行监听器集合：

- 构造 ArgumentResolver，绑定 SpringApplication.class → this 和 String[].class → args

- SpringFactoriesLoader.forDefaultResourceLocation().load(SpringApplicationRunListener.class, argumentResolver) — 从 spring.factories 加载

- 检查 ThreadLocal<SpringApplicationHook> 是否有 hook，有则追加 hook 提供的 listener

- 包装成 SpringApplicationRunListeners（聚合器，内部 List<SpringApplicationRunListener>）

核心实现是 EventPublishingRunListener（见下方详解），它将 RunListener 回调转换为 Spring ApplicationEvent 发布。

**6.**`listeners.starting(bootstrapContext, mainApplicationClass) L313`

触发 starting 事件：

SpringApplicationRunListeners.starting() (L55-62) → 遍历所有 listener 调用 listener.starting(bootstrapContext)

`EventPublishingRunListener.starting() (L75-77)：`

  multicastInitialEvent(new ApplicationStartingEvent(bootstrapContext, this.application, this.args))

  → refreshApplicationListeners() — 把 SpringApplication 的 listeners 加到 initialMulticaster

  → initialMulticaster.multicastEvent(event) — 用 SimpleApplicationEventMulticaster 广播

⚡ 发布事件：ApplicationStartingEvent

此时 ApplicationContext 还没创建，事件通过 initialMulticaster（非 ApplicationContext 的 multicaster）广播。

### 阶段 B：环境准备（步骤 7-8）

> **【阶段】**

**7.**`new DefaultApplicationArguments(args) L315 / DefaultApplicationArguments.java L42-46`

包装命令行参数：

- this.source = new Source(args) — Source extends SimpleCommandLinePropertySource

- 解析 --key=value 格式的参数为 option args，非 -- 开头的为 non-option args

- 提供 getOptionNames()、getOptionValues(name)、getNonOptionArgs() 等方法

后续注册为单例 Bean "springApplicationArguments"，供其他组件注入使用。

**8.**`prepareEnvironment(listeners, bootstrapContext, applicationArguments) L316 / L350-368`

这是最复杂的方法之一，分 7 个子步骤：

#### 8.1 getOrCreateEnvironment()

L353 / L475-485

根据 WebApplicationType 创建 Environment：

- SERVLET → StandardServletEnvironment（通过 ApplicationContextFactory 从 spring.factories 查找）

- NONE → ApplicationEnvironment（继承 StandardEnvironment，重写 createPropertyResolver() 使用 ConfigurationPropertySources）

Environment 内部初始 PropertySources 顺序：

systemProperties → systemEnvironment → applicationInfo（后续会插入更多）

#### 8.2 configureEnvironment(environment, args)

L354 / L498-504

- **setConversionService** — new ApplicationConversionService()，注册 Spring Boot 自定义的类型转换器（String→Duration、String→DataSize 等）

- **configurePropertySources()** L513-533：

- 如果有 defaultProperties → DefaultPropertiesPropertySource.addOrMerge() 放到最后

- 如果有命令行参数 → 包装为 SimpleCommandLinePropertySource 放到**最前**（优先级最高）

- 添加 ApplicationInfoPropertySource（包含 PID 和应用版本号）到最后

- **configureProfiles()** L543-544：空实现，profile 由后续配置文件加载时处理

#### 8.3 ConfigurationPropertySources.attach(environment)

L355

把 Environment 的 PropertySources 包装成 ConfigurationPropertySourcesPropertySource，名为 "configurationProperties"，加到 PropertySources 中。

这使得 Spring Boot 的 Binder 能通过 spring.main.xxx 等前缀绑定属性。

#### 8.4 listeners.environmentPrepared(bootstrapContext, environment)

L356

⚡ 发布事件：ApplicationEnvironmentPreparedEvent

通过 EventPublishingRunListener (L80-84) → initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(...))

**这是最关键的事件之一**，触发的监听器包括：

- ConfigDataEnvironmentPostProcessor — 加载 application.yml / application.properties / application-profile.yml，以及 spring.config.import 指定的额外配置

- LoggingApplicationListener — 初始化日志系统（Logback/Log4j2）

- 各种 EnvironmentPostProcessor — 对 Environment 做后置处理（如注入随机数、JSON 配置等）

#### 8.5 ApplicationInfoPropertySource.moveToEnd(environment)

L357

把 applicationInfo 移到最后（优先级最低），确保用户配置能覆盖它。

DefaultPropertiesPropertySource.moveToEnd(environment) L358 — 同理。

#### 8.6 bindToSpringApplication(environment)

L361 / L550-557

Binder.get(environment).bind("spring.main", Bindable.ofInstance(this.properties))

把 spring.main.* 前缀的属性绑定到 ApplicationProperties 对象上：

spring.main.lazy-initialization → properties.lazyInitialization

spring.main.banner-mode → properties.bannerMode

spring.main.web-application-type → properties.webApplicationType（可覆盖构造时推断的类型）

等等。

#### 8.7 EnvironmentConverter.convertEnvironmentIfNecessary()

L362-365 / EnvironmentConverter.java L73-78

如果用户自定义了 Environment（this.isCustomEnvironment == true），跳过转换。

否则：检查当前 Environment 类型是否与 deduceEnvironmentClass() 推断的类型一致：

- 一致 → 原样返回

- 不一致 → convertEnvironment()：反射创建目标类型 Environment，复制 profiles、conversionService、propertySources（排除 servlet 相关的 PropertySource）

最后再次 ConfigurationPropertySources.attach() L366 重新包装。

### 阶段 C：Banner 打印 + ApplicationContext 创建（步骤 9-10）

> **【阶段】**

**9.**`printBanner(environment) L317 / L559-570`

#### printBanner() 流程

- 检查 properties.getBannerMode(environment)：

- OFF → 返回 null，不打印

- 如果结构化日志开启 → 自动设为 OFF（ApplicationProperties.getBannerMode() L104-112）

- 默认 → CONSOLE

- 创建 SpringApplicationBannerPrinter(resourceLoader, this.banner)

- getBanner(environment) (L73-82)：

- 先找 spring.banner.location（默认 banner.txt）资源文件 → ResourceBanner

- 没有 → 用自定义 this.banner（如果设了）

- 都没有 → SpringBootBanner（默认 ASCII Art Spring Logo）

- CONSOLE 模式 → banner.printBanner(environment, sourceClass, System.out)

- LOG 模式 → banner.printBanner() 输出到 ByteArrayOutputStream → 转字符串 → logger.info()

返回 PrintedBanner（装饰器，后续可重复打印无需再传 sourceClass）。

**10.**`createApplicationContext() L318 / L579-584`

创建 ApplicationContext：

- applicationContextFactory.create(webApplicationType)

- DefaultApplicationContextFactory.create() (L55-64)：

- 先从 spring.factories 加载所有 ApplicationContextFactory 实现，逐个尝试 create(webApplicationType)

- 都返回 null → createDefaultApplicationContext() (L66-71)：

- 非 AOT → new AnnotationConfigApplicationContext()（内含 AnnotatedBeanDefinitionReader + ClassPathBeanDefinitionScanner）

- AOT → new GenericApplicationContext()（更轻量，不自动扫描）

- 断言 context 不为 null

AnnotationConfigApplicationContext 是最常用的实现，它继承了 GenericApplicationContext（提供 BeanFactory）并组合了 AnnotatedBeanDefinitionReader（注册 @Configuration/@Bean）和 ClassPathBeanDefinitionScanner（扫描 @Component）。

10.5`context.setApplicationStartup(this.applicationStartup) L319`

设置 ApplicationStartup（启动指标收集器），默认 ApplicationStartup.DEFAULT（无操作）。可用于收集各步骤耗时。

### 阶段 D：Context 准备（步骤 11）

> **【阶段】**

**11.**`prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner) L320 / L380-419`

这是启动流程中最长的方法，分 12 个子步骤：

#### 11.1 context.setEnvironment(environment)

L383

把准备好的 Environment 设置到 ApplicationContext 上（替换默认的）。

#### 11.2 postProcessApplicationContext(context)

L384 / L591-607

对 context 做后置处理：

- 如果有 beanNameGenerator → 注册为单例 "org.springframework.context.annotation.ConfigurationClassPostProcessor.importBeanNameGenerator"

- 如果有 resourceLoader：

- context 是 GenericApplicationContext → setResourceLoader()

- context 是 DefaultResourceLoader → setClassLoader()

- 如果 addConversionService → beanFactory.setConversionService(environment.getConversionService())

#### 11.3 addAotGeneratedInitializerIfNecessary(this.initializers)

L385 / L421-439

AOT 模式下（AotDetector.useGeneratedArtifacts()）：

- 过滤出 AotApplicationContextInitializer 类型的 initializer

- 如果没有 → 查找 mainApplicationClass.getName() + "__ApplicationContextInitializer" 类

- 找不到 → 抛 AotInitializerNotFoundException

- 找到 → 用 AotApplicationContextInitializer.forInitializerClasses() 包装

- 移除原有 AOT initializer，把新的插到列表最前

Native Image 模式下还会检查 NativeImageRequirementsException.throwIfNotMet()（需要 Java 25+）。

#### 11.4 配置 BeanFactory

L387-392

- autowireCapableBeanFactory.setAllowCircularReferences(properties.isAllowCircularReferences()) — 默认 false

- listableBeanFactory.setAllowBeanDefinitionOverriding(properties.isAllowBeanDefinitionOverriding()) — 默认 false

#### 11.5 applyInitializers(context)

L393 / L616-625

遍历所有 ApplicationContextInitializer（按 @Order 排序），调用 initializer.initialize(context)。

每个 initializer 的泛型类型通过 GenericTypeResolver.resolveTypeArgument() 解析，断言 context 是该类型的实例。

典型 initializer 包括：

- DelegatingApplicationContextInitializer — 从环境变量加载额外 initializer

- ContextIdApplicationContextInitializer — 设置 contextId

- ConfigurationWarningsApplicationContextInitializer — 检查有问题的配置

- 各种 autoconfigure 模块提供的 initializer

#### 11.6 listeners.contextPrepared(context)

L394

⚡ 发布事件：ApplicationContextInitializedEvent

EventPublishingRunListener.contextPrepared() (L87-89)：

multicastInitialEvent(new ApplicationContextInitializedEvent(this.application, this.args, context))

此时 context 已创建并初始化，但还没加载 BeanDefinition。

#### 11.7 bootstrapContext.close(context)

L395 / DefaultBootstrapContext.java L139-141

events.multicastEvent(new BootstrapContextClosedEvent(this, applicationContext))

BootstrapContext 使命完成，发布关闭事件。注册了 addCloseListener 的组件可以在此把 BootstrapContext 中的实例迁移到 ApplicationContext。

#### 11.8 logStartupInfo / logStartupProfileInfo

L396-399 / L633-661

- logStartupInfo() — 如果是 root context（parent == null），调用 StartupInfoLogger.logStarting()：

- 输出 "Starting MyApplication v1.0 using Java 17 with PID 12345 (/path/to/app started by user in /dir)"

- debug 级别输出 "Running with Spring Boot vX, Spring vX"

- logStartupProfileInfo() — 输出 active profiles 或 default profiles：

- 有 active profiles → "The following 1 profile is active: \"dev\""

- 没有 → "No active profile set, falling back to 1 default profile: \"default\""

#### 11.9 注册单例 Bean

L401-404

- beanFactory.registerSingleton("springApplicationArguments", applicationArguments)

- beanFactory.registerSingleton("springBootBanner", printedBanner)（如果 banner 不为 null）

这些单例不经过 BeanDefinition，直接注册到 BeanFactory 的 singletonObjects。

#### 11.10 懒加载 / KeepAlive / 属性排序

L405-411

- properties.isLazyInitialization() → context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor()) — 所有 Bean 默认懒加载

- properties.isKeepAlive() → context.addApplicationListener(new KeepAlive()) — ContextRefreshedEvent 时启动非守护线程保持 JVM 存活，ContextClosedEvent 时停止

- context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context)) — 保证 defaultProperties 排在 PropertySources 最后（HIGHEST_PRECEDENCE）

#### 11.11 加载 Sources

L412-417

非 AOT 模式下：

- getAllSources() (L1189-1198) — 合并 primarySources（构造时传入的类）和 properties.getSources()（通过 spring.main.sources 设置的额外源）

- load(context, sources) (L683-698)：

- 创建 BeanDefinitionLoader(registry, sources)

- 设置 beanNameGenerator、resourceLoader、environment

- loader.load() (BeanDefinitionLoader.java L128-132)：遍历所有 source，按类型分发：

- **Class** → annotatedReader.register(source) — 注册为 BeanDefinition（@Configuration 类）

- **Resource** → xmlReader.loadBeanDefinitions(resource) — 加载 XML 配置

- **Package** → scanner.scan(packageName) — 包扫描 @Component/@Service 等

- **CharSequence** → 先尝试当类名加载，失败再当资源路径，再失败当包名

BeanDefinitionLoader 内部维护三个 reader：

- AnnotatedBeanDefinitionReader — 处理 @Configuration/@Bean 注解

- XmlBeanDefinitionReader — 处理 XML

- GroovyBeanDefinitionReader（如果 Groovy 在 classpath）

- ClassPathBeanDefinitionScanner — 处理包扫描

还有一个 ClassExcludeFilter — 确保指定的 Class source 不会在包扫描时被重复注册。

#### 11.12 listeners.contextLoaded(context)

L418

⚡ 发布事件：ApplicationPreparedEvent

EventPublishingRunListener.contextLoaded() (L92-100)：

- 遍历 application.getListeners()：

- 如果是 ApplicationContextAware → setApplicationContext(context)

- context.addApplicationListener(listener) — 把 listener 注册到 context

- multicastInitialEvent(new ApplicationPreparedEvent(...))

此后所有事件都通过 ApplicationContext 的 ApplicationEventMulticaster 广播，不再用 initialMulticaster。

### 阶段 E：刷新 Context（步骤 12-14）

> **【阶段】**

**12.**`refreshContext(context) L321 / L441-446`

#### 12.1 注册 ShutdownHook

L442-444

shutdownHook.registerApplicationContext(context) (SpringApplicationShutdownHook.java L78-85)：

- addRuntimeShutdownHookIfNecessary() (L87-91)：

- 如果 shutdownHookAdditionEnabled 且 shutdownHookAdded CAS(false→true) 成功

- Runtime.getRuntime().addShutdownHook(new Thread(this, "SpringApplicationShutdownHook"))

- 给 context 加 ApplicationContextClosedListener（监听 ContextClosedEvent，把 context 从 contexts 移到 closedContexts）

- 把 context 加入 contexts 集合

ShutdownHook 的 run() (L105-119) 在 JVM 关闭时执行：

- 关闭所有注册的 context（closeAndWait()，最多等 10 分钟）

- 关闭所有 closedContexts

- 逆序执行所有注册的 Handler（SpringApplicationShutdownHandlers.add() 添加的）

#### 12.2 context.refresh()

L445 → L756

applicationContext.refresh() — 进入 Spring Framework 的 AbstractApplicationContext.refresh()，这是整个 Spring 的核心方法：

// AbstractApplicationContext.refresh() 内部流程（Spring Framework 源码，非 Spring Boot）

prepareRefresh() — 初始化前准备
  initPropertySources() — 初始化属性源
  getEnvironment().validateRequiredProperties() — 校验必需属性

obtainFreshBeanFactory() — 获取 BeanFactory
  refreshBeanFactory() — GenericApplicationContext 创建 DefaultListableBeanFactory
  getBeanFactory() — 返回 BeanFactory

prepareBeanFactory(beanFactory) — 配置 BeanFactory
  setBeanClassLoader() — 设置类加载器
  addBeanPostProcessor(new ApplicationContextAwareProcessor) — 处理 Aware 接口注入
  ignoreDependencyInterface(BeanNameAware.class) — 忽略 Aware 接口的自动注入
  registerResolvableDependency(BeanFactory.class, this) — 注册可解析依赖
  addBeanPostProcessor(new ApplicationListenerDetector) — 检测 ApplicationListener Bean

postProcessBeanFactory(beanFactory) — 子类扩展点（空实现）

invokeBeanFactoryPostProcessors(beanFactory) ★ 最核心步骤
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()
    先执行 BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry()
      ConfigurationClassPostProcessor — 处理 @Configuration 类
        → 解析 @ComponentScan → 包扫描注册 BeanDefinition
        → 解析 @Import → 导入 AutoConfigurationImportSelector
          → 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
          → 批量注册自动配置类的 BeanDefinition
        → 解析 @Bean 方法 → 注册为 BeanDefinition
        → 解析 @ImportResource → 加载 XML
    再执行 BeanFactoryPostProcessor.postProcessBeanFactory()
      ConfigurationClassPostProcessor — 增强 @Configuration 类（CGLIB 代理）
      PropertySourceOrderingBeanFactoryPostProcessor — 调整属性源顺序
      LazyInitializationBeanFactoryPostProcessor — 设置 Bean 为懒加载

registerBeanPostProcessors(beanFactory)
  PostProcessorRegistrationDelegate.registerBeanPostProcessors()
    注册 PriorityOrdered BeanPostProcessor
    注册 Ordered BeanPostProcessor
    注册普通 BeanPostProcessor
    注册 MergedBeanDefinitionPostProcessor
    将 ApplicationListenerDetector 重新注册到最后

initMessageSource() — 初始化国际化消息源

initApplicationEventMulticaster() — 初始化事件广播器
  SimpleApplicationEventMulticaster

onRefresh() ★ 创建嵌入式 Web 服务器（详见专题三）
  ServletWebServerApplicationContext.onRefresh() [ServletWebServerApplicationContext:161]
    createWebServer() [ServletWebServerApplicationContext:183]
      getWebServerFactory() [ServletWebServerApplicationContext:214]
        // → getBeanNamesForType(ServletWebServerFactory) → getBean("tomcatServletWebServerFactory")
      factory.getWebServer(getSelfInitializer()) [TomcatServletWebServerFactory:163]
        // getSelfInitializer() = WebApplicationContextInitializer(this)::initialize
        createTomcat(tempDirs) [TomcatWebServerFactory:389]
          // → new Tomcat() → 内部创建 Server/Service/Engine/Host
          // → new Connector("Http11NioProtocol") → NioEndpoint + CoyoteAdapter
          // → customizeConnector: setPort(8080) + SSL + 压缩 + HTTP2
          // → configureEngine: setBackgroundProcessorDelay + addValve
        prepareContext(host, initializers, tempDirs) [TomcatServletWebServerFactory:189]
          // → new TomcatEmbeddedContext() → 设 DocBase/Resources/Locale/TLD
          // → new WebappLoader(TomcatEmbeddedWebappClassLoader) → delegate=true
          // → addDefaultServlet("/") + addJspServlet("*.jsp")
          // → configureContext: Session/Cookie/ErrorPage/MimeMappings/Valves
          // → host.addChild(context)
        getTomcatWebServer(tomcat) [TomcatServletWebServerFactory:428]
          new TomcatWebServer(tomcat, autoStart, shutdown) [TomcatWebServer:105]
            initialize() [TomcatWebServer:113] ★ Tomcat 初始化
              // → addInstanceIdToEngineName
              // → context.addLifecycleListener → START_EVENT 时 removeServiceConnectors
              //   ★ 移除 Connector，让 tomcat.start() 不绑定端口
              // → disableBindOnInit → bindOnInit=false
              // → tomcat.start() → Server→Service→Engine→Host→Context 启动
              //   → Context START_EVENT → removeServiceConnectors 回调
              //   → ServletContainerInitializer.onStartup → 注册 DispatcherServlet/Filter
              //   → loadOnStartup: DefaultServlet + DispatcherServlet + JspServlet
              // → rethrowDeferredStartupExceptions
              // → startNonDaemonAwaitThread → 非守护线程保持 JVM 存活
      registerSingleton("webServerStartStop", new WebServerStartStopLifecycle) [ServletWebServerApplicationContext:195]
        // ★ SmartLifecycle，在 finishRefresh() 中被 start() → 真正绑定端口

registerListeners() — 注册事件监听器
  把 ApplicationListener Bean 注册到 multicaster
  注册早期事件

finishBeanFactoryInstantiation() ★ 实例化所有非懒加载单例 Bean
  beanFactory.preInstantiateSingletons()
    遍历所有 BeanDefinition（按名字排序）
      getBean(beanName) → 创建 Bean 实例
        createBean() // AbstractAutowireCapableBeanFactory L518
          resolveBeforeInstantiation() // L1136: 实例化前
            // 给 InstantiationAwareBeanPostProcessor 机会返回代理替代真实 Bean
            // AbstractAutoProxyCreator.postProcessBeforeInstantiation() L273
            //   → 只有自定义 TargetSource 才返回代理，否则返回 null
            //   → 如果返回非 null → 跳过 doCreateBean，直接 applyBeanPostProcessorsAfterInitialization
          doCreateBean() // L560: 核心创建流程
            createBeanInstance() // L1188: 反射创建实例
              // → getInstanceSupplier() / instantiateUsingFactoryMethod() / autowireConstructor() / instantiateBean()
              // → 得到 raw bean（原始对象，还没注入属性）
            applyMergedBeanDefinitionPostProcessors() // L581: 处理 @Autowired/@Value 注入元数据
              // → AutowiredAnnotationBeanPostProcessor.determineCandidateConstructors()
              //   解析 @Autowired/@Value 字段和方法，缓存 InjectionMetadata
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))
              // L600 → DefaultSingletonBeanRegistry L183
              // ★ 三级缓存：singletonFactories.put(beanName, lambda)
              //   lambda 此时未执行！只有循环依赖时才被调用
              //   条件：mbd.isSingleton() && allowCircularReferences && isSingletonCurrentlyInCreation(beanName)
            populateBean() // L1405: 属性注入
              // 1. InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation() — 是否继续注入
              // 2. autowireByName() / autowireByType() — 按名字/类型查找依赖 Bean
              //    → getBean(dependencyBeanName) → 可能触发循环依赖！
              // 3. InstantiationAwareBeanPostProcessor.postProcessProperties() — @Autowired/@Value 注入
              //    → AutowiredAnnotationBeanPostProcessor.inject() → resolveDependency() → getBean()
              // 4. applyPropertyValues() — 应用属性值
              // ★ 循环依赖在此发生：getBean(B) → B 创建 → B populateBean → getBean(A)
              //   → getSingleton(A, allowEarlyReference=true) → 三级缓存 lambda 执行
              //   → getEarlyBeanReference() → 可能提前创建 AOP 代理
              //   → 返回 A 的早期引用给 B
            initializeBean() // L1813: 初始化
              invokeAwareMethods() // L1835: BeanNameAware/BeanClassLoaderAware/BeanFactoryAware
              applyBeanPostProcessorsBeforeInitialization() // L424: @PostConstruct
                // → CommonAnnotationBeanPostProcessor.postProcessBeforeInitialization()
                //   → 处理 @PostConstruct、@Resource
              invokeInitMethods() // L1865: 初始化方法
                // → InitializingBean.afterPropertiesSet()  // L1873
                // → invokeCustomInitMethod()  // @Bean(initMethod="xxx")  // L1897
              applyBeanPostProcessorsAfterInitialization() // L440: AOP 代理
                // ★ AbstractAutoProxyCreator.postProcessAfterInitialization() L315
                //   → if earlyBeanReferences.remove(cacheKey) != bean
                //     → 没有提前代理过 → wrapIfNecessary() → createProxy() → 返回代理
                //   → if == bean → 已提前代理过（循环依赖时）→ 跳过，避免重复代理
            earlySingletonExposure 检查 // L618-643
              // → getSingleton(beanName, false) — 查二级缓存
              // → 如果有早期引用（循环依赖发生过）→ exposedObject = earlySingletonReference
              // → 保证 B 拿到的 A 和最终暴露的 A 是同一个对象
            registerDisposableBeanIfNecessary() // L647
          // → afterSingletonCreation(beanName) — 从 singletonsCurrentlyInCreation 移除
          // → addSingleton(beanName, singletonObject) — 放入一级缓存，清理二/三级缓存

finishRefresh() — 刷新完成
  clearResourceCaches()
  LifecycleProcessor.onRefresh() → 遍历 SmartLifecycle Bean → start()
    WebServerStartStopLifecycle.start() [WebServerStartStopLifecycle:43] ★ 真正启动 Tomcat
      webServer.start() [TomcatWebServer:226]
        // → addPreviouslyRemovedConnectors → 加回 Connector → 绑定端口
        // → NioEndpoint.start → ServerSocketChannel.bind(8080) ★ 端口绑定
        // → performDeferredLoadOnStartup → 补加载 Servlet
        // → checkThatConnectorsHaveStarted → 检查状态
        // → "Tomcat started on port 8080 (http)"
      publishEvent(new ServletWebServerInitializedEvent) ⚡
  registerApplicationContextAsSingleton() — 把 context 注册到 BeanFactory
  publishEvent(new ContextRefreshedEvent(this)) ⚡
  注册 LiveBeansView（如果启用）

**13.**`afterRefresh(context, applicationArguments) L322 / L764-765`

空实现，模板方法，子类可覆盖做额外处理。

**14.**`startup.started() L323 / L1754-1758`

Duration.ofMillis(System.currentTimeMillis() - startTime)

计算从启动到 context 刷新完成的总耗时。

### 阶段 F：启动后处理（步骤 15-20）

> **【阶段】**

**15.**`logStarted() L324-326 / StartupInfoLogger.java L58-98`

如果 properties.isLogStartupInfo()（默认 true）：

new StartupInfoLogger(mainApplicationClass, environment).logStarted(getApplicationLog(), startup)

输出："Started MyApplication in 2.345 seconds (process running for 3.456)"

startup.action() 返回 "Started"（StandardStartup）或 "Restored"（CRaC 恢复）。

**16.**`listeners.started(context, timeTakenToStarted) L327`

⚡ 发布事件：ApplicationStartedEvent

⚡ 发布事件：AvailabilityChangeEvent(LivenessState.CORRECT)

EventPublishingRunListener.started() (L103-106)：

- context.publishEvent(new ApplicationStartedEvent(...)) — 此时用 context 的 multicaster

- AvailabilityChangeEvent.publish(context, LivenessState.CORRECT) — 标记应用存活状态为 CORRECT

**17.**`callRunners(context, applicationArguments) L328 / L767-777`

查找并调用所有 Runner Bean：

- beanFactory.getBeanNamesForType(Runner.class) — 查找所有 Runner 类型 Bean

- 遍历获取实例，放入 IdentityHashMap<Runner, String>

- 获取排序器：getOrderComparator(beanFactory) (L779-784)：

- 如果 beanFactory 是 DefaultListableBeanFactory → 用其 dependencyComparator（如果它是 OrderComparator）

- 否则 → AnnotationAwareOrderComparator.INSTANCE

- 包装 FactoryAwareOrderSourceProvider — 从 BeanDefinition 获取工厂方法和目标类型作为排序源

- 排序后遍历调用 callRunner(runner, args) (L786-794)：

- 如果是 ApplicationRunner → applicationRunner.run(args)（args 是 ApplicationArguments）

- 如果是 CommandLineRunner → commandLineRunner.run(args.getSourceArgs())（args 是 String[]）

Runner 的执行顺序由 @Order 或 Ordered 接口决定。ApplicationRunner 和 CommandLineRunner 混在一起排序，不区分类型优先级。如果同一个类同时实现了两个接口，两个 run 方法都会被调用。

**18.**`context.isRunning() + `listeners.ready(context, startup.ready()) L333-336

检查 context 是否在运行，如果是：
⚡ 发布事件：ApplicationReadyEvent

⚡ 发布事件：AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC)

EventPublishingRunListener.ready() (L109-112)：

- context.publishEvent(new ApplicationReadyEvent(...))

- AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC) — 标记应用就绪，可以接收流量

startup.ready() 计算从启动到 ready 的总耗时。

**19.**`return context L341`

返回运行中的 ConfigurableApplicationContext。

**20.**`handleRunFailure(context, ex, listeners) L330-331 / L803-828`

**异常处理（catch 块）**：

- 如果是 AbandonedRunException → 直接返回，不处理

- handleExitCode() (L881-892)：

- 从异常中提取 ExitCode（通过 ExitCodeExceptionMapper Bean 或 ExitCodeGenerator 接口）

- 发布 ExitCodeEvent

- 注册 exitCode 到 SpringBootExceptionHandler

- listeners.failed(context, exception) — 触发 RunListener 的 failed

  ⚡ 发布事件：ApplicationFailedEvent

  EventPublishingRunListener.failed() (L115-133)：

- 如果 context 活跃 → context.publishEvent(event)

- 如果 context 不活跃 → 把 context 的 listeners 加到 initialMulticaster，用 initialMulticaster 广播

- reportFailure() (L842-867) — 加载 SpringBootExceptionReporter（从 spring.factories），报告异常

- context.close() — 关闭 Context

- shutdownHook.deregisterFailedApplicationContext(context) — 从 ShutdownHook 注销失败的 context

## 专题一：循环依赖注入（三级缓存机制）

> **【阶段】**

源码位置：DefaultSingletonBeanRegistry.java + AbstractAutowireCapableBeanFactory.java（Spring Framework 6.2.18）

### 三级缓存结构

```
DefaultSingletonBeanRegistry 内部维护三个 Map：

// 一级缓存：完整的单例 Bean（初始化完毕）
private final Map singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：早期单例引用（半成品，还没初始化完）
private final Map earlySingletonObjects = new ConcurrentHashMap<>(16);

// 三级缓存：单例工厂（lambda，调用时才生成早期引用）
private final Map> singletonFactories = new ConcurrentHashMap<>(16);
```

三级缓存存在的意义：

1. **singletonObjects** — 存放完全初始化好的 Bean，所有 getBean 请求最终都从这里取

2. **earlySingletonObjects** — 存放"半成品"引用，Bean 实例已创建但还没 populateBean + initializeBean，用于解决循环依赖

3. **singletonFactories** — 存放 ObjectFactory（lambda），**延迟执行** getEarlyBeanReference，只有真正发生循环依赖时才调用，避免不必要的提前代理

### 循环依赖完整流程（以 A 依赖 B、B 依赖 A 为例）

```
假设：A 的属性上有 @Autowired B，B 的属性上有 @Autowired A

步骤1: getBean("A")
  └─ getSingleton("A", singletonFactory)                    // DefaultSingletonBeanRegistry L257
      ├─ singletonObjects.get("A") == null                   // 一级缓存没有
      ├─ beforeSingletonCreation("A")                         // L542: singletonsCurrentlyInCreation.add("A")
      │                                                       // 标记 A 正在创建中
      └─ singletonFactory.getObject()                        // L373: 回调到 AbstractAutowireCapableBeanFactory
          └─ createBean("A", mbd, args)                      // L518
              └─ doCreateBean("A", mbd, args)                // L560
                  │
                  ├─ ① createBeanInstance("A", mbd, args)     // L569: 反射创建 A 的实例
                  │   └─ 得到 A 的原始对象（raw bean）
                  │
                  ├─ ② applyMergedBeanDefinitionPostProcessors // L581: 处理 @Autowired/@Value 注入元数据
                  │   └─ AutowiredAnnotationBeanPostProcessor
                  │       └─ determineCandidateConstructors() → 解析 @Autowired 字段/方法
                  │
                  ├─ ③ addSingletonFactory("A", () -> getEarlyBeanReference("A", mbd, bean))
                  │   // L600: DefaultSingletonBeanRegistry L183
                  │   // 把 A 的 ObjectFactory 放入三级缓存 singletonFactories
                  │   // 注意：lambda 此时还没执行！
                  │   // singletonFactories.put("A", lambda)
                  │   // earlySingletonObjects.remove("A")
                  │
                  ├─ ④ populateBean("A", mbd, instanceWrapper) // L606: 属性注入
                  │   └─ AutowiredAnnotationBeanPostProcessor.postProcessProperties()
                  │       └─ 解析 @Autowired B
                  │       └─ resolveDependency() → getBean("B")  // 需要注入 B
                  │           │
                  │           └─ getSingleton("B", singletonFactory)
                  │               ├─ singletonObjects.get("B") == null
                  │               ├─ beforeSingletonCreation("B")
                  │               └─ singletonFactory.getObject() → createBean("B")
                  │                   └─ doCreateBean("B")
                  │                       ├─ createBeanInstance("B")     // 反射创建 B
                  │                       ├─ addSingletonFactory("B", lambda)  // B 的工厂放入三级缓存
                  │                       ├─ populateBean("B")          // B 的属性注入
                  │                       │   └─ @Autowired A → getBean("A")  // B 需要 A！循环依赖发生！
                  │                       │       │
                  │                       │       └─ getSingleton("A", allowEarlyReference=true)
                  │                       │           // DefaultSingletonBeanRegistry L210
                  │                       │           ├─ singletonObjects.get("A") == null    // 一级没有
                  │                       │           ├─ isSingletonCurrentlyInCreation("A") == true  // A 正在创建
                  │                       │           ├─ earlySingletonObjects.get("A") == null  // 二级也没有
                  │                       │           └─ singletonFactories.get("A") != null  // 三级有！
                  │                       │               └─ singletonFactory.getObject()     // L228: 执行 lambda
                  │                       │                   └─ getEarlyBeanReference("A", mbd, bean)
                  │                       │                       // AbstractAutowireCapableBeanFactory L975
                  │                       │                       └─ 遍历 SmartInstantiationAwareBeanPostProcessor
                  │                       │                           └─ AbstractAutoProxyCreator.getEarlyBeanReference()
                  │                       │                               // L265: earlyBeanReferences.put(cacheKey, bean)
                  │                       │                               // L268: wrapIfNecessary(bean, beanName, cacheKey)
                  │                       │                               //   → 如果需要 AOP 代理，此时创建早期代理！
                  │                       │                               //   → 如果不需要代理，返回原始 bean
                  │                       │                       └─ 返回 A 的早期引用（可能是代理对象）
                  │                       │               └─ singletonFactories.remove("A")     // L230: 从三级缓存移除
                  │                       │               └─ earlySingletonObjects.put("A", obj) // L231: 放入二级缓存
                  │                       │               └─ 返回 A 的早期引用给 B
                  │                       │       └─ B 拿到 A 的早期引用，注入到 B 的 @Autowired A 字段
                  │                       │
                  │                       ├─ initializeBean("B")          // B 的初始化
                  │                       │   ├─ invokeAwareMethods
                  │                       │   ├─ applyBeanPostProcessorsBeforeInitialization  // @PostConstruct
                  │                       │   ├─ invokeInitMethods  // afterPropertiesSet + initMethod
                  │                       │   └─ applyBeanPostProcessorsAfterInitialization   // AOP 代理（如果需要）
                  │                       └─ B 创建完成
                  │               └─ addSingleton("B", singletonObject)  // B 放入一级缓存
                  │               └─ afterSingletonCreation("B")        // 从 singletonsCurrentlyInCreation 移除 B
                  │           └─ 返回 B 给 A 的 populateBean
                  │       └─ A 拿到 B，注入到 A 的 @Autowired B 字段
                  │
                  ├─ ⑤ initializeBean("A", exposedObject, mbd)  // L607: A 的初始化
                  │   ├─ invokeAwareMethods(beanName, bean)
                  │   │   ├─ BeanNameAware.setBeanName()
                  │   │   ├─ BeanClassLoaderAware.setBeanClassLoader()
                  │   │   └─ BeanFactoryAware.setBeanFactory()
                  │   ├─ applyBeanPostProcessorsBeforeInitialization(bean, beanName)
                  │   │   └─ CommonAnnotationBeanPostProcessor → @PostConstruct
                  │   ├─ invokeInitMethods(beanName, bean, mbd)
                  │   │   ├─ InitializingBean.afterPropertiesSet()  // 如果实现了
                  │   │   └─ invokeCustomInitMethod()  // @Bean(initMethod="xxx")
                  │   └─ applyBeanPostProcessorsAfterInitialization(bean, beanName)
                  │       └─ AbstractAutoProxyCreator.postProcessAfterInitialization()
                  │           // L315: 检查 earlyBeanReferences
                  │           // L318: if earlyBeanReferences.remove(cacheKey) != bean
                  │           //   → 说明没有提前代理过（没发生循环依赖），正常创建代理
                  │           //   → 如果 == bean，说明已经提前代理过（循环依赖时），跳过！
                  │           //   → wrapIfNecessary() → createProxy() → 返回代理对象
                  │
                  ├─ ⑥ earlySingletonExposure 检查  // L618-643
                  │   └─ getSingleton("A", false)  // allowEarlyReference=false
                  │       ├─ 如果二级缓存有（说明发生了循环依赖，B 拿过 A 的早期引用）
                  │       │   └─ earlySingletonReference != null
                  │       │       └─ if exposedObject == bean  // 没被 initializeBean 改变
                  │       │           └─ exposedObject = earlySingletonReference  // 用早期引用替换！
                  │       │           // 保证 B 拿到的 A 和最终暴露的 A 是同一个对象
                  │       └─ 如果二级缓存没有（没发生循环依赖）
                  │           └─ earlySingletonReference == null → 正常流程
                  │
                  └─ ⑦ registerDisposableBeanIfNecessary  // L647
      └─ afterSingletonCreation("A")  // L401: singletonsCurrentlyInCreation.remove("A")
      └─ addSingleton("A", singletonObject)  // L406: 放入一级缓存
          // singletonObjects.put("A", obj)
          // singletonFactories.remove("A")
          // earlySingletonObjects.remove("A")
          // registeredSingletons.add("A")
```

**为什么用三级缓存而不是二级缓存？**

如果只有二级缓存（singletonObjects + earlySingletonObjects），那么在 doCreateBean 时就必须立即调用 getEarlyBeanReference()，把半成品代理放入二级缓存。但问题是：**不是所有 Bean 都会发生循环依赖**，提前为每个 Bean 创建代理是浪费的。

三级缓存用 ObjectFactory（lambda）延迟了这个过程：只有当另一个 Bean 真正来 getSingleton("A", allowEarlyReference=true) 时，lambda 才被执行，代理才被创建。没有循环依赖时，lambda 永远不执行，代理在 initializeBean 的 postProcessAfterInitialization 中正常创建。

### 三级缓存状态变化图

```
时刻    操作                          singletonObjects    earlySingletonObjects    singletonFactories
────    ────                          ────────────────    ──────────────────────    ──────────────────
T1      getBean("A") 开始             {}                  {}                       {}
T2      createBeanInstance("A")       {}                  {}                       {}
T3      addSingletonFactory("A")      {}                  {}                       {A: lambda}
T4      populateBean("A") → getBean("B")  {}              {}                       {A: lambda}
T5      createBeanInstance("B")       {}                  {}                       {A: lambda, B: lambda}
T6      populateBean("B") → getBean("A")  循环依赖！
T7      getSingleton("A", true)       {}                  {}                       {A: lambda, B: lambda}
T8      执行 A 的 lambda → 早期引用   {}                  {A: 早期引用}            {B: lambda}
T9      B 注入 A 的早期引用           {}                  {A: 早期引用}            {B: lambda}
T10     initializeBean("B")           {}                  {A: 早期引用}            {B: lambda}
T11     addSingleton("B")             {B: 完整}           {A: 早期引用}            {}
T12     A 注入 B                       {B: 完整}           {A: 早期引用}            {}
T13     initializeBean("A")           {B: 完整}           {A: 早期引用}            {}
T14     earlySingletonExposure 检查    {B: 完整}           {A: 早期引用}            {}
        exposedObject = earlySingletonReference
T15     addSingleton("A")             {B: 完整, A: 完整}  {}                       {}
```

### 循环依赖的限制条件

- **必须 isSingleton** — 原型（prototype）Bean 不缓存，无法解决循环依赖，直接抛 BeanCurrentlyInCreationException

- **必须 allowCircularReferences=true** — Spring Boot 默认 false（AbstractAutowireCapableBeanFactory L593），需显式开启 spring.main.allow-circular-references=true

- **构造器注入无法解决** — 循环依赖发生在 createBeanInstance 阶段（构造器需要参数），此时还没到 addSingletonFactory，三级缓存中没有 Bean，直接报错

- **只能解决单例 setter/字段注入** — @Autowired 字段注入和 setter 注入发生在 populateBean 阶段，此时三级缓存已就绪

## 专题二：动态代理（AOP 代理创建机制）

> **【阶段】**

源码位置：AbstractAutoProxyCreator.java + DefaultAopProxyFactory.java + JdkDynamicAopProxy.java + CglibAopProxy.java + ConfigurationClassPostProcessor.java（Spring Framework 6.2.18）

### Spring 中两种动态代理

```
Spring AOP 创建代理的入口：DefaultAopProxyFactory.createAopProxy()

// DefaultAopProxyFactory.java L60-76
public AopProxy createAopProxy(AdvisedSupport config) {
    // 判断条件：optimize=true || proxyTargetClass=true || 没有用户提供的接口
    if (config.isOptimize() || config.isProxyTargetClass() || !config.hasUserSuppliedInterfaces()) {
        Class targetClass = config.getTargetClass();
        if (targetClass == null || targetClass.isInterface() ||
                Proxy.isProxyClass(targetClass) || ClassUtils.isLambdaClass(targetClass)) {
            return new JdkDynamicAopProxy(config);    // 目标类是接口 → JDK 代理
        }
        return new ObjenesisCglibAopProxy(config);     // 目标类是普通类 → CGLIB 代理
    }
    else {
        return new JdkDynamicAopProxy(config);         // 有接口且未强制 CGLIB → JDK 代理
    }
}
```

**代理选择规则：**

1. 目标类实现了接口 + 未设置 proxyTargetClass → **JDK 动态代理**（基于接口）

2. 目标类没实现接口 → **CGLIB 代理**（基于子类继承）

3. 设置了 proxyTargetClass=true → **CGLIB 代理**（强制）

4. 目标类本身是接口 → **JDK 动态代理**（即使 proxyTargetClass=true）

5. Spring Boot 2.x 起默认 proxyTargetClass=true → 倾向 CGLIB

### 代理创建的三个时机

- **postProcessBeforeInstantiation** — 实例化前（仅当有自定义 TargetSource 时，极少用）

- **getEarlyBeanReference** — 循环依赖时提前创建代理（三级缓存触发）

- **postProcessAfterInitialization** — 初始化后创建代理（正常流程，最常见）

### 时机 1：postProcessBeforeInstantiation（实例化前）

```
// AbstractAutoProxyCreator.java L273-301
public Object postProcessBeforeInstantiation(Class beanClass, String beanName) {
    Object cacheKey = getCacheKey(beanClass, beanName);
    // 已处理过 → 跳过
    if (this.advisedBeans.containsKey(cacheKey)) return null;
    // 基础设施类（Advice/Advisor/Pointcut）→ 跳过
    if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return null;
    }
    // 只有自定义 TargetSource 才走这条路（极少用）
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        return proxy;  // 返回非 null → 跳过正常 createBean 流程！
    }
    return null;  // 正常情况返回 null，走 doCreateBean
}
```

### 时机 2：getEarlyBeanReference（循环依赖时提前代理）

```
// AbstractAutoProxyCreator.java L265-269
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyBeanReferences.put(cacheKey, bean);  // 记录原始 bean
    return wrapIfNecessary(bean, beanName, cacheKey);  // 如果需要，创建代理
}

// wrapIfNecessary L354-378
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (this.targetSourcedBeans.contains(beanName)) return bean;
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) return bean;
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    // 查找匹配的 Advisor（切面、通知）
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors,
                                    new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;  // 返回代理对象
    }
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;  // 不需要代理
}
```

### 时机 3：postProcessAfterInitialization（初始化后代理，最常见）

```
// AbstractAutoProxyCreator.java L315-323
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // 关键判断：earlyBeanReferences 中是否记录过这个 bean
        if (this.earlyBeanReferences.remove(cacheKey) != bean) {
            // != bean 说明：没有提前代理过（没发生循环依赖）
            // → 正常创建代理
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
        // == bean 说明：已经通过 getEarlyBeanReference 提前代理过（循环依赖时）
        // → 跳过！避免重复代理
    }
    return bean;
}
```

**避免重复代理的关键：** earlyBeanReferences Map 记录了哪些 Bean 在循环依赖时被提前代理过。postProcessAfterInitialization 时检查：如果 earlyBeanReferences 里有记录且是同一个原始 bean → 说明已经代理过，直接返回原始 bean（代理对象已经在二级缓存中）。如果没有记录 → 正常创建代理。

### createProxy → buildProxy 完整流程

```
// AbstractAutoProxyCreator.java L474-518
private Object buildProxy(Class beanClass, String beanName,
        Object[] specificInterceptors, TargetSource targetSource, boolean classOnly) {

    // 1. 暴露目标类（让 @Configuration 类的 @Bean 方法能被正确拦截）
    if (this.beanFactory instanceof ConfigurableListableBeanFactory clbf) {
        AutoProxyUtils.exposeTargetClass(clbf, beanName, beanClass);
    }

    // 2. 创建 ProxyFactory，复制配置
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);  // 复制 proxyTargetClass、optimize 等配置

    // 3. 判断代理方式
    if (proxyFactory.isProxyTargetClass()) {
        // proxyTargetClass=true → CGLIB
        // 但如果目标本身是 JDK 代理或 lambda → 提取接口用 JDK 代理
        if (Proxy.isProxyClass(beanClass) || ClassUtils.isLambdaClass(beanClass)) {
            for (Class ifc : beanClass.getInterfaces()) {
                proxyFactory.addInterface(ifc);
            }
        }
    } else {
        // 检查是否应该用 CGLIB
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        } else {
            evaluateProxyInterfaces(beanClass, proxyFactory);  // 提取接口 → JDK 代理
        }
    }

    // 4. 构建 Advisor 链（切面 + 通知）
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);
    proxyFactory.setFrozen(this.freezeProxy);  // 冻结 Advisor 配置
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // 5. 创建代理对象
    // ProxyFactory.getProxy() → ProxyCreatorSupport.createAopProxy()
    //   → DefaultAopProxyFactory.createAopProxy()
    //     → new JdkDynamicAopProxy(config) 或 new ObjenesisCglibAopProxy(config)
    //       → .getProxy(classLoader)
    return proxyFactory.getProxy(classLoader);
}
```

### JDK 动态代理：JdkDynamicAopProxy

```
// JdkDynamicAopProxy.java L120-125
public Object getProxy(@Nullable ClassLoader classLoader) {
    // Proxy.newProxyInstance → 生成实现了目标接口的代理类
    // this = InvocationHandler，所有方法调用都走 invoke()
    return Proxy.newProxyInstance(determineClassLoader(classLoader),
            this.cache.proxiedInterfaces, this);
}

// L167-256: invoke() 方法 — 每次调用代理对象方法时执行
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 1. equals/hashCode → 代理自身处理
    // 2. DecoratingProxy.getDecoratedClass() → 返回目标类
    // 3. Advised 接口方法 → 直接反射调用 advised 配置
    // 4. exposeProxy=true → AopContext.setCurrentProxy(proxy)

    target = targetSource.getTarget();  // 获取目标对象
    targetClass = target.getClass();

    // 5. 获取拦截器链（Advisor → MethodInterceptor）
    List chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

    if (chain.isEmpty()) {
        // 没有拦截器 → 直接反射调用目标方法
        retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
    } else {
        // 有拦截器 → 创建 ReflectiveMethodInvocation，执行拦截器链
        MethodInvocation invocation = new ReflectiveMethodInvocation(
                proxy, target, method, args, targetClass, chain);
        retVal = invocation.proceed();  // 责任链模式：依次执行 before → 目标方法 → after
    }

    return retVal;
}
```

### CGLIB 代理：ObjenesisCglibAopProxy

```
// CglibAopProxy.java L176-246
private Object buildProxy(@Nullable ClassLoader classLoader, boolean classOnly) {
    Class rootClass = this.advised.getTargetClass();
    Class proxySuperClass = rootClass;
    // 如果目标类已经是 CGLIB 代理类 → 取父类作为代理父类
    if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
        proxySuperClass = rootClass.getSuperclass();
    }

    // 1. 创建 CGLIB Enhancer
    Enhancer enhancer = createEnhancer();
    enhancer.setClassLoader(classLoader);
    enhancer.setSuperclass(proxySuperClass);           // 设置父类（目标类）
    enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);

    // 2. 获取 Callback 数组
    //    核心：DynamicAdvisedInterceptor（等价于 JDK 的 invoke）
    //    + EqualsInterceptor、HashCodeInterceptor、StaticUnadvisedInterceptor 等
    Callback[] callbacks = getCallbacks(rootClass);
    Class[] types = new Class[callbacks.length];
    for (int x = 0; x  getEarlyBeanReference("A", mbd, bean))
   → lambda 未执行，三级缓存存的是 lambda

2. A populateBean → 注入 B → getBean("B")
   → B 创建 → addSingletonFactory("B", lambda)
   → B populateBean → 注入 A → getSingleton("A", allowEarlyReference=true)
   → 三级缓存有 A 的 lambda → 执行！
   → getEarlyBeanReference("A", mbd, bean)
     → AbstractAutoProxyCreator.getEarlyBeanReference()
       → earlyBeanReferences.put(cacheKey, bean)  // 记录原始 bean
       → wrapIfNecessary(bean, beanName, cacheKey)
         → getAdvicesAndAdvisorsForBean() → 找到 @Transactional 的 Advisor
         → createProxy() → 创建 A 的 AOP 代理
       → 返回 A 的代理对象
   → 二级缓存 earlySingletonObjects.put("A", 代理对象)
   → B 拿到 A 的代理对象注入

3. B initializeBean → postProcessAfterInitialization
   → AbstractAutoProxyCreator.postProcessAfterInitialization()
   → earlyBeanReferences.remove(cacheKey) != bean  // B 没有提前代理过
   → wrapIfNecessary() → 创建 B 的代理

4. A populateBean 完成 → initializeBean
   → postProcessAfterInitialization
   → earlyBeanReferences.remove(cacheKey) == bean  // A 已经提前代理过！
   → 跳过 wrapIfNecessary，返回原始 bean

5. earlySingletonExposure 检查
   → getSingleton("A", false) → 二级缓存有 → 早期代理对象
   → exposedObject == bean → exposedObject = earlySingletonReference
   → 最终 A 暴露的是早期创建的代理对象

结果：B 中的 A 和最终暴露的 A 是同一个代理对象 ✓
```

## 专题三：Spring Boot 启动 Tomcat 全流程

> **【阶段】**

源码位置：ServletWebServerApplicationContext.java + TomcatServletWebServerFactory.java + TomcatWebServerFactory.java + TomcatWebServer.java + WebServerStartStopLifecycle.java + WebApplicationContextInitializer.java

### 触发入口

Tomcat 的启动发生在 context.refresh() 的 onRefresh() 阶段（见完整调用链树形图中的 onRefresh() ★ 创建嵌入式 Web 服务器）。
ServletWebServerApplicationContext 重写了 onRefresh()，在这里创建并初始化嵌入式 Tomcat。
但此时 Tomcat 只是"初始化"（tomcat.start()），Connector 被临时移除，端口还没绑定。
真正的"启动"（绑定端口、接收请求）发生在 finishRefresh() 阶段，通过 WebServerStartStopLifecycle 的 start() 触发。

### 完整调用链

```
// ===== 阶段 1：onRefresh() — 创建并初始化 Tomcat =====

// ServletWebServerApplicationContext.java L161
onRefresh()
  └─ super.onRefresh()  // GenericWebApplicationContext 空实现
  └─ createWebServer()  // L183

      // 1.1 获取 ServletWebServerFactory Bean
      // L188
      getWebServerFactory()
        ├─ getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class)
        │   └─ 找到 "tomcatServletWebServerFactory"（由 ServletWebServerFactoryAutoConfiguration 注册）
        ├─ beanNames.length == 0 → throw MissingWebServerFactoryBeanException
        ├─ beanNames.length > 1 → throw ApplicationContextException（只能有一个）
        └─ getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class)
            └─ 返回 TomcatServletWebServerFactory 实例

      // 1.2 创建 WebServer（核心！）
      // L190
      webServer = factory.getWebServer(getSelfInitializer())
        │
        │ // getSelfInitializer() L234:
        │ // return new WebApplicationContextInitializer(this)::initialize
        │ // 这是一个 ServletContextInitializer，在 Tomcat 启动时被回调
        │
        └─ TomcatServletWebServerFactory.getWebServer(initializers)  // L163
            │
            ├─ TempDirs tempDirs = new TempDirs(getPort())  // L164: 创建临时目录管理器
            │
            ├─ Tomcat tomcat = createTomcat(tempDirs)  // L165
            │   │  // TomcatWebServerFactory.java L389
            │   │
            │   ├─ Registry.disableRegistry()  // L390: 禁用 JMX MBean 注册（默认）
            │   │
            │   ├─ Tomcat tomcat = new Tomcat()  // L393: 创建 Tomcat 实例
            │   │   └─ 内部创建：
            │   │       ├─ new StandardServer()  → Server（顶层容器）
            │   │       ├─ new StandardService() → Service（服务）
            │   │       └─ new StandardEngine()  → Engine（引擎）
            │   │           └─ new StandardHost() → Host（虚拟主机）
            │   │
            │   ├─ tomcat.setBaseDir(baseDir.getAbsolutePath())  // L395: 设置工作目录
            │   │   └─ 默认创建临时目录 .tomcat.xxx
            │   │
            │   ├─ 添加 Server 级 LifecycleListener  // L396-399
            │   │   ├─ AprLifecycleListener（如果 useApr=true）
            │   │   └─ CleanTempDirsListener（销毁时清理临时目录）
            │   │
            │   ├─ Connector connector = new Connector(getProtocol())  // L400
            │   │   └─ getProtocol() = "org.apache.coyote.http11.Http11NioProtocol"（默认 NIO）
            │   │       └─ 内部创建：
            │   │           ├─ Http11NioProtocol（协议处理器）
            │   │           ├─ NioEndpoint（I/O 端点，负责接收连接）
            │   │           └─ CoyoteAdapter（将 Coyote 请求转为 Servlet 请求）
            │   │
            │   ├─ connector.setThrowOnFailure(true)  // L401: 启动失败抛异常
            │   │
            │   ├─ tomcat.getService().addConnector(connector)  // L402: Connector 加入 Service
            │   │
            │   ├─ customizeConnector(connector)  // L403: 自定义 Connector
            │   │   │  // TomcatWebServerFactory.java L415
            │   │   ├─ connector.setPort(port)  // L417: 设置端口（默认 8080）
            │   │   ├─ connector.setProperty("server", serverHeader)  // L419: Server header
            │   │   ├─ customizeProtocol(abstractProtocol)  // L422: 设置绑定地址
            │   │   │   └─ protocol.setAddress(getAddress())  // 绑定 IP
            │   │   ├─ invokeProtocolHandlerCustomizers()  // L424: 自定义 ProtocolHandler
            │   │   ├─ connector.setURIEncoding("UTF-8")  // L426: URI 编码
            │   │   ├─ http2.isEnabled() → connector.addUpgradeProtocol(new Http2Protocol())  // L428
            │   │   ├─ Ssl.isEnabled(ssl) → customizeSsl(connector, ssl)  // L432: SSL 配置
            │   │   │   └─ new SslConnectorCustomizer() → 配置 SSL 协议、证书、密钥
            │   │   ├─ CompressionConnectorCustomizer → 配置 Gzip 压缩  // L435
            │   │   └─ 遍历 connectorCustomizers → customizer.customize(connector)  // L437
            │   │
            │   ├─ tomcat.setConnector(connector)  // L404: 设为主 Connector
            │   │
            │   ├─ registerConnectorExecutor(tomcat, connector)  // L405: 注册线程池
            │   │   └─ 如果 ProtocolHandler 有 Executor → service.addExecutor(executor)
            │   │
            │   ├─ tomcat.getHost().setAutoDeploy(false)  // L406: 禁用自动部署
            │   │
            │   ├─ configureEngine(tomcat.getEngine())  // L407: 配置 Engine
            │   │   │  // TomcatWebServerFactory.java L479
            │   │   ├─ engine.setBackgroundProcessorDelay(delay)  // 后台线程延迟
            │   │   └─ 遍历 engineValves → engine.getPipeline().addValve(valve)  // 添加 Valve
            │   │
            │   ├─ 遍历 additionalConnectors → service.addConnector(additional)  // L408
            │   │
            │   └─ return tomcat  // L412: 返回配置好的 Tomcat 实例
            │
            ├─ prepareContext(tomcat.getHost(), initializers, tempDirs)  // L166
            │   │  // TomcatServletWebServerFactory.java L189
            │   │
            │   ├─ DocumentRoot documentRoot = new DocumentRoot(logger)  // L190
            │   │   └─ 设置静态资源根目录
            │   │
            │   ├─ TomcatEmbeddedContext context = new TomcatEmbeddedContext()  // L193
            │   │   └─ 嵌入式 Context（不使用 web.xml，不扫描 WEB-INF）
            │   │
            │   ├─ 配置资源根  // L194-197
            │   │   ├─ documentRootFile != null → new LoaderHidingResourceRoot(context)
            │   │   └─ documentRootFile == null → new StandardRoot(context)
            │   │   └─ resourceRoot.setReadOnly(true)  // 只读
            │   │
            │   ├─ context.setResources(resourceRoot)  // L197
            │   ├─ context.setName(contextPath)  // L199
            │   ├─ context.setPath(contextPath)  // L201
            │   ├─ context.setDocBase(docBase.getAbsolutePath())  // L204: 临时 docBase
            │   │
            │   ├─ context.addLifecycleListener(new FixContextListener())  // L205
            │   │   └─ 修复 Context 配置（跳过 web.xml 解析）
            │   │
            │   ├─ context.setParentClassLoader(parentClassLoader)  // L208
            │   │
            │   ├─ resetDefaultLocaleMapping(context)  // L209: 重置 Locale→Charset 映射
            │   ├─ addLocaleMappings(context)  // L210: 添加自定义 Locale 映射
            │   │
            │   ├─ configureTldPatterns(context)  // L212: 配置 TLD 扫描规则
            │   │   └─ StandardJarScanFilter → setTldSkip / setTldScan
            │   │
            │   ├─ WebappLoader loader = new WebappLoader()  // L213
            │   │   └─ loader.setLoaderInstance(new TomcatEmbeddedWebappClassLoader(parent))  // L214
            │   │   └─ loader.setDelegate(true)  // L215: 委托给父类加载器
            │   │   └─ context.setLoader(loader)  // L216
            │   │
            │   ├─ registerDefaultServlet → addDefaultServlet(context)  // L217-219
            │   │   └─ DefaultServlet 映射 "/"，处理静态资源
            │   │
            │   ├─ shouldRegisterJspServlet() → addJspServlet + addJasperInitializer  // L220-223
            │   │   └─ JSP Servlet 映射 "*.jsp" 和 "*.jspx"
            │   │
            │   ├─ context.addLifecycleListener(new StaticResourceConfigurer(context))  // L224
            │   │   └─ 在 BEFORE_INIT_EVENT 时添加静态资源 JAR
            │   │
            │   ├─ ServletContextInitializers initializersToUse = ServletContextInitializers.from(settings, initializers)  // L225
            │   │   └─ 合并配置的 initializers + 传入的 initializers
            │   │
            │   ├─ host.addChild(context)  // L226: Context 加入 Host
            │   │
            │   ├─ configureContext(context, initializersToUse)  // L227
            │   │   │  // TomcatServletWebServerFactory.java L312
            │   │   ├─ DeferredServletContainerInitializers(deferredInitializers)  // L313
            │   │   │   └─ 延迟执行 ServletContainerInitializer（在 Context 启动时才执行）
            │   │   ├─ embeddedContext.setDeferredStartupExceptions(deferredInitializers)  // L316
            │   │   ├─ embeddedContext.setFailCtxIfServletStartFails(true)  // L317
            │   │   ├─ context.addServletContainerInitializer(deferredInitializers, NO_CLASSES)  // L319
            │   │   ├─ 遍历 contextLifecycleListeners → context.addLifecycleListener()  // L320
            │   │   ├─ 遍历 contextValves → context.getPipeline().addValve()  // L323
            │   │   ├─ 遍历 errorPages → context.addErrorPage()  // L326
            │   │   ├─ setMimeMappings(context)  // L333: MIME 类型映射
            │   │   ├─ configureSession(context)  // L334: Session 配置
            │   │   │   ├─ context.setSessionTimeout(timeout)  // 超时时间
            │   │   │   ├─ context.setUseHttpOnly(httpOnly)  // HttpOnly
            │   │   │   └─ persistent → StandardManager + configurePersistSession  // 持久化
            │   │   ├─ configureCookieProcessor(context)  // L335: Cookie 处理器
            │   │   │   └─ SuppliedSameSiteCookieProcessor → 处理 SameSite 属性
            │   │   ├─ DisableReferenceClearingContextCustomizer  // L336
            │   │   ├─ 遍历 webListenerClassNames → context.addApplicationListener()  // L337
            │   │   └─ 遍历 contextCustomizers → customizer.customize(context)  // L340
            │   │
            │   └─ postProcessContext(context)  // L228: 空实现，子类可覆盖
            │
            └─ getTomcatWebServer(tomcat)  // L167
                │  // TomcatServletWebServerFactory.java L428
                └─ return new TomcatWebServer(tomcat, port >= 0, shutdown)
                    │  // TomcatWebServer.java L105
                    │
                    └─ initialize()  // L110-153 ★★★ Tomcat 初始化
                        │
                        ├─ logger.info("Tomcat initialized with port(s) 8080 (http)")  // L114
                        │
                        ├─ addInstanceIdToEngineName()  // L117
                        │   └─ 多实例时给 Engine 名加后缀 "-1"、"-2"
                        │
                        ├─ Context context = findContext()  // L119
                        │   └─ 遍历 Host 的子容器，找到 TomcatEmbeddedContext
                        │
                        ├─ context.addLifecycleListener(event → {  // L120
                        │       if START_EVENT:
                        │           removeServiceConnectors()  // L124 ★ 关键！
                        │           // 移除 Service 中的 Connector，保存到 serviceConnectors Map
                        │           // 目的：让 tomcat.start() 时不绑定端口
                        │           // 端口绑定延迟到 WebServerStartStopLifecycle.start()
                        │   })
                        │
                        ├─ disableBindOnInit()  // L128
                        │   └─ 遍历 Connector → setProperty("bindOnInit", "false")
                        │       // 防止 Connector 在 init 阶段就绑定端口
                        │
                        ├─ this.tomcat.start()  // L131 ★ Tomcat 启动！
                        │   │  // 此时 Connector 已被移除，不会绑定端口
                        │   │  // Tomcat 内部启动流程：
                        │   │
                        │   ├─ Server.start()
                        │   │   ├─ Server.initInternal()
                        │   │   │   ├─ Service.initInternal()
                        │   │   │   │   ├─ Engine.initInternal()
                        │   │   │   │   ├─ Executor.init()（如果有）
                        │   │   │   │   └─ 遍历 Connector → connector.init()
                        │   │   │   │       └─ ProtocolHandler.init()
                        │   │   │   │           └─ NioEndpoint.init()
                        │   │   │   │               └─ 创建 ServerSocket（但 bindOnInit=false → 不绑定）
                        │   │   │   └─ namingResources.init()
                        │   │   │
                        │   │   ├─ Server.startInternal()
                        │   │   │   ├─ Service.startInternal()
                        │   │   │   │   ├─ Executor.start()（如果有）
                        │   │   │   │   ├─ Engine.startInternal()
                        │   │   │   │   │   ├─ 遍历 Host → host.start()
                        │   │   │   │   │   │   └─ 遍历 Context → context.start()
                        │   │   │   │   │   │       ├─ Context 启动 → 触发 START_EVENT
                        │   │   │   │   │   │       │   └─ removeServiceConnectors() 被回调！
                        │   │   │   │   │   │       │
                        │   │   │   │   │   │       ├─ ServletContainerInitializer.onStartup()
                        │   │   │   │   │   │       │   └─ DeferredServletContainerInitializers.onStartup()
                        │   │   │   │   │   │       │       └─ 遍历 ServletContextInitializer → onStartup(servletContext)
                        │   │   │   │   │   │       │           └─ WebApplicationContextInitializer.initialize(servletContext)
                        │   │   │   │   │   │       │               │  // WebApplicationContextInitializer.java L48
                        │   │   │   │   │   │       │               ├─ prepareWebApplicationContext(servletContext)
                        │   │   │   │   │   │       │               │   ├─ servletContext.setAttribute(ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, context)
                          │   │   │   │   │   │       │               │   └─ context.setServletContext(servletContext)
                        │   │   │   │   │   │       │               ├─ registerApplicationScope(servletContext, beanFactory)
                        │   │   │   │   │   │       │               │   └─ new ServletContextScope(servletContext) → registerScope("application", scope)
                        │   │   │   │   │   │       │               ├─ WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, servletContext)
                        │   │   │   │   │   │       │               └─ 遍历 ServletContextInitializerBeans → initializer.onStartup(servletContext)
                        │   │   │   │   │   │       │                   ├─ DispatcherServletRegistrationBean → 注册 DispatcherServlet 映射 "/"
                        │   │   │   │   │   │       │                   ├─ FilterRegistrationBean → 注册 Filter 映射 "/*"
                        │   │   │   │   │   │       │                   └─ 其他 ServletContextInitializer
                        │   │   │   │   │   │       │
                        │   │   │   │   │   │       ├─ 加载并初始化 Servlet（loadOnStartup >= 0 的）
                        │   │   │   │   │   │       │   ├─ DefaultServlet（loadOnStartup=1）
                        │   │   │   │   │   │       │   ├─ DispatcherServlet（loadOnStartup=1）
                        │   │   │   │   │   │       │   └─ JspServlet（loadOnStartup=3，如果注册了）
                        │   │   │   │   │   │       └─ Context 状态 → STARTED
                        │   │   │   │   │   └─ Engine 状态 → STARTED
                        │   │   │   │   └─ 遍历 Connector → connector.start()（但 Connector 已被移除！）
                        │   │   │   └─ Server 状态 → STARTED
                        │   │   └─ Server.await()（被阻塞在非守护线程中，见下方）
                        │   │
                        │   ├─ rethrowDeferredStartupExceptions()  // L134
                        │   │   └─ 检查所有 Context 是否真正启动成功
                        │   │
                        │   ├─ ContextBindings.bindClassLoader(context, namingToken, classLoader)  // L137
                        │   │   └─ 绑定 JNDI ClassLoader（如果启用了 Naming）
                        │   │
                        │   └─ startNonDaemonAwaitThread()  // L145 ★ 关键！
                        │       └─ new Thread("container-N") { tomcat.getServer().await() }
                        │           ├─ setDaemon(false)  // 非守护线程！
                        │           └─ start()
                        │           // Tomcat 的所有线程都是守护线程，
                        │           // 如果没有非守护线程，JVM 会直接退出
                        │           // 这个线程阻塞在 Server.await()，保持 JVM 存活
                        │
                        └─ 异常处理：
                            ├─ stopSilently() → tomcat.stop()
                            ├─ destroySilently() → tomcat.destroy()
                            └─ throw WebServerException("Unable to start embedded Tomcat")

      // 1.3 注册 Lifecycle Bean
      // L193-195
      getBeanFactory().registerSingleton("webServerGracefulShutdown",
          new WebServerGracefulShutdownLifecycle(webServer))
      getBeanFactory().registerSingleton("webServerStartStop",
          new WebServerStartStopLifecycle(this, webServer))
          // ★ WebServerStartStopLifecycle 实现 SmartLifecycle
          //   在 finishRefresh() 的 LifecycleProcessor.onRefresh() 中被 start()

// ===== 阶段 2：finishRefresh() — 真正启动 Tomcat（绑定端口） =====

// AbstractApplicationContext.finishRefresh()
  └─ LifecycleProcessor.onRefresh()
      └─ 遍历所有 SmartLifecycle Bean → 按 phase 排序 → lifecycle.start()
          └─ WebServerStartStopLifecycle.start()  // L43
              │
              ├─ this.webServer.start()  // L44 ★ 真正启动！
              │   │  // TomcatWebServer.java L226
              │   │
              │   ├─ if started → return  // L228: 防止重复启动
              │   │
              │   ├─ addPreviouslyRemovedConnectors()  // L233 ★ 关键！
              │   │   │  // 把之前移除的 Connector 重新加回 Service
              │   │   └─ 遍历 serviceConnectors Map
              │   │       └─ service.addConnector(connector)  // L307
              │   │           └─ Connector 重新加入 → 触发 init + start
              │   │               ├─ ProtocolHandler.start()
              │   │               │   └─ NioEndpoint.start()
              │   │               │       ├─ 创建 NioSelectorRunnable
              │   │               │       ├─ ServerSocketChannel.open() → bind(port)  ★ 绑定端口！
              │   │               │       ├─ 创建 Worker 线程池（默认 maxThreads=200）
              │   │               │       ├─ 创建 Poller 线程（默认 2 个）
              │   │               │       └─ Poller 线程开始轮询 Selector
              │   │               └─ Connector 状态 → STARTED
              │   │
              │   ├─ Connector connector = this.tomcat.getConnector()  // L234
              │   │
              │   ├─ if connector != null && autoStart:
              │   │   └─ performDeferredLoadOnStartup()  // L236
              │   │       └─ 遍历 Host 子容器
              │   │           └─ TomcatEmbeddedContext.deferredLoadOnStartup()
              │   │               └─ 加载并初始化所有 loadOnStartup >= 0 的 Servlet
              │   │                   // （之前 initialize 阶段可能跳过了，这里补上）
              │   │
              │   ├─ checkThatConnectorsHaveStarted()  // L238
              │   │   └─ 遍历所有 Connector → 检查状态 == STARTED
              │   │       └─ FAILED → throw ConnectorStartFailedException(port)
              │   │
              │   ├─ this.started = true  // L239
              │   │
              │   ├─ logger.info("Tomcat started on port 8080 (http) with context path '/'")  // L240
              │   │
              │   └─ finally:
              │       └─ ContextBindings.unbindClassLoader(context, namingToken, classLoader)  // L252
              │
              └─ this.running = true  // L45
              └─ applicationContext.publishEvent(new ServletWebServerInitializedEvent(webServer, applicationContext))  // L46
                  ⚡ 发布事件：ServletWebServerInitializedEvent

// ===== 阶段 3：请求处理流程（运行时） =====

// 当有 HTTP 请求到达时：
//
// NioEndpoint 的 Poller 线程检测到 Selector 事件
//   └─ SocketProcessor.run()  // 处理连接
//       └─ CoyoteAdapter.service(req, res)  // Coyote → Servlet 转换
//           └─ Mapper.internalMapWrapper()  // 根据 URL 匹配到 Wrapper（Servlet）
//               └─ Engine.getPipeline().getFirst().invoke(req, res)
//                   └─ Host.getPipeline().getFirst().invoke(req, res)
//                       └─ Context.getPipeline().getFirst().invoke(req, res)
//                           └─ Wrapper.getPipeline().getFirst().invoke(req, res)
//                               └─ ApplicationFilterChain.doFilter(req, res)
//                                   ├─ 遍历 Filter → filter.doFilter()
//                                   └─ Servlet.service(req, res)
//                                       └─ DispatcherServlet.doDispatch()
//                                           ├─ HandlerMapping.getHandler()  // 找到 @RequestMapping
//                                           ├─ HandlerAdapter.handle()  // 执行 Controller 方法
//                                           └─ ViewResolver/ResponseBody  // 渲染响应
```

### 两阶段启动总结

| 阶段 | 触发时机 | 方法 | 做了什么 | 端口状态 |
|---|---|---|---|---|
| 初始化 | onRefresh() | TomcatWebServer.initialize() | 创建 Tomcat 实例、配置 Connector/Engine/Host/Context、tomcat.start()（移除 Connector）、注册 Servlet/Filter、启动非守护线程 | 未绑定 |
| 启动 | finishRefresh() → SmartLifecycle.start() | WebServerStartStopLifecycle.start() → TomcatWebServer.start() | 加回 Connector、绑定端口、初始化 Servlet（loadOnStartup）、检查启动状态 | 已绑定，可接收请求 |

**为什么要两阶段？**

因为 onRefresh() 在 finishBeanFactoryInstantiation() 之前执行，此时 Bean 还没全部创建完。如果在 onRefresh 就绑定端口，外部请求可能进来但 Controller 还没初始化。所以先 tomcat.start() 初始化内部组件（Context/Servlet/Filter），移除 Connector 不绑定端口；等所有 Bean 都创建完后，在 finishRefresh() 阶段通过 SmartLifecycle 回调加回 Connector 绑定端口，此时所有 Bean 都就绪了。

### Tomcat 内部组件层级

```
Server（StandardServer）                    // 顶层
  └─ Service（StandardService）              // 服务
      ├─ Executor（ThreadPoolExecutor）       // 线程池（maxThreads=200）
      ├─ Connector（Connector）                // 连接器
      │   ├─ ProtocolHandler（Http11NioProtocol）  // 协议处理器
      │   │   ├─ NioEndpoint                   // I/O 端点
      │   │   │   ├─ ServerSocketChannel        // 监听端口
      │   │   │   ├─ Poller（2个线程）           // 轮询 I/O 事件
      │   │   │   └─ Worker（线程池）            // 处理请求
      │   │   └─ Processor（Http11Processor）    // HTTP 协议解析
      │   └─ CoyoteAdapter                      // Coyote→Servlet 适配
      └─ Engine（StandardEngine）               // 引擎
          └─ Host（StandardHost）               // 虚拟主机
              └─ Context（TomcatEmbeddedContext） // 应用上下文
                  ├─ Wrapper（DefaultServlet）    // 静态资源 "/"
                  ├─ Wrapper（DispatcherServlet） // Spring MVC "/"
                  └─ Wrapper（JspServlet）        // JSP "*.jsp"
                      └─ Pipeline（Filter Chain）  // 过滤器链
```

## 专题四：MySQL 事务控制 + Spring 事务控制 + 二者关联

> **【阶段】**

MySQL 部分基于 InnoDB 存储引擎原理 + 官方文档；Spring 部分源码位置：Transactional.java + TransactionInterceptor.java + TransactionAspectSupport.java + AbstractPlatformTransactionManager.java + DataSourceTransactionManager.java + TransactionSynchronizationManager.java（Spring Framework 6.2.18）

### Part A：MySQL InnoDB 事务控制

#### A1. ACID 特性

```
A（Atomicity 原子性）  — undo log 实现，回滚时根据 undo log 反向操作
I（Isolation 隔离性）   — 锁机制 + MVCC 实现
D（Durability 持久性） — redo log（binlog 也有作用）实现，先写日志后刷盘
C（Consistency 一致性） — 由 AID 共同保证，最终数据状态一致

举例：
  事务 A 执行：UPDATE account SET balance = balance - 100 WHERE id = 1
  事务 A 执行：UPDATE account SET balance = balance + 100 WHERE id = 2
  → 如果中间崩溃 → undo log 回滚两条 → 原子性
  → 如果提交后崩溃 → redo log 恢复 → 持久性
  → 事务 B 并发读取 → MVCC + 锁 → 隔离性
  → 最终 id=1 减 100、id=2 加 100 → 一致性
```

#### A2. 事务隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现 |
|---|---|---|---|---|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 不加锁，读最新版本 |
| READ COMMITTED（RC） | 不可能 | 可能 | 可能 | MVCC：每次 SELECT 生成新 read view |
| REPEATABLE READ（RR）默认 | 不可能 | 不可能 | 不可能* | MVCC：事务开始时生成 read view，后续复用 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | 所有 SELECT 隐式加 LOCK IN SHARE MODE |

* InnoDB 的 RR 通过 next-key lock（行锁+间隙锁）解决了幻读，但 SQL 标准定义 RR 仍可能幻读

#### A2.1 隔离级别示例

```
-- 脏读示例（READ UNCOMMITTED）
-- 事务 A                          -- 事务 B
BEGIN;                             BEGIN;
UPDATE account SET balance=0 WHERE id=1;
                                   SELECT * FROM account WHERE id=1;  -- 读到 balance=0（脏读！）
ROLLBACK;                          -- 事务 A 回滚了，balance 恢复原值
                                   -- 但事务 B 已经读到了错误的 0
COMMIT;

-- 不可重复读示例（READ COMMITTED）
-- 事务 A                          -- 事务 B
BEGIN;                             BEGIN;
SELECT * FROM account WHERE id=1;  -- 读到 balance=1000
UPDATE account SET balance=2000 WHERE id=1;
COMMIT;
                                   SELECT * FROM account WHERE id=1;  -- 读到 balance=2000
                                   -- 同一事务内两次读结果不同！
                                   COMMIT;

-- 幻读示例（READ COMMITTED）
-- 事务 A                          -- 事务 B
BEGIN;                             BEGIN;
SELECT * FROM account WHERE age>20; -- 读到 3 条
INSERT INTO account VALUES(...);     -- 插入 1 条 age=25
COMMIT;
                                   SELECT * FROM account WHERE age>20; -- 读到 4 条！幻读
                                   COMMIT;

-- RR 级别解决幻读（InnoDB 默认）
-- 事务 A                          -- 事务 B
BEGIN;                             BEGIN;
                                   SELECT * FROM account WHERE age>20; -- 读到 3 条（生成 read view）
UPDATE account SET name='x' WHERE age>20; -- 加 Next-Key Lock
                                   INSERT INTO account VALUES(age=25); -- 阻塞！被间隙锁挡住
COMMIT;                            -- 释放锁
                                   INSERT 成功
                                   SELECT * FROM account WHERE age>20; -- 仍读到 3 条（read view 不变）
                                   COMMIT;
```

#### A3. MVCC 机制（多版本并发控制）

```
每行数据在 InnoDB 中有两个隐藏列：
  ├─ DB_TRX_ID    — 最后修改该行的事务 ID
  └─ DB_ROLL_PTR  — 指向 undo log 的指针（旧版本链表）

Read View（读视图）结构：
  ├─ m_ids        — 生成 read view 时活跃（未提交）事务 ID 列表
  ├─ min_trx_id   — m_ids 中最小值
  ├─ max_trx_id   — 下一个要分配的事务 ID
  └─ creator_trx_id — 当前事务 ID

可见性判断规则（RR 级别）：
  1. DB_TRX_ID == creator_trx_id → 自己修改的，可见
  2. DB_TRX_ID = max_trx_id → 修改事务在 read view 后才开启，不可见
  4. min_trx_id = min_trx_id(200) && 200  undo log 的 DB_TRX_ID 时
  → 该 undo log 不再被任何 read view 需要 → 可清理
  → 清理从链表尾部开始（最老的版本）
```

#### A6. redo log 详解

#### A6.1 redo log 的作用

```
redo log（重做日志）保证事务持久性：
  → 即使 MySQL 崩溃，重启后也能用 redo log 恢复已提交的数据

redo log 特点：
  ├─ 物理日志 — 记录"哪个数据页的哪个偏移量改了什么"（不是 SQL 级别）
  ├─ 顺序写入 — 追加写，不需要随机 IO，性能高
  ├─ 循环使用 — 固定大小（如 2 个文件，每个 48MB），写满后从头覆盖
  └─ WAL（Write-Ahead Logging）— 先写日志再改数据页

redo log 文件结构：
  ├─ ib_logfile0  ─┐
  ├─ ib_logfile1  ─┤  循环写入
  └─ ...           │
  参数：innodb_log_files_in_group（文件数，默认2）
        innodb_log_file_size（每个文件大小）
        innodb_log_buffer_size（redo log buffer 大小，默认 16MB）
```

#### A6.2 redo log 写入流程

```
写入流程（3 层）：

  1. 内存层：redo log buffer（InnoDB 内存中的缓冲区）
     → 事务执行时，修改数据页的同时写 redo log 到 buffer
     → 事务提交时触发刷盘

  2. OS 层：文件系统缓存（Page Cache）
     → os cache 是操作系统的文件缓存，断电会丢

  3. 磁盘层：redo log file（物理文件）
     → fsync 系统调用强制刷到磁盘

刷盘策略 innodb_flush_log_at_trx_commit：
  ├─ 0：每秒刷盘（后台线程定时 fsync）
  │   → 提交时不刷盘，性能最高
  │   → 崩溃可能丢失 1 秒数据
  │
  ├─ 1：每次提交都 fsync 刷盘（默认）
  │   → 最安全，不丢数据
  │   → 性能最低（每次提交都有 IO）
  │
  └─ 2：每次提交写 os cache，每秒 fsync 刷盘
      → 折中方案
      → MySQL 崩溃不丢数据，OS 崩溃可能丢 1 秒

示例：
  事务 A 执行：UPDATE account SET balance=2000 WHERE id=1

  1. InnoDB 修改 Buffer Pool 中的数据页（balance=2000）
  2. 同时写 redo log buffer：
     "space_id=10, page_no=5, offset=100, 旧值=1000, 新值=2000"
  3. 事务提交 → redo log buffer 刷到磁盘（根据刷盘策略）
  4. 数据页（脏页）延迟刷盘（由后台线程 checkpoint 处理）
  5. 如果在步骤 4 之前崩溃 → 重启后读 redo log → 重做 → 恢复 balance=2000
```

#### A6.3 redo log 与 binlog 的两阶段提交

```
两阶段提交（2PC）保证 redo log 和 binlog 的一致性：

  事务 A 执行 COMMIT 时：

  阶段 1：Prepare
  ├─ InnoDB 写 redo log，标记为 PREPARE 状态
  │   → redo log 内容：修改记录 + XID（事务 ID）+ PREPARE 标记
  └─ redo log 刷盘（根据 innodb_flush_log_at_trx_commit）

  阶段 2：Commit
  ├─ MySQL Server 写 binlog
  │   → binlog 内容：SQL 语句或行变更（逻辑日志）
  │   → binlog 刷盘（根据 sync_binlog）
  └─ InnoDB 写 redo log，标记为 COMMIT 状态
      → 只改标记，内容不变

  崩溃恢复规则：
  ├─ 如果 redo log 有 PREPARE 但没有 COMMIT：
  │   ├─ 检查 binlog 是否有对应的 XID
  │   │   ├─ 有 → 说明 binlog 已写 → 提交事务（重做 redo log）
  │   │   └─ 没有 → 说明 binlog 没写 → 回滚事务（用 undo log）
  │   └─ 保证 redo log 和 binlog 一致
  │
  └─ 如果 redo log 有 COMMIT：
      → 直接重做（事务已提交）

  为什么要两阶段提交？
  ├─ 如果先写 redo log 后写 binlog：
  │   → redo log 写完崩溃 → binlog 没写
  │   → 恢复后主库有数据，从库没有 → 主从不一致
  │
  ├─ 如果先写 binlog 后写 redo log：
  │   → binlog 写完崩溃 → redo log 没写
  │   → 恢复后主库没数据，从库有 → 主从不一致
  │
  └─ 两阶段提交：先 redo log(PREPARE) → binlog → redo log(COMMIT)
      → 任何阶段崩溃都能通过 redo log 状态 + binlog 是否存在来判断
```

#### A6.4 redo log vs binlog 对比

| 特性 | redo log（InnoDB） | binlog（MySQL Server） |
|---|---|---|
| 日志类型 | 物理日志（页级别） | 逻辑日志（SQL/行变更） |
| 写入方式 | 循环写入（固定大小） | 追加写入（文件切换） |
| 用途 | 崩溃恢复（crash safe） | 主从复制 + 数据恢复 |
| 写入层级 | InnoDB 引擎层 | MySQL Server 层 |
| 刷盘参数 | innodb_flush_log_at_trx_commit | sync_binlog |
| 内容 | "page 5 offset 100: 1000→2000" | "UPDATE account SET balance=2000 WHERE id=1" |

#### A7. MySQL 事务控制语句

```
BEGIN / START TRANSACTION  — 开启事务（自动提交设为 false）
COMMIT                     — 提交事务
ROLLBACK                   — 回滚事务
SAVEPOINT name             — 创建保存点
ROLLBACK TO name           — 回滚到保存点
RELEASE SAVEPOINT name     — 删除保存点
SET TRANSACTION ISOLATION LEVEL READ COMMITTED — 设置隔离级别
SET autocommit = 0/1       — 关闭/开启自动提交

InnoDB 默认：autocommit=1（每条 SQL 自动提交）
  → 执行 BEGIN 后 → autocommit 隐式设为 0
  → 执行 COMMIT/ROLLBACK 后 → autocommit 恢复为 1

SAVEPOINT 示例：
  BEGIN;
  INSERT INTO account VALUES(1, '张三', 1000);
  SAVEPOINT sp1;
  INSERT INTO account VALUES(2, '李四', 2000);
  ROLLBACK TO sp1;          -- 回滚到 sp1，id=2 被删除，id=1 保留
  COMMIT;                   -- 只提交 id=1
```

### Part B：Spring 事务控制

#### B1. @Transactional 注解属性

```
// Transactional.java L127-285
@Transactional(
    propagation   = Propagation.REQUIRED,     // 传播行为（默认 REQUIRED）
    isolation     = Isolation.DEFAULT,         // 隔离级别（默认用数据库的）
    timeout       = TransactionDefinition.TIMEOUT_DEFAULT, // 超时（秒）
    readOnly      = false,                     // 是否只读
    rollbackFor   = {},                         // 指定回滚异常类型
    noRollbackFor = {},                         // 指定不回滚异常类型
    transactionManager = "",                   // 指定事务管理器
    label         = {}                          // 事务标签
)

// 默认回滚规则：
// - RuntimeException 和 Error → 回滚
// - Checked Exception（受检异常）→ 不回滚
// 可通过 rollbackFor/noRollbackFor 覆盖
```

#### B2. 七种传播行为

| 传播行为 | 有事务时 | 无事务时 | 源码处理 |
|---|---|---|---|
| REQUIRED（默认） | 加入当前事务 | 新建事务 | handleExistingTransaction → 参与现有事务 |
| REQUIRES_NEW | 挂起当前事务，新建事务 | 新建事务 | suspend → startTransaction → doBegin |
| NESTED | 创建保存点（嵌套事务） | 新建事务 | createAndHoldSavepoint → JDBC Savepoint |
| SUPPORTS | 加入当前事务 | 非事务执行 | 参与现有事务 / 空 TransactionStatus |
| NOT_SUPPORTED | 挂起当前事务，非事务执行 | 非事务执行 | suspend → 空 TransactionStatus |
| MANDATORY | 加入当前事务 | 抛异常 | 无事务 → IllegalTransactionStateException |
| NEVER | 抛异常 | 非事务执行 | 有事务 → IllegalTransactionStateException |

#### B3. Spring 事务完整调用链（源码级）

```
// ===== 1. AOP 拦截入口 =====

// TransactionInterceptor.java L112
TransactionInterceptor.invoke(MethodInvocation invocation)
  ├─ targetClass = AopUtils.getTargetClass(invocation.getThis())
  └─ invokeWithinTransaction(method, targetClass, invocation::proceed)  // L119
      │  // TransactionAspectSupport.java L342
      │
      ├─ ① 获取事务属性
      │  // L346-347
      │  TransactionAttributeSource tas = getTransactionAttributeSource()
      │  TransactionAttribute txAttr = tas.getTransactionAttribute(method, targetClass)
      │  // → 解析 @Transactional 注解 → RuleBasedTransactionAttribute
      │  //   包含 propagation、isolation、timeout、readOnly、rollbackRules
      │
      ├─ ② 确定事务管理器
      │  // L348
      │  TransactionManager tm = determineTransactionManager(txAttr, targetClass)
      │  // → 根据 @Transactional.transactionManager() 或 qualifier 找 Bean
      │  // → 默认 DataSourceTransactionManager（JDBC）/ JtaTransactionManager（JTA）
      │
      ├─ ③ 创建事务（如果需要）
      │  // L374 → createTransactionIfNecessary L623
      │  TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification)
      │  │
      │  │  // L639: tm.getTransaction(txAttr)
      │  │  // AbstractPlatformTransactionManager.java L373
      │  │  TransactionStatus status = tm.getTransaction(txAttr)
      │  │  │
      │  │  ├─ doGetTransaction()  // L379 → DataSourceTransactionManager L248
      │  │  │   └─ 从 TransactionSynchronizationManager 获取线程绑定的 ConnectionHolder
      │  │  │       // TransactionSynchronizationManager.getResource(dataSource)
      │  │  │       // → ThreadLocal> → key=DataSource, value=ConnectionHolder
      │  │  │
      │  │  ├─ isExistingTransaction(transaction)  // L382 → DataSourceTransactionManager L258
      │  │  │   └─ 检查 ConnectionHolder 是否存在且 isTransactionActive()
      │  │  │
      │  │  ├─ if 有现有事务 → handleExistingTransaction(def, transaction, debug)  // L426
      │  │  │   ├─ PROPAGATION_NEVER → throw IllegalTransactionStateException  // L430
      │  │  │   ├─ PROPAGATION_NOT_SUPPORTED → suspend → 空 TransactionStatus  // L435
      │  │  │   ├─ PROPAGATION_REQUIRES_NEW → suspend → startTransaction → doBegin  // L445
      │  │  │   │   └─ suspend(transaction)  // L450
      │  │  │   │       └─ doSuspend → unbindResource(dataSource) → 解绑 ConnectionHolder
      │  │  │   │       └─ 保存到 SuspendedResourcesHolder（后续 resume 用）
      │  │  │   ├─ PROPAGATION_NESTED → createAndHoldSavepoint  // L460
      │  │  │   │   └─ useSavepointForNestedTransaction() → true
      │  │  │   │       └─ status.createAndHoldSavepoint() → Connection.setSavepoint()
      │  │  │   └─ PROPAGATION_REQUIRED/SUPPORTS/MANDATORY → 参与现有事务  // L494
      │  │  │       └─ prepareTransactionStatus → 不新建事务，标记 newTransaction=false
      │  │  │
      │  │  └─ if 无现有事务：
      │  │      ├─ PROPAGATION_MANDATORY → throw IllegalTransactionStateException  // L393
      │  │      ├─ PROPAGATION_REQUIRED/REQUIRES_NEW/NESTED → startTransaction  // L397
      │  │      │   └─ startTransaction(def, transaction, false, debug, suspendedResources)  // L524
      │  │      │       ├─ newTransactionStatus → new DefaultTransactionStatus  // L528
      │  │      │       ├─ doBegin(transaction, definition)  // L532 ★ 开启事务
      │  │      │       │   // DataSourceTransactionManager.java L264
      │  │      │       │   ├─ 从连接池获取 Connection  // L271
      │  │      │       │   │   └─ obtainDataSource().getConnection()
      │  │      │       │   │       → HikariPool.getConnection() → 物理连接
      │  │      │       │   ├─ DataSourceUtils.prepareConnectionForTransaction(con, isolationLevel, readOnly)
      │  │      │       │   │   ├─ con.setTransactionIsolation(isolationLevel)  // 设置隔离级别
      │  │      │       │   │   └─ con.setReadOnly(readOnly)
      │  │      │       │   ├─ if con.getAutoCommit() → con.setAutoCommit(false)  // L293-298 ★ 关键！
      │  │      │       │   │   // ★ 等价于执行 BEGIN
      │  │      │       │   │   // 保存原 autoCommit 状态（mustRestoreAutoCommit=true）
      │  │      │       │   ├─ prepareTransactionalConnection(con, definition)  // L301
      │  │      │       │   │   └─ if enforceReadOnly && readOnly → "SET TRANSACTION READ ONLY"
      │  │      │       │   ├─ connectionHolder.setTransactionActive(true)  // L302
      │  │      │       │   └─ TransactionSynchronizationManager.bindResource(dataSource, connectionHolder)
      │  │      │       │       // ★ 把 Connection 绑定到当前线程（ThreadLocal）
      │  │      │       │       // 后续同线程的 JdbcTemplate 通过 DataSourceUtils.getConnection()
      │  │      │       │       //   → 从 ThreadLocal 取同一个 Connection → 保证同一事务
      │  │      │       └─ prepareSynchronization(status, definition)  // L538
      │  │      │           └─ 初始化 ThreadLocal 同步变量
      │  │      │
      │  │      └─ PROPAGATION_SUPPORTS/NOT_SUPPORTED/NEVER → 空 TransactionStatus  // L412
      │  │
      │  └─ prepareTransactionInfo → txInfo.bindToThread()  // L659-684
      │      // 把 TransactionInfo 绑定到 ThreadLocal 栈（支持嵌套）
      │
      ├─ ④ 执行业务方法
      │  // L380
      │  retVal = invocation.proceedWithInvocation()
      │  // → 调用被 @Transactional 标注的业务方法
      │  // → 如果方法内调用其他 @Transactional 方法 → 递归进入此流程
      │  //   → 传播行为决定是复用还是新建事务
      │
      ├─ ⑤ 异常处理
      │  // L382-386
      │  catch (Throwable ex):
      │  │  completeTransactionAfterThrowing(txInfo, ex)  // L708
      │  │  │
      │  │  │  // L714: 检查是否应该回滚
      │  │  │  if txInfo.transactionAttribute.rollbackOn(ex):
      │  │  │  │  // RuleBasedTransactionAttribute.rollbackOn()
      │  │  │  │  // → 遍历 rollbackFor/noRollbackFor 规则匹配
      │  │  │  │  // → 默认：RuntimeException/Error → true，Checked Exception → false
      │  │  │  │
      │  │  │  │  tm.rollback(txInfo.getTransactionStatus())  // L716
      │  │  │  │  // AbstractPlatformTransactionManager.java L859
      │  │  │  │  └─ processRollback(defStatus, false)  // L875
      │  │  │  │      ├─ triggerBeforeCompletion(status)  // 回调 TransactionSynchronization
      │  │  │  │      ├─ if hasSavepoint → rollbackToHeldSavepoint()  // 嵌套事务回滚到保存点
      │  │  │  │      │   └─ Connection.rollback(savepoint)
      │  │  │  │      ├─ else if isNewTransaction → doRollback(status)  // 新事务 → 回滚
      │  │  │  │      │   └─ DataSourceTransactionManager.doRollback()  // L352
      │  │  │  │      │       └─ con.rollback()  // ★ 调用 JDBC Connection.rollback()
      │  │  │  │      │           → MySQL 执行 ROLLBACK → 根据 undo log 回滚
      │  │  │  │      └─ else if hasTransaction → doSetRollbackOnly(status)  // 参与事务 → 标记回滚
      │  │  │  │          └─ connectionHolder.setRollbackOnly(true)
      │  │  │  │              // ★ 不立即回滚，等外层事务提交时发现 rollbackOnly → 回滚
      │  │  │  │      └─ triggerAfterCompletion(status, STATUS_ROLLED_BACK)
      │  │  │  │      └─ cleanupAfterCompletion(status)  // L1056
      │  │  │  │          ├─ TransactionSynchronizationManager.clear()  // 清理 ThreadLocal
      │  │  │  │          ├─ doCleanupAfterCompletion  // DataSourceTransactionManager L377
      │  │  │  │          │   ├─ unbindResource(dataSource)  // 解绑 Connection
      │  │  │  │          │   ├─ con.setAutoCommit(true)  // 恢复自动提交
      │  │  │  │          │   ├─ resetConnectionAfterTransaction  // 恢复隔离级别
      │  │  │  │          │   └─ DataSourceUtils.releaseConnection(con)  // 归还连接池
      │  │  │  │          └─ resume(suspendedResources)  // 恢复挂起的事务
      │  │  │  │
      │  │  │  else:  // 不回滚 → 提交
      │  │  │    tm.commit(txInfo.getTransactionStatus())  // L732
      │  │  │    // → 同正常流程的 commit
      │  │  │
      │  └─ throw ex  // 重新抛出异常
      │
      └─ ⑥ 正常返回处理
         // L416
         commitTransactionAfterReturning(txInfo)  // L693
         │
         │  // L698: tm.commit(txInfo.getTransactionStatus())
         │  // AbstractPlatformTransactionManager.java L735
         │  tm.commit(status)
         │  │
         │  ├─ if isLocalRollbackOnly → processRollback  // L742: 代码中标记了回滚
         │  ├─ if isGlobalRollbackOnly && !shouldCommitOnGlobalRollbackOnly → processRollback  // L750
         │  └─ processCommit(defStatus)  // L758 → L767
         │      │
         │      ├─ prepareForCommit(status)
         │      ├─ triggerBeforeCommit(status)  // 回调 TransactionSynchronization.beforeCommit
         │      ├─ triggerBeforeCompletion(status)  // 回调 TransactionSynchronization.beforeCompletion
         │      │
         │      ├─ if hasSavepoint → releaseHeldSavepoint()  // 嵌套事务 → 释放保存点
         │      │   └─ Connection.releaseSavepoint(savepoint)
         │      │
         │      ├─ else if isNewTransaction → doCommit(status)  // 新事务 → 提交  // L795
         │      │   └─ DataSourceTransactionManager.doCommit()  // L337
         │      │       └─ con.commit()  // ★ 调用 JDBC Connection.commit()
         │      │           → MySQL 执行 COMMIT → 写 redo log + binlog → 释放锁
         │      │
         │      ├─ else → 参与事务，不提交（等外层提交）
         │      │   // 如果 isFailEarlyOnGlobalRollbackOnly → 检查 globalRollbackOnly
         │      │
         │      ├─ triggerAfterCommit(status)  // 回调 TransactionSynchronization.afterCommit
         │      │   // ★ 此时事务已提交，但 Connection 还未释放
         │      │   // 可以在此执行一些后置操作（如发消息）
         │      │
         │      ├─ triggerAfterCompletion(status, STATUS_COMMITTED)
         │      │   // 回调 TransactionSynchronization.afterCompletion
         │      │
         │      └─ cleanupAfterCompletion(status)  // L1056
         │          ├─ status.setCompleted()
         │          ├─ TransactionSynchronizationManager.clear()  // 清理 ThreadLocal
         │          ├─ if isNewTransaction → doCleanupAfterCompletion  // L1062
         │          │   └─ DataSourceTransactionManager.doCleanupAfterCompletion()  // L377
         │          │       ├─ unbindResource(dataSource)  // 解绑 ConnectionHolder
         │          │       ├─ if mustRestoreAutoCommit → con.setAutoCommit(true)  // 恢复
         │          │       ├─ resetConnectionAfterTransaction  // 恢复隔离级别/readOnly
         │          │       └─ DataSourceUtils.releaseConnection(con)  // 归还连接池
         │          │           → HikariPool.recycle(connection)
         │          └─ if hasSuspendedResources → resume  // 恢复挂起的事务
         │              └─ doResume → bindResource(dataSource, suspendedConnectionHolder)
         │                  // 把挂起的 Connection 重新绑定到当前线程
         │
         └─ cleanupTransactionInfo(txInfo)  // L388
             └─ txInfo.restoreThreadLocalStatus()  // 恢复上层 TransactionInfo
```

#### B4. Spring 事务的核心机制

**1. ThreadLocal 绑定 Connection**

TransactionSynchronizationManager 用 ThreadLocal<Map<Object, Object>> 把 DataSource → ConnectionHolder 绑定到线程。同一线程内，JdbcTemplate/MyBatis 通过 DataSourceUtils.getConnection(dataSource) 从 ThreadLocal 取同一个 Connection，保证同一事务。

**2. autoCommit 控制**

doBegin 时 con.setAutoCommit(false) → 等价于 BEGIN

doCleanupAfterCompletion 时 con.setAutoCommit(true) → 恢复

所以 Spring 事务的本质就是控制 JDBC Connection 的 autoCommit

**3. AOP 代理拦截**

@Transactional 生效的前提是 Bean 被 AOP 代理（见专题二）。TransactionInterceptor 作为 MethodInterceptor，在方法调用前后执行事务管理逻辑。类内部方法直接调用（this.xxx()）不经过代理，事务不生效。

**4. 传播行为 = suspend + resume**

REQUIRES_NEW：suspend 当前事务（解绑 Connection）→ 新建事务（新 Connection）→ 完成后 resume（重新绑定旧 Connection）

NESTED：不 suspend，在当前 Connection 上创建 Savepoint

**5. rollbackOnly 标记**

参与事务的子方法抛异常时，不立即回滚，而是标记 connectionHolder.setRollbackOnly(true)。外层事务 commit 时检查此标记 → 发现 true → 改为 rollback → 抛 UnexpectedRollbackException

### Part C：Spring 事务与 MySQL 事务的关联

#### C1. Spring 如何映射到 MySQL 事务

| Spring @Transactional | Spring 内部操作 | MySQL InnoDB 对应 |
|---|---|---|
| propagation=REQUIRED | doBegin → con.setAutoCommit(false) | BEGIN（开启事务，关闭自动提交） |
| isolation=READ_COMMITTED | con.setTransactionIsolation(READ_COMMITTED) | SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED |
| readOnly=true | con.setReadOnly(true) / SET TRANSACTION READ ONLY | InnoDB 标记只读，优化器可优化（不加锁读） |
| timeout=30 | connectionHolder.setTimeoutInSeconds(30) | Statement.setQueryTimeout(30)（执行 SQL 时生效） |
| 正常返回 → commit | doCommit → con.commit() | COMMIT → 写 redo log(commit) + binlog → 释放锁 |
| 异常 → rollback | doRollback → con.rollback() | ROLLBACK → 根据 undo log 回滚 → 释放锁 |
| propagation=NESTED | con.setSavepoint("SAVEPOINT_1") | SAVEPOINT → InnoDB 创建 undo log 保存点 |
| propagation=REQUIRES_NEW | suspend → 新 Connection → doBegin → commit/rollback → resume | 两个独立的 MySQL 事务（两个 Connection） |

#### C2. 完整时序图：Spring 调用 MySQL 事务

```
用户调用 service.updateOrder()
  │
  ├─ AOP 代理拦截 → TransactionInterceptor.invoke()
  │   ├─ 解析 @Transactional(propagation=REQUIRED, isolation=READ_COMMITTED)
  │   ├─ tm.getTransaction(txAttr)
  │   │   ├─ doGetTransaction → 从 ThreadLocal 取 ConnectionHolder（无）
  │   │   ├─ isExistingTransaction → false（无现有事务）
  │   │   ├─ startTransaction → doBegin
  │   │   │   ├─ HikariPool.getConnection() → 获取物理连接
  │   │   │   ├─ con.setTransactionIsolation(READ_COMMITTED)  → MySQL 设置隔离级别
  │   │   │   ├─ con.setAutoCommit(false)                     → MySQL BEGIN
  │   │   │   ├─ connectionHolder.setTransactionActive(true)
  │   │   │   └─ TransactionSynchronizationManager.bindResource(dataSource, connHolder)
  │   │   │       → 绑定到 ThreadLocal
  │   │   └─ prepareSynchronization → 初始化 ThreadLocal 同步变量
  │   │
  │   ├─ invocation.proceedWithInvocation() → 执行业务方法
  │   │   │
  │   │   ├─ JdbcTemplate.update("UPDATE order_info SET status=? WHERE id=?", ...)
  │   │   │   ├─ DataSourceUtils.getConnection(dataSource)
  │   │   │   │   └─ 从 ThreadLocal 取绑定的 Connection（同一个！）
  │   │   │   ├─ PreparedStatement.execute()
  │   │   │   │   → MySQL 执行 UPDATE
  │   │   │   │   ├─ 加 X 锁（行锁/临键锁）
  │   │   │   │   ├─ 写 undo log（旧值）
  │   │   │   │   └─ 写 redo log（新值）
  │   │   │   └─ 不关闭 Connection（事务中，归还连接池会延迟到事务结束）
  │   │   │
  │   │   └─ JdbcTemplate.update("UPDATE order_item SET qty=? WHERE order_id=?", ...)
  │   │       └─ 同上，用同一个 Connection → 同一个事务
  │   │
  │   ├─ 正常返回 → commitTransactionAfterReturning
  │   │   └─ tm.commit(status)
  │   │       ├─ triggerBeforeCommit → 回调 Synchronization.beforeCommit
  │   │       ├─ doCommit → con.commit()
  │   │       │   → MySQL COMMIT
  │   │       │   ├─ 写 redo log（commit 状态）
  │   │       │   ├─ 写 binlog
  │   │       │   ├─ 释放行锁
  │   │       │   └─ 清理 undo log（标记可清理）
  │   │       ├─ triggerAfterCommit → 回调 Synchronization.afterCommit
  │   │       └─ cleanupAfterCompletion
  │   │           ├─ TransactionSynchronizationManager.clear() → 清理 ThreadLocal
  │   │           ├─ doCleanupAfterCompletion
  │   │           │   ├─ unbindResource(dataSource) → 解绑 Connection
  │   │           │   ├─ con.setAutoCommit(true) → 恢复自动提交
  │   │           │   └─ DataSourceUtils.releaseConnection → HikariPool.recycle()
  │   │           └─ resume（如果有挂起的事务）
  │   │
  │   └─ 返回业务结果
  │
  └─ 如果异常 → completeTransactionAfterThrowing
      ├─ rollbackOn(ex) → RuntimeException → true
      └─ tm.rollback(status)
          ├─ triggerBeforeCompletion
          ├─ doRollback → con.rollback()
          │   → MySQL ROLLBACK
          │   ├─ 根据 undo log 回滚数据
          │   ├─ 释放行锁
          │   └─ 清理 undo log
          ├─ triggerAfterCompletion(STATUS_ROLLED_BACK)
          └─ cleanupAfterCompletion → 同上清理
```

#### C3. 关键关联点总结

**1. Spring 事务本质 = JDBC Connection 控制**

Spring 不直接操作 MySQL 事务，而是通过 JDBC API（Connection.setAutoCommit/setTransactionIsolation/commit/rollback）间接控制 MySQL。DataSourceTransactionManager.doBegin/doCommit/doRollback 就是对 Connection 的封装。

**2. 隔离级别传递链**

@Transactional(isolation=READ_COMMITTED) → DataSourceUtils.prepareConnectionForTransaction → con.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED) → MySQL SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED → InnoDB 调整 read view 生成策略

**3. 传播行为 = Connection 生命周期管理**

REQUIRED → 一个 Connection 贯穿整个调用链

REQUIRES_NEW → 两个 Connection（挂起旧的，新建新的）→ 两个独立的 MySQL 事务

NESTED → 一个 Connection + Savepoint → MySQL 内部一个事务的子保存点

**4. 回滚规则差异**

MySQL：任何异常（包括 SQL 语法错误）都会导致事务回滚

Spring：默认只对 RuntimeException/Error 回滚，Checked Exception 不回滚

→ 如果业务抛 Checked Exception 但希望回滚，必须配置 rollbackFor=xxx.class

**5. 连接池与事务**

Spring 事务期间，Connection 从连接池借出并绑定到 ThreadLocal，事务结束后归还。如果事务方法执行很慢，Connection 被占用时间长，可能导致连接池耗尽。timeout 属性可以设置超时自动回滚。

## 完整调用链树形图

> **【阶段】**

SpringApplication.run(MyApplication.class, args) [SpringApplication:1353]
│
├── new SpringApplication(primarySources) [SpringApplication:274] // 构造阶段 L274-284
│   ├── 保存 primarySources → LinkedHashSet [SpringApplication:277] // L277
│   ├── WebApplicationType.deduce() [WebApplicationType:63] // L278
│   │   ├── SpringFactoriesLoader.load(Deducer.class) [WebApplicationType:64] // 查 spring.factories
│   │   │   └── deducer.deduceWebApplicationType() [WebApplicationType:66] // 每个 Deducer 尝试推断
│   │   └── isServletApplication() [WebApplicationType:73] // 回退逻辑
│   │       ├── ClassUtils.isPresent("jakarta.servlet.Servlet") [WebApplicationType:74]
│   │       └── ClassUtils.isPresent("ConfigurableWebApplicationContext") [WebApplicationType:74]
│   │       └── 两者都在 → SERVLET；否则 → NONE
│   ├── getSpringFactoriesInstances(BootstrapRegistryInitializer.class) [SpringApplication:280] // L279
│   │   └── SpringFactoriesLoader.forDefaultResourceLocation().load(type) [SpringFactoriesLoader]
│   │       ├── 读取所有 META-INF/spring.factories 文件 [SpringFactoriesLoader]
│   │       ├── 按 type 过滤实现类列表 [SpringFactoriesLoader]
│   │       └── 反射实例化（有参构造 → ArgumentResolver 注入） [SpringFactoriesLoader]
│   │           └── BeanUtils.instantiateClass(constructor, args) [BeanUtils]
│   ├── getSpringFactoriesInstances(ApplicationContextInitializer.class) [SpringApplication:281] // L281
│   │   └── 同上 → setInitializers(collection)
│   │       └── AnnotationAwareOrderComparator 排序 [SpringApplication:1478]
│   ├── getSpringFactoriesInstances(ApplicationListener.class) [SpringApplication:282] // L282
│   │   └── 同上 → setListeners(collection)
│   │       └── AnnotationAwareOrderComparator 排序 [SpringApplication:1478]
│   └── deduceMainApplicationClass() [SpringApplication:283] // L283
│       └── StackWalker.getInstance(RETAIN_CLASS_REFERENCE) [SpringApplication:287]
│           └── walk(frames → findMainClass(frames)) [SpringApplication:288]
│               └── filter(frame → "main".equals(frame.getMethodName())).findFirst() [SpringApplication:293]
│                   └── map(StackFrame::getDeclaringClass) → MyApplication.class [SpringApplication:295]
│
└── .run(args) [SpringApplication:304] // 启动阶段 L304-342
    │
    ├── Startup.create() [SpringApplication:305] // L305
    │   ├── ClassUtils.isPresent("jdk.crac.management.CRaCMXBean") [SpringApplication:1773]
    │   ├── 有 CRaC → new CoordinatedRestoreAtCheckpointStartup()
    │   │   └── startTime() → CRaCMXBean.getRestoreTime()（≥0 用恢复时间） [SpringApplication:1828]
    │   └── 无 CRaC → new StandardStartup()
    │       └── startTime = System.currentTimeMillis() [SpringApplication:1785]
    │
    ├── shutdownHook.enableShutdownHookAddition() [SpringApplication:307] // L306-308
    │   └── shutdownHookAdditionEnabled = true [SpringApplicationShutdownHook:75] // 只标记，不立即注册
    │
    ├── createBootstrapContext() [SpringApplication:309] // L309
    │   ├── new DefaultBootstrapContext() [SpringApplication:345]
    │   │   ├── instanceSuppliers = new HashMap() [DefaultBootstrapContext:40] // 类型 → InstanceSupplier
    │   │   ├── instances = new HashMap() [DefaultBootstrapContext:42] // 类型 → 单例实例
    │   │   └── events = new SimpleApplicationEventMulticaster() [DefaultBootstrapContext:44]
    │   └── 遍历 bootstrapRegistryInitializers [SpringApplication:346]
    │       └── initializer.initialize(bootstrapContext) [SpringApplication:346]
    │           └── bootstrapContext.register(type, supplier) → instanceSuppliers.put(type, supplier) [DefaultBootstrapContext:47]
    │
    ├── configureHeadlessProperty() [SpringApplication:311] // L311
    │   └── System.setProperty("java.awt.headless", "true") [SpringApplication:449]
    │
    ├── getRunListeners(args) [SpringApplication:312] // L312
    │   ├── ArgumentResolver.of(SpringApplication.class, this).and(String[].class, args) [SpringApplication:454]
    │   ├── SpringFactoriesLoader.load(SpringApplicationRunListener.class, argumentResolver) [SpringApplication:456]
    │   │   └── 找到 EventPublishingRunListener（spring.factories 中注册） [EventPublishingRunListener:55]
    │   │       └── new EventPublishingRunListener(this, args) [EventPublishingRunListener:63]
    │   │           └── new SimpleApplicationEventMulticaster() → initialMulticaster [EventPublishingRunListener:66]
    │   ├── 检查 ThreadLocal<SpringApplicationHook>
    │   │   └── 有 hook → hook.getRunListener(this) → 追加到 listeners [SpringApplication:459]
    │   └── new SpringApplicationRunListeners(logger, listeners, applicationStartup) [SpringApplication:464]
    │
    ├── listeners.starting(bootstrapContext, mainApplicationClass) [SpringApplication:313] // L313
    │   └── SpringApplicationRunListeners.starting() [SpringApplicationRunListeners:55]
    │       └── doWithListeners("spring.boot.application.starting", listener → listener.starting(bootstrapContext)) [SpringApplicationRunListeners:56]
    │           └── EventPublishingRunListener.starting() [EventPublishingRunListener:75]
    │               └── multicastInitialEvent(new ApplicationStartingEvent(bootstrapContext, application, args)) [EventPublishingRunListener:76] ⚡
    │                   ├── refreshApplicationListeners() → initialMulticaster.addApplicationListener(each) [EventPublishingRunListener:141]
    │                   └── initialMulticaster.multicastEvent(event) [EventPublishingRunListener:137]
    │                       └── // 遍历 ApplicationListener，调用 onApplicationEvent
    │
    ├── new DefaultApplicationArguments(args) [SpringApplication:315] // L315
    │   └── new Source(args) extends SimpleCommandLinePropertySource [DefaultApplicationArguments:44]
    │       └── SimpleCommandLineArgsParser.parse(args) [SimpleCommandLinePropertySource]
    │           ├── --key=value → optionArgs.put(key, [value]) [SimpleCommandLineArgsParser]
    │           └── 非 -- 开头 → nonOptionArgs.add(arg) [SimpleCommandLineArgsParser]
    │
    ├── prepareEnvironment(listeners, bootstrapContext, applicationArguments) [SpringApplication:316] // L316
    │   ├── getOrCreateEnvironment() [SpringApplication:353] // L353
    │   │   ├── applicationContextFactory.createEnvironment(webApplicationType) [SpringApplication:480]
    │   │   │   └── DefaultApplicationContextFactory → 查 spring.factories [DefaultApplicationContextFactory:49]
    │   │   └── SERVLET → StandardServletEnvironment / NONE → ApplicationEnvironment [SpringApplication:484]
    │   │       └── StandardEnvironment 构造 → customizePropertySources() [StandardEnvironment]
    │   │           ├── systemProperties → System.getProperties() [StandardEnvironment]
    │   │           └── systemEnvironment → System.getenv() [StandardEnvironment]
    │   ├── configureEnvironment(environment, args) [SpringApplication:354] // L354
    │   │   ├── environment.setConversionService(new ApplicationConversionService()) [SpringApplication:500]
    │   │   │   └── 注册各种 Converter：StringToDuration, StringToDataSize, StringToCharset 等 [ApplicationConversionService]
    │   │   ├── configurePropertySources(environment, args) [SpringApplication:502] // L513
    │   │   │   ├── defaultProperties 非空 → DefaultPropertiesPropertySource.addOrMerge(sources) [SpringApplication:516]
    │   │   │   ├── args.length > 0 → new SimpleCommandLinePropertySource(args) → sources.addFirst() [SpringApplication:529]
    │   │   │   │   └── // 命令行参数优先级最高
    │   │   │   └── new ApplicationInfoPropertySource(mainApplicationClass) → sources.addLast() [SpringApplication:532]
    │   │   │       └── // 包含 spring.application.pid + spring.application.version
    │   │   └── configureProfiles(environment, args) [SpringApplication:503] // L543 → 空实现
    │   ├── ConfigurationPropertySources.attach(environment) [SpringApplication:355]
    │   │   └── new ConfigurationPropertySourcesPropertySource("configurationProperties", sources) [ConfigurationPropertySources]
    │   │       └── sources.addFirst(configurationProperties) → 让 Binder 能工作 [ConfigurationPropertySources]
    │   ├── listeners.environmentPrepared(bootstrapContext, environment) [SpringApplication:356]
    │   │   └── doWithListeners("...environment-prepared", listener → listener.environmentPrepared(...)) [SpringApplicationRunListeners:64]
    │   │       └── EventPublishingRunListener.environmentPrepared() [EventPublishingRunListener:80]
    │   │           └── multicastInitialEvent(new ApplicationEnvironmentPreparedEvent(...)) [EventPublishingRunListener:83] ⚡
    │   │               └── // 触发以下监听器（通过 initialMulticaster 广播）：
    │   │                   ├── ConfigDataEnvironmentPostListener → 加载 application.yml/properties
    │   │                   │   └── → ConfigDataEnvironmentPostProcessor.process()
    │   │                   │       └── → 读取 classpath:application.yml + application-{profile}.yml
    │   │                   │           └── → 处理 spring.config.import（Nacos 等）
    │   │                   │               └── → 加入 PropertySource 到 environment
    │   │                   ├── LoggingApplicationListener → 初始化日志系统
    │   │                   │   └── → LoggingSystem.get(classLoader) → Logback/Log4j2
    │   │                   │       └── → loggingSystem.beforeInitialize() + initialize()
    │   │                   └── 各种 EnvironmentPostProcessor → 后置处理
    │   ├── ApplicationInfoPropertySource.moveToEnd(environment) [SpringApplication:357]
    │   │   └── propertySources.remove("applicationInfo") → addLast() [ApplicationInfoPropertySource:78]
    │   ├── DefaultPropertiesPropertySource.moveToEnd(environment) [SpringApplication:358]
    │   ├── bindToSpringApplication(environment) [SpringApplication:361] // L361
    │   │   └── Binder.get(environment).bind("spring.main", Bindable.ofInstance(this.properties)) [SpringApplication:552]
    │   │       ├── Binder.bind() → ConfigurationPropertyName.of("spring.main") [Binder]
    │   │       │   └── 遍历 properties 的 setter，从 environment 取值绑定 [Binder]
    │   │       │       ├── spring.main.lazy-initialization → setLazyInitialization() [ApplicationProperties:130]
    │   │       │       ├── spring.main.banner-mode → setBannerMode() [ApplicationProperties:114]
    │   │       │       ├── spring.main.web-application-type → setWebApplicationType() [ApplicationProperties:162]
    │   │       │       ├── spring.main.allow-circular-references → setAllowCircularReferences() [ApplicationProperties:100]
    │   │       │       └── spring.main.allow-bean-definition-overriding → setAllowBeanDefinitionOverriding() [ApplicationProperties:92]
    │   │       └── 绑定失败 → throw IllegalStateException [SpringApplication:555]
    │   ├── EnvironmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass()) [SpringApplication:363]
    │   │   ├── type.equals(environment.getClass()) → 原样返回 [EnvironmentConverter:75]
    │   │   └── 不匹配 → convertEnvironment() [EnvironmentConverter:78]
    │   │       ├── 反射创建目标 Environment → setActiveProfiles + setConversionService [EnvironmentConverter:83]
    │   │       └── copyPropertySources → 排除 servlet 相关 PropertySource [EnvironmentConverter:101]
    │   └── ConfigurationPropertySources.attach()（再次） [SpringApplication:366]
    │
    ├── printBanner(environment) [SpringApplication:317] // L317
    │   ├── properties.getBannerMode(environment) [SpringApplication:560]
    │   │   ├── bannerMode != null → 用用户设置的 [ApplicationProperties:105]
    │   │   └── null → 检查结构化日志 → 有 → OFF；无 → CONSOLE [ApplicationProperties:111]
    │   ├── new SpringApplicationBannerPrinter(resourceLoader, this.banner) [SpringApplication:565]
    │   │   └── getBanner(environment) [SpringApplicationBannerPrinter:73]
    │   │       ├── getTextBanner(environment) → 找 banner.txt [SpringApplicationBannerPrinter:84]
    │   │       │   └── environment.getProperty("spring.banner.location", "banner.txt") [SpringApplicationBannerPrinter:85]
    │   │       │       └── resourceLoader.getResource(location) → resource.exists() → ResourceBanner [SpringApplicationBannerPrinter:86]
    │   │       ├── fallbackBanner != null → 用自定义 [SpringApplicationBannerPrinter:78]
    │   │       └── DEFAULT_BANNER → SpringBootBanner [SpringApplicationBannerPrinter:45]
    │   │           └── printBanner() → println(ASCII Spring Logo + version) [SpringBootBanner:49]
    │   └── CONSOLE → banner.printBanner(env, sourceClass, System.out) [SpringApplicationBannerPrinter:67]
    │       └── LOG → createStringFromBanner() → logger.info() [SpringApplicationBannerPrinter:56]
    │           └── 返回 PrintedBanner（装饰器） [SpringApplicationBannerPrinter:112]
    │
    ├── createApplicationContext() [SpringApplication:318] // L318
    │   ├── applicationContextFactory.create(webApplicationType) [SpringApplication:580]
    │   │   └── DefaultApplicationContextFactory.create() [DefaultApplicationContextFactory:55]
    │   │       ├── getFromSpringFactories(webApplicationType, ApplicationContextFactory::create, ...) [DefaultApplicationContextFactory:57]
    │   │       │   └── SpringFactoriesLoader.load(ApplicationContextFactory.class) → 逐个尝试 [DefaultApplicationContextFactory:77]
    │   │       └── createDefaultApplicationContext() [DefaultApplicationContextFactory:66]
    │   │           ├── 非 AOT → new AnnotationConfigApplicationContext() [DefaultApplicationContextFactory:68]
    │   │           │   ├── super() → GenericApplicationContext() [AnnotationConfigApplicationContext]
    │   │           │   │   ├── new DefaultListableBeanFactory() → beanFactory [GenericApplicationContext]
    │   │           │   │   └── setBeanFactory(beanFactory) [GenericApplicationContext]
    │   │           │   ├── new AnnotatedBeanDefinitionReader(this) [AnnotationConfigApplicationContext]
    │   │           │   │   └── AnnotationConfigUtils.registerAnnotationConfigProcessors(registry) [AnnotationConfigUtils]
    │   │           │   │       ├── 注册 ConfigurationClassPostProcessor（处理 @Configuration） [AnnotationConfigUtils]
    │   │           │   │       ├── 注册 AutowiredAnnotationBeanPostProcessor（处理 @Autowired） [AnnotationConfigUtils]
    │   │           │   │       ├── 注册 CommonAnnotationBeanPostProcessor（处理 @PostConstruct/@Resource） [AnnotationConfigUtils]
    │   │           │   │       └── 注册 EventListenerMethodProcessor（处理 @EventListener） [AnnotationConfigUtils]
    │   │           │   └── new ClassPathBeanDefinitionScanner(this) [AnnotationConfigApplicationContext]
    │   │           │       └── 注册默认 TypeFilter（@Component/@ManagedBean） [ClassPathBeanDefinitionScanner]
    │   │           └── AOT → new GenericApplicationContext() [DefaultApplicationContextFactory:70]
    │   └── Assert.state(context != null) [SpringApplication:582]
    │
    ├── context.setApplicationStartup(this.applicationStartup) [SpringApplication:319] // L319
    │
    ├── prepareContext(bootstrapContext, context, environment, listeners, ...) [SpringApplication:320] // L320
    │   ├── context.setEnvironment(environment) [SpringApplication:383] // L383
    │   ├── postProcessApplicationContext(context) [SpringApplication:384] // L384
    │   │   ├── beanNameGenerator != null → registerSingleton("...BeanNameGenerator", beanNameGenerator) [SpringApplication:593]
    │   │   ├── resourceLoader != null → setResourceLoader() + setClassLoader() [SpringApplication:596]
    │   │   └── addConversionService → beanFactory.setConversionService(env.getConversionService()) [SpringApplication:604]
    │   ├── addAotGeneratedInitializerIfNecessary(this.initializers) [SpringApplication:385] // L385
    │   │   └── AotDetector.useGeneratedArtifacts() → 查找 __ApplicationContextInitializer 类 [SpringApplication:422]
    │   ├── 配置 BeanFactory [SpringApplication:387] // L387-392
    │   │   ├── autowireCapableBeanFactory.setAllowCircularReferences(properties.isAllowCircularReferences()) [SpringApplication:388]
    │   │   └── listableBeanFactory.setAllowBeanDefinitionOverriding(properties.isAllowBeanDefinitionOverriding()) [SpringApplication:390]
    │   ├── applyInitializers(context) [SpringApplication:393] // L393
    │   │   └── 遍历 getInitializers()（按 @Order 排序） [SpringApplication:617]
    │   │       └── GenericTypeResolver.resolveTypeArgument(initializer.getClass(), ApplicationContextInitializer.class) [SpringApplication:618]
    │   │           └── 断言 requiredType.isInstance(context) → initializer.initialize(context) [SpringApplication:623]
    │   │               └── // 各 initializer 对 context 做定制（如设置 contextId、注册额外 Bean 等）
    │   ├── listeners.contextPrepared(context) [SpringApplication:394] // L394
    │   │   └── EventPublishingRunListener.contextPrepared() [EventPublishingRunListener:87]
    │   │       └── multicastInitialEvent(new ApplicationContextInitializedEvent(application, args, context)) [EventPublishingRunListener:88] ⚡
    │   ├── bootstrapContext.close(context) [SpringApplication:395] // L395
    │   │   └── events.multicastEvent(new BootstrapContextClosedEvent(this, applicationContext)) [DefaultBootstrapContext:140] ⚡
    │   │       └── addCloseListener 注册的回调 → 把 BootstrapContext 实例迁移到 context [DefaultBootstrapContext:84]
    │   ├── logStartupInfo(context) [SpringApplication:396] // L396
    │   │   └── StartupInfoLogger.logStarting(applicationLog) [SpringApplication:636]
    │   │       └── "Starting MyApplication v1.0 using Java 17 with PID 12345 (/path started by user in /dir)" [StartupInfoLogger:64]
    │   ├── logStartupProfileInfo(context) [SpringApplication:398] // L398
    │   │   └── activeProfiles 非空 → "The following 1 profile is active: \"dev\"" [SpringApplication:656]
    │   │   └── activeProfiles 为空 → "No active profile set, falling back to default profiles" [SpringApplication:652]
    │   ├── beanFactory.registerSingleton("springApplicationArguments", applicationArguments) [SpringApplication:401] // L401
    │   ├── printedBanner != null → beanFactory.registerSingleton("springBootBanner", printedBanner) [SpringApplication:403] // L403
    │   ├── properties.isLazyInitialization() → context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor()) [SpringApplication:406]
    │   ├── properties.isKeepAlive() → context.addApplicationListener(new KeepAlive()) [SpringApplication:409]
    │   ├── context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context)) [SpringApplication:411]
    │   ├── getAllSources() [SpringApplication:414] // L414
    │   │   ├── allSources.addAll(primarySources) → MyApplication.class [SpringApplication:1191]
    │   │   └── allSources.addAll(properties.getSources()) → spring.main.sources 额外配置 [SpringApplication:1194]
    │   ├── load(context, sources.toArray()) [SpringApplication:416] // L416
    │   │   ├── createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources) [SpringApplication:687]
    │   │   │   ├── new AnnotatedBeanDefinitionReader(registry) [BeanDefinitionLoader:88]
    │   │   │   │   └── AnnotationConfigUtils.registerAnnotationConfigProcessors() → 注册后处理器 [BeanDefinitionLoader:88]
    │   │   │   ├── new XmlBeanDefinitionReader(registry) [BeanDefinitionLoader:89]
    │   │   │   ├── isGroovyPresent() → new GroovyBeanDefinitionReader(registry) [BeanDefinitionLoader:90]
    │   │   │   └── new ClassPathBeanDefinitionScanner(registry) [BeanDefinitionLoader:91]
    │   │   │       └── addExcludeFilter(new ClassExcludeFilter(sources)) → 防止重复注册 [BeanDefinitionLoader:92]
    │   │   └── loader.load() [BeanDefinitionLoader:128]
    │   │       └── 遍历 sources → load(source) [BeanDefinitionLoader:129]
    │   │           ├── Class → isEligible(source) → annotatedReader.register(source) [BeanDefinitionLoader:162]
    │   │           │   └── // AnnotatedBeanDefinitionReader.register()
    │   │           │       └── → registerBeanDefinition(beanName, beanDefinition)
    │   │           │           └── → 解析 @Configuration/@Component/@Bean/@Import/@ComponentScan
    │   │           │               └── （实际解析在 refresh() 的 ConfigurationClassPostProcessor 中）
    │   │           ├── Resource → xmlReader.loadBeanDefinitions(resource) [BeanDefinitionLoader:176]
    │   │           │   └── → 读取 XML → 注册 BeanDefinition
    │   │           ├── Package → scanner.scan(packageName) [BeanDefinitionLoader:181]
    │   │           │   └── → ClassPath扫描 → 匹配 @Component 的类 → 注册 BeanDefinition
    │   │           └── CharSequence → 尝试 Class.forName → 尝试 Resource → 尝试 Package [BeanDefinitionLoader:184]
    │   └── listeners.contextLoaded(context) [SpringApplication:418] // L418
    │       └── EventPublishingRunListener.contextLoaded() [EventPublishingRunListener:92]
    │           ├── 遍历 application.getListeners() [EventPublishingRunListener:93]
    │           │   ├── instanceof ApplicationContextAware → setApplicationContext(context) [EventPublishingRunListener:94]
    │           │   └── context.addApplicationListener(listener) → 注册到 context [EventPublishingRunListener:97]
    │           └── multicastInitialEvent(new ApplicationPreparedEvent(application, args, context)) [EventPublishingRunListener:99] ⚡
    │               // 此后事件通过 context 的 multicaster 广播
    │
    ├── refreshContext(context) [SpringApplication:321] // L321
    │   ├── shutdownHook.registerApplicationContext(context) [SpringApplication:443] // L443
    │   │   ├── addRuntimeShutdownHookIfNecessary() [SpringApplicationShutdownHook:78]
    │   │   │   └── shutdownHookAdditionEnabled && CAS(false→true) → Runtime.getRuntime().addShutdownHook(new Thread("SpringApplicationShutdownHook")) [SpringApplicationShutdownHook:88]
    │   │   ├── context.addApplicationListener(contextCloseListener) [SpringApplicationShutdownHook:82]
    │   │   │   └── ApplicationContextClosedListener → ContextClosedEvent 时移到 closedContexts [SpringApplicationShutdownHook:242]
    │   │   └── contexts.add(context) [SpringApplicationShutdownHook:83]
    │   └── refresh(context) → context.refresh() [SpringApplication:445] ★ Spring Framework 核心 L756
    │       │
    │       ├── prepareRefresh() [AbstractApplicationContext] // AbstractApplicationContext
    │       │   ├── prepareBeanFactory() 前的初始化 [AbstractApplicationContext]
    │       │   ├── initPropertySources() → 子类可覆盖 [AbstractApplicationContext]
    │       │   │   └── ServletWebServerApplicationContext → initPropertySources() 加 servletContext
    │       │   ├── getEnvironment().validateRequiredProperties() [AbstractApplicationContext]
    │       │   │   └── 检查 @Value("${xxx}") 中 required 的属性是否存在
    │       │   └── earlyApplicationEvents = new LinkedHashSet<>()
    │       │
    │       ├── obtainFreshBeanFactory() [AbstractApplicationContext]
    │       │   ├── refreshBeanFactory() [AbstractApplicationContext]
    │       │   │   └── GenericApplicationContext → 已有 beanFactory，只设置 serializationId
    │       │   └── getBeanFactory() → 返回 DefaultListableBeanFactory [AbstractApplicationContext]
    │       │
    │       ├── prepareBeanFactory(beanFactory) [AbstractApplicationContext]
    │       │   ├── beanFactory.setBeanClassLoader(getClassLoader()) [AbstractApplicationContext]
    │       │   ├── beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver()) [AbstractApplicationContext]
    │       │   │   └── 支持 #{...} SpEL 表达式
    │       │   ├── beanFactory.addEmbeddedValueResolver(new EnvironmentStringValueResolver()) [AbstractApplicationContext]
    │       │   │   └── 支持 ${...} 属性占位符
    │       │   ├── beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this)) [AbstractApplicationContext]
    │       │   │   └── 处理 EnvironmentAware/ResourceLoaderAware/ApplicationEventPublisherAware/ApplicationContextAware
    │       │   ├── beanFactory.ignoreDependencyInterface(BeanNameAware.class) [AbstractApplicationContext]
    │       │   │   └── Aware 接口由 ApplicationContextAwareProcessor 处理，不走自动注入
    │       │   ├── beanFactory.registerResolvableDependency(BeanFactory.class, this) [AbstractApplicationContext]
    │       │   │   └── 注入 BeanFactory/ApplicationContext 时直接注入 this
    │       │   └── beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this)) [AbstractApplicationContext]
    │       │       └── Bean 初始化后检测是否是 ApplicationListener，如果是则注册到 multicaster
    │       │
    │       ├── postProcessBeanFactory(beanFactory) [AbstractApplicationContext]
    │       │   └── ServletWebServerApplicationContext → 添加 WebApplicationContextAwareProcessor
    │       │       └── 处理 ServletContextAware/ServletConfigAware
    │       │
    │       ├── invokeBeanFactoryPostProcessors(beanFactory) [AbstractApplicationContext] ★★★ 最核心
    │       │   └── PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors() [PostProcessorRegistrationDelegate]
    │       │       │
    │       │       ├── ① 执行 BeanDefinitionRegistryPostProcessor（PriorityOrdered） [PostProcessorRegistrationDelegate]
    │       │       │   └── ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry() [ConfigurationClassPostProcessor]
    │       │       │       └── // 处理 @Configuration 类的解析
    │       │       │       ├── processConfigBeanDefinitions(registry) [ConfigurationClassPostProcessor]
    │       │       │       │   ├── 遍历 BeanDefinition → 检查 @Configuration/@Component/@Import
    │       │       │       │   │   └── ConfigurationClassUtils.checkConfigurationClassAttribute() → full/lite
    │       │       │       │   ├── new ConfigurationClassParser() → 解析每个 @Configuration 类 [ConfigurationClassPostProcessor]
    │       │       │       │   │   ├── parser.parse(candidates) → processConfigurationClass() [ConfigurationClassParser]
    │       │       │       │   │   │   ├── → 处理 @PropertySource → 加载属性文件 [ConfigurationClassParser]
    │       │       │       │   │   │   ├── → 处理 @ComponentScan → ClassPathBeanDefinitionScanner.scan() [ConfigurationClassParser]
    │       │       │       │   │   │   │   └── → 扫描包下 @Component/@Service/@Repository/@Controller [ClassPathBeanDefinitionScanner]
    │       │       │   │   │   │   │       └── → registerBeanDefinition() → 注册到 BeanFactory [ClassPathBeanDefinitionScanner]
    │       │       │   │   │   │   ├── → 处理 @Import → ImportSelector/ImportBeanDefinitionRegistrar [ConfigurationClassParser]
    │       │       │   │   │   │   │   ├── AutoConfigurationImportSelector.selectImports() [AutoConfigurationImportSelector]
    │       │       │   │   │   │   │   │   └── → 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports [AutoConfigurationImportSelector]
    │       │       │   │   │   │   │   │       └── → 批量返回自动配置类名 → 注册为 BeanDefinition [AutoConfigurationImportSelector]
    │       │       │   │   │   │   │   └── 其他 ImportBeanDefinitionRegistrar → registerBeanDefinitions() [ImportBeanDefinitionRegistrar]
    │       │       │   │   │   │   ├── → 处理 @ImportResource → 加载 XML BeanDefinition [ConfigurationClassParser]
    │       │       │   │   │   │   └── → 处理 @Bean 方法 → 注册为 BeanDefinition [ConfigurationClassParser]
    │       │       │   │   │   │       └── → ConfigurationClassBeanDefinitionReader.loadBeanDefinitions() [ConfigurationClassBeanDefinitionReader]
    │       │       │   │   │   │           └── → 为每个 @Bean 方法创建 RootBeanDefinition [ConfigurationClassBeanDefinitionReader]
    │       │       │   │   │   │               └── → registry.registerBeanDefinition(beanName, beanDefinition) [ConfigurationClassBeanDefinitionReader]
    │       │       │   │   │   └── parser.validate() → 校验配置类合法性 [ConfigurationClassParser]
    │       │       │   │   └── 再次检查是否有新的 @Configuration → 循环直到没有新的 [ConfigurationClassPostProcessor]
    │       │       │       └── // ★ 此时所有 BeanDefinition 已注册到 BeanFactory
    │       │       │
    │       │       ├── ② 执行 BeanFactoryPostProcessor（PriorityOrdered + Ordered + 普通） [PostProcessorRegistrationDelegate]
    │       │       │   ├── ConfigurationClassPostProcessor.postProcessBeanFactory() [ConfigurationClassPostProcessor:298]
    │       │       │   │   └── enhanceConfigurationClasses(beanFactory) → CGLIB 增强 @Configuration 类 [ConfigurationClassPostProcessor:477]
    │       │       │   │       ├── 遍历 BeanDefinition → 找 CONFIGURATION_CLASS_FULL 的 [ConfigurationClassPostProcessor:507]
    │       │       │   │       ├── new ConfigurationClassEnhancer() [ConfigurationClassPostProcessor:532]
    │       │       │   │       │   └── enhancer.enhance(configClass, classLoader) → CGLIB 生成子类 [ConfigurationClassEnhancer]
    │       │       │   │       │       └── 拦截 @Bean 方法 → 保证单例语义 [ConfigurationClassEnhancer]
    │       │       │   │       └── beanDef.setBeanClass(enhancedClass) → 替换 BeanDefinition 的 class [ConfigurationClassPostProcessor:545]
    │       │       │   │   └── 添加 ImportAwareBeanPostProcessor → 处理 @Import 注入 [ConfigurationClassPostProcessor:552]
    │       │       │   ├── PropertySourceOrderingBeanFactoryPostProcessor [SpringApplication:1594]
    │       │       │   │   └── DefaultPropertiesPropertySource.moveToEnd() → 调整属性源顺序 [SpringApplication:1609]
    │       │       │   ├── LazyInitializationBeanFactoryPostProcessor [SpringApplication]
    │       │       │   │   └── 遍历 BeanDefinition → setLazyInit(true) → 懒加载 [LazyInitializationBeanFactoryPostProcessor]
    │       │       │   └── 其他 BeanFactoryPostProcessor（如 PropertySourcesPlaceholderConfigurer） [PropertySourcesPlaceholderConfigurer]
    │       │       │       └── 处理 ${...} 占位符替换 [PropertySourcesPlaceholderConfigurer]
    │       │       │
    │       │       └── ③ 再次检查是否有新的 BeanDefinitionRegistryPostProcessor（循环直到没有新的） [PostProcessorRegistrationDelegate]
    │       │
    │       ├── registerBeanPostProcessors(beanFactory) [AbstractApplicationContext]
    │       │   └── PostProcessorRegistrationDelegate.registerBeanPostProcessors() [PostProcessorRegistrationDelegate]
    │       │       ├── ① PriorityOrdered BeanPostProcessor → 排序 → addBeanPostProcessor() [PostProcessorRegistrationDelegate]
    │       │       │   └── 如：CommonAnnotationBeanPostProcessor（@PostConstruct/@Resource）
    │       │       ├── ② Ordered BeanPostProcessor → 排序 → addBeanPostProcessor() [PostProcessorRegistrationDelegate]
    │       │       │   └── 如：AutowiredAnnotationBeanPostProcessor（@Autowired/@Value）
    │       │       ├── ③ 普通 BeanPostProcessor → addBeanPostProcessor() [PostProcessorRegistrationDelegate]
    │       │       │   └── 如：AbstractAutoProxyCreator（AOP 代理创建器）
    │       │       └── ④ ApplicationListenerDetector → 重新注册到最后 [PostProcessorRegistrationDelegate]
    │       │           └── 确保在所有 BeanPostProcessor 之后执行
    │       │
    │       ├── initMessageSource() [AbstractApplicationContext]
    │       │   └── 检查是否有 messageSource Bean → 没有则注册 DelegatingMessageSource [AbstractApplicationContext]
    │       │       └── 用于国际化（i18n）消息解析
    │       │
    │       ├── initApplicationEventMulticaster() [AbstractApplicationContext]
    │       │   ├── 检查是否有 applicationEventMulticaster Bean → 有则用 [AbstractApplicationContext]
    │       │   └── 没有 → new SimpleApplicationEventMulticaster() → registerSingleton [AbstractApplicationContext]
    │       │       └── 默认用同步执行器，可配置 taskExecutor 实现异步
    │       │
    │       ├── onRefresh() [AbstractApplicationContext] ★ 创建嵌入式 Web 服务器（详见专题三）
    │       │   └── ServletWebServerApplicationContext.onRefresh() [ServletWebServerApplicationContext:161]
    │       │       └── createWebServer() [ServletWebServerApplicationContext:183]
    │       │           ├── getWebServerFactory() [ServletWebServerApplicationContext:214]
    │       │           │   └── → getBeanNamesForType(ServletWebServerFactory) → getBean("tomcatServletWebServerFactory")
    │       │           ├── factory.getWebServer(getSelfInitializer()) [TomcatServletWebServerFactory:163]
    │       │           │   ├── createTomcat(tempDirs) [TomcatWebServerFactory:389]
    │       │           │   │   ├── → new Tomcat() → 内部创建 Server/Service/Engine/Host
    │       │           │   │   ├── → new Connector("Http11NioProtocol") → NioEndpoint + CoyoteAdapter
    │       │           │   │   ├── → customizeConnector: setPort(8080) + SSL + 压缩 + HTTP2 [TomcatWebServerFactory:415]
    │       │           │   │   ├── → configureEngine: setBackgroundProcessorDelay + addValve [TomcatWebServerFactory:479]
    │       │           │   │   └── → tomcat.getHost().setAutoDeploy(false)
    │       │           │   ├── prepareContext(host, initializers, tempDirs) [TomcatServletWebServerFactory:189]
    │       │           │   │   ├── → new TomcatEmbeddedContext() → 设 DocBase/Resources/Locale/TLD
    │       │           │   │   ├── → new WebappLoader(TomcatEmbeddedWebappClassLoader) → delegate=true
    │       │           │   │   ├── → addDefaultServlet("/") + addJspServlet("*.jsp")
    │       │           │   │   ├── → configureContext: Session/Cookie/ErrorPage/MimeMappings/Valves [TomcatServletWebServerFactory:312]
    │       │           │   │   └── → host.addChild(context)
    │       │           │   └── getTomcatWebServer(tomcat) [TomcatServletWebServerFactory:428]
    │       │           │       └── new TomcatWebServer(tomcat, autoStart, shutdown) [TomcatWebServer:105]
    │       │           │           └── initialize() [TomcatWebServer:113] ★ Tomcat 初始化
    │       │           │               ├── → addInstanceIdToEngineName [TomcatWebServer:164]
    │       │           │               ├── → context.addLifecycleListener → START_EVENT 时 removeServiceConnectors [TomcatWebServer:120]
    │       │           │               │   ★ 移除 Connector，让 tomcat.start() 不绑定端口
    │       │           │               ├── → disableBindOnInit → bindOnInit=false [TomcatWebServer:181]
    │       │           │               ├── → tomcat.start() → Server→Service→Engine→Host→Context 启动 [TomcatWebServer:131]
    │       │           │               │   → Context START_EVENT → removeServiceConnectors 回调
    │       │           │               │   → ServletContainerInitializer.onStartup
    │       │           │               │   → WebApplicationContextInitializer.initialize(servletContext) [WebApplicationContextInitializer:48]
    │       │           │               │   ├── → prepareWebApplicationContext → setAttribute(ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE)
    │       │           │               │   ├── → registerApplicationScope → new ServletContextScope
    │       │           │               │   └── → 遍历 ServletContextInitializerBeans → onStartup
    │       │           │               │       → 注册 DispatcherServlet("/") + Filter("/*")
    │       │           │               │   → loadOnStartup: DefaultServlet + DispatcherServlet + JspServlet
    │       │           │               ├── → rethrowDeferredStartupExceptions [TomcatWebServer:199]
    │       │           │               └── → startNonDaemonAwaitThread → 非守护线程保持 JVM 存活 [TomcatWebServer:211]
    │       │           ├── registerSingleton("webServerGracefulShutdown", new WebServerGracefulShutdownLifecycle) [ServletWebServerApplicationContext:193]
    │       │           └── registerSingleton("webServerStartStop", new WebServerStartStopLifecycle) [ServletWebServerApplicationContext:195]
    │       │               ★ SmartLifecycle，在 finishRefresh() 中被 start() → 真正绑定端口
    │       │
    │       ├── registerListeners() [AbstractApplicationContext]
    │       │   ├── 遍历 applicationListeners → applicationEventMulticaster.addApplicationListener() [AbstractApplicationContext]
    │       │   ├── 遍历 BeanDefinition 中 ApplicationListener 类型 → addListenerBeanName() [AbstractApplicationContext]
    │       │   └── 发布早期事件 earlyApplicationEvents → multicastEvent() [AbstractApplicationContext]
    │       │
    │       ├── finishBeanFactoryInstantiation(beanFactory) [AbstractApplicationContext] ★ 实例化所有非懒加载单例 Bean
    │       │   └── beanFactory.preInstantiateSingletons() [DefaultListableBeanFactory] // DefaultListableBeanFactory
    │       │       └── 遍历 beanDefinitionNames（按注册顺序） [DefaultListableBeanFactory]
    │       │           └── getBean(beanName) → doGetBean(beanName) [AbstractBeanFactory]
    │       │               │
    │       │               ├── getSingleton(beanName, true) [DefaultSingletonBeanRegistry:210] // 先查缓存
    │       │               │   ├── singletonObjects.get(beanName) → 一级缓存有 → 直接返回 [DefaultSingletonBeanRegistry:212]
    │       │               │   ├── 一级没有 + isSingletonCurrentlyInCreation → earlySingletonObjects.get() [DefaultSingletonBeanRegistry:214]
    │       │               │   │   └── 二级缓存有 → 返回早期引用（循环依赖场景） [DefaultSingletonBeanRegistry:214]
    │       │               │   └── 二级没有 + allowEarlyReference → singletonFactories.get() [DefaultSingletonBeanRegistry:226]
    │       │               │       └── 三级缓存有 → 执行 lambda → getEarlyBeanReference() [DefaultSingletonBeanRegistry:228]
    │       │               │           ├── → AbstractAutoProxyCreator.getEarlyBeanReference() → 可能创建代理 [AbstractAutoProxyCreator:265]
    │       │               │           ├── → earlySingletonObjects.put() → 放入二级缓存 [DefaultSingletonBeanRegistry:231]
    │       │               │           └── → singletonFactories.remove() → 清理三级缓存 [DefaultSingletonBeanRegistry:230]
    │       │               │
    │       │               ├── 缓存没有 → createBean(beanName, mbd, args) [AbstractAutowireCapableBeanFactory:518] // AbstractAutowireCapableBeanFactory L518
    │       │               │   ├── resolveBeanClass(mbd, beanName) → 解析目标 Class [AbstractAutowireCapableBeanFactory:1190]
    │       │               │   ├── resolveBeforeInstantiation(beanName, mbd) [AbstractAutowireCapableBeanFactory:1136] // L1136
    │       │               │   │   ├── applyBeanPostProcessorsBeforeInstantiation(beanClass, beanName) [AbstractAutowireCapableBeanFactory:1166]
    │       │               │   │   │   └── AbstractAutoProxyCreator.postProcessBeforeInstantiation() L273 [AbstractAutoProxyCreator:273]
    │       │               │   │   │       ├── isInfrastructureClass() → Advice/Advisor/Pointcut → 跳过 [AbstractAutoProxyCreator:392]
    │       │               │   │   │       ├── shouldSkip() → 原始实例 → 跳过 [AbstractAutoProxyCreator:415]
    │       │               │   │   │       └── getCustomTargetSource() → 有则 createProxy() 返回代理 [AbstractAutoProxyCreator:289]
    │       │               │   │   │           └── 返回非 null → applyBeanPostProcessorsAfterInitialization() → 直接返回 [AbstractAutowireCapableBeanFactory:1145]
    │       │               │   │   └── 返回 null → 继续正常创建 [AbstractAutowireCapableBeanFactory]
    │       │               │   │
    │       │               │   └── doCreateBean(beanName, mbd, args) [AbstractAutowireCapableBeanFactory:560] // L560 ★★★
    │       │               │       │
    │       │               │       ├── ① createBeanInstance(beanName, mbd, args) [AbstractAutowireCapableBeanFactory:1188] // L1188
    │       │               │       │   ├── mbd.getInstanceSupplier() != null → obtainFromSupplier() [AbstractAutowireCapableBeanFactory:1198]
    │       │               │       │   ├── mbd.getFactoryMethodName() != null → instantiateUsingFactoryMethod() [AbstractAutowireCapableBeanFactory:1204]
    │       │               │       │   │   └── → 解析工厂方法参数 → autowireConstructor() → 反射调用
    │       │               │       │   ├── 有构造器参数 → autowireConstructor() [AbstractAutowireCapableBeanFactory]
    │       │               │       │   │   └── → resolveConstructorArguments() → getBean(依赖) → 反射创建
    │       │               │       │   └── 无参 → instantiateBean() [AbstractAutowireCapableBeanFactory]
    │       │               │       │       └── → BeanUtils.instantiateClass(constructor) → 反射 newInstance()
    │       │               │       │           └── 得到 raw bean（原始对象，还没注入属性）
    │       │               │       │
    │       │               │       ├── ② applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName) [AbstractAutowireCapableBeanFactory:581] // L581
    │       │               │       │   └── 遍历 MergedBeanDefinitionPostProcessor
    │       │               │       │       ├── AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition() [AutowiredAnnotationBeanPostProcessor]
    │       │               │       │       │   └── findAutowiringMetadata() → 解析 @Autowired/@Value 字段和方法 [AutowiredAnnotationBeanPostProcessor]
    │       │               │       │       │       └── → 缓存 InjectionMetadata 到 checkedElements
    │       │               │       │       └── CommonAnnotationBeanPostProcessor.postProcessMergedBeanDefinition() [CommonAnnotationBeanPostProcessor]
    │       │               │       │           └── 解析 @Resource/@PostConstruct/@PreDestroy [CommonAnnotationBeanPostProcessor]
    │       │               │       │
    │       │               │       ├── ③ addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean)) [AbstractAutowireCapableBeanFactory:600] // L600
    │       │               │       │   └── DefaultSingletonBeanRegistry.addSingletonFactory() L183 [DefaultSingletonBeanRegistry:183]
    │       │               │       │       ├── singletonFactories.put(beanName, lambda) → 三级缓存 [DefaultSingletonBeanRegistry:185]
    │       │               │       │       ├── earlySingletonObjects.remove(beanName) → 清理二级 [DefaultSingletonBeanRegistry:186]
    │       │               │       │       └── registeredSingletons.add(beanName) [DefaultSingletonBeanRegistry:187]
    │       │               │       │       // 条件：mbd.isSingleton() && allowCircularReferences && isSingletonCurrentlyInCreation
    │       │               │       │       // ★ lambda 此时未执行！只有循环依赖时才被调用
    │       │               │       │
    │       │               │       ├── ④ populateBean(beanName, mbd, instanceWrapper) [AbstractAutowireCapableBeanFactory:1405] // L1405 ★ 属性注入
    │       │               │       │   ├── InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation() [AbstractAutowireCapableBeanFactory:1432]
    │       │               │       │   │   └── → 返回 false 则跳过属性注入
    │       │               │       │   ├── autowireByName() / autowireByType() [AbstractAutowireCapableBeanFactory:1442]
    │       │               │       │   │   └── → unsatisfiedNonSimpleProperties() → getBean(propertyName)
    │       │               │       │   │       └── → registerDependentBean() → 记录依赖关系
    │       │               │       │   ├── InstantiationAwareBeanPostProcessor.postProcessProperties(pvs, bean, beanName) [AbstractAutowireCapableBeanFactory:1458]
    │       │               │       │   │   ├── AutowiredAnnotationBeanPostProcessor.postProcessProperties() [AutowiredAnnotationBeanPostProcessor]
    │       │               │       │   │   │   └── InjectionMetadata.inject(bean, beanName, pvs) [InjectionMetadata]
    │       │               │       │   │   │       └── 遍历 @Autowired 字段 → resolveDependency() [InjectionMetadata]
    │       │               │       │   │   │           └── DefaultListableBeanFactory.resolveDependency() [DefaultListableBeanFactory]
    │       │               │       │   │   │               ├── @Qualifier → 按名字匹配 [DefaultListableBeanFactory]
    │       │               │       │   │   │               ├── @Primary → 优先选择 [DefaultListableBeanFactory]
    │       │               │       │   │   │               ├── @Priority → 按优先级 [DefaultListableBeanFactory]
    │       │               │       │   │   │               └── 按类型匹配 → getBean(candidateName) [DefaultListableBeanFactory]
    │       │               │       │   │   │                   └── ★ 可能触发循环依赖！
    │       │               │       │   │   │                       getBean(B) → B.doCreateBean → B.populateBean → getBean(A)
    │       │               │       │   │   │                       → getSingleton(A, allowEarlyReference=true)
    │       │               │       │   │   │                       → 三级缓存 lambda 执行 → getEarlyBeanReference()
    │       │               │       │   │   │                       → AbstractAutoProxyCreator.getEarlyBeanReference() L265
    │       │               │       │   │   │                       → earlyBeanReferences.put(cacheKey, bean) — 记录
    │       │               │       │   │   │                       → wrapIfNecessary() → getAdvicesAndAdvisorsForBean()
    │       │               │       │   │   │                       → 有 Advisor → createProxy() → 提前创建 AOP 代理
    │       │               │       │   │   │                       → earlySingletonObjects.put(A, 代理) → 二级缓存
    │       │               │       │   │   │                       → singletonFactories.remove(A) → 清理三级
    │       │               │       │   │   │                       → 返回 A 的早期引用给 B
    │       │               │       │   │   └── CommonAnnotationBeanPostProcessor.postProcessProperties() [CommonAnnotationBeanPostProcessor]
    │       │               │       │   │       └── 处理 @Resource 注入 [CommonAnnotationBeanPostProcessor]
    │       │               │       │   └── applyPropertyValues(beanName, mbd, bw, pvs) [AbstractAutowireCapableBeanFactory:1474]
    │       │               │       │       └── → 将解析后的 Bean 引用设置到 BeanWrapper
    │       │               │       │           └── → BeanWrapper.setPropertyValue() → 反射 setter 或直接字段
    │       │               │       │
    │       │               │       ├── ⑤ initializeBean(beanName, exposedObject, mbd) [AbstractAutowireCapableBeanFactory:1813] // L1813 ★ 初始化
    │       │               │       │   │
    │       │               │       │   ├── invokeAwareMethods(beanName, bean) [AbstractAutowireCapableBeanFactory:1835] // L1835
    │       │               │       │   │   ├── BeanNameAware.setBeanName(beanName) [AbstractAutowireCapableBeanFactory:1837]
    │       │               │       │   │   ├── BeanClassLoaderAware.setBeanClassLoader(classLoader) [AbstractAutowireCapableBeanFactory:1840]
    │       │               │       │   │   └── BeanFactoryAware.setBeanFactory(this) [AbstractAutowireCapableBeanFactory:1846]
    │       │               │       │   │
    │       │               │       │   ├── applyBeanPostProcessorsBeforeInitialization(bean, beanName) [AbstractAutowireCapableBeanFactory:424] // L424
    │       │               │       │   │   └── 遍历 BeanPostProcessor → postProcessBeforeInitialization() [AbstractAutowireCapableBeanFactory]
    │       │               │       │   │       ├── CommonAnnotationBeanPostProcessor [CommonAnnotationBeanPostProcessor]
    │       │               │       │   │       │   └── → 处理 @PostConstruct → InitMethod.invoke() [CommonAnnotationBeanPostProcessor]
    │       │               │       │   │       │       └── → 处理 @Resource（如果 postProcessProperties 没处理）
    │       │               │       │   │       ├── ApplicationContextAwareProcessor [ApplicationContextAwareProcessor]
    │       │               │       │   │       │   ├── → EnvironmentAware.setEnvironment() [ApplicationContextAwareProcessor]
    │       │               │       │   │       │   ├── → ResourceLoaderAware.setResourceLoader() [ApplicationContextAwareProcessor]
    │       │               │       │   │       │   ├── → ApplicationEventPublisherAware.setApplicationEventPublisher() [ApplicationContextAwareProcessor]
    │       │               │       │   │       │   └── → ApplicationContextAware.setApplicationContext() [ApplicationContextAwareProcessor]
    │       │               │       │   │       └── ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor [ConfigurationClassPostProcessor:552]
    │       │               │       │   │           └── → EnhancedConfiguration.setBeanFactory() → 给 @Configuration 代理类注入 [ConfigurationClassPostProcessor:565]
    │       │               │       │   │
    │       │               │       │   ├── invokeInitMethods(beanName, bean, mbd) [AbstractAutowireCapableBeanFactory:1865] // L1865
    │       │               │       │   │   ├── InitializingBean.afterPropertiesSet() [AbstractAutowireCapableBeanFactory:1873] // L1873
    │       │               │       │   │   │   └── → 如果 bean 实现了 InitializingBean 接口
    │       │               │       │   │   └── invokeCustomInitMethod(beanName, bean, mbd, initMethodName) [AbstractAutowireCapableBeanFactory:1897] // L1897
    │       │               │       │   │       └── → @Bean(initMethod="xxx") → 反射调用指定方法 [AbstractAutowireCapableBeanFactory:1897]
    │       │               │       │   │           └── → BeanUtils.findMethod() → Method.invoke(bean) [AbstractAutowireCapableBeanFactory]
    │       │               │       │   │
    │       │               │       │   └── applyBeanPostProcessorsAfterInitialization(bean, beanName) [AbstractAutowireCapableBeanFactory:440] // L440 ★ AOP 代理
    │       │               │       │       └── 遍历 BeanPostProcessor → postProcessAfterInitialization() [AbstractAutowireCapableBeanFactory]
    │       │               │       │           └── AbstractAutoProxyCreator.postProcessAfterInitialization() L315 [AbstractAutoProxyCreator:315]
    │       │               │       │               ├── earlyBeanReferences.remove(cacheKey) != bean [AbstractAutoProxyCreator:318]
    │       │               │       │               │   └── → 没有提前代理过 → wrapIfNecessary() L354 [AbstractAutoProxyCreator:354]
    │       │               │       │               │       ├── isInfrastructureClass() → 跳过 [AbstractAutoProxyCreator:392]
    │       │               │       │               │       ├── getAdvicesAndAdvisorsForBean() → 找匹配的 Advisor [AbstractAutoProxyCreator:367]
    │       │               │       │               │       │   └── → 找 @Aspect/@Before/@After/@Around/@AfterReturning/@AfterThrowing [AbstractAutoProxyCreator]
    │       │               │       │               │       │       └── → 匹配 Pointcut（execution/@annotation 等） [AbstractAutoProxyCreator]
    │       │               │       │               │       ├── specificInterceptors != DO_NOT_PROXY → createProxy() L462 [AbstractAutoProxyCreator:370]
    │       │               │       │               │       │   └── buildProxy() L474 [AbstractAutoProxyCreator:474]
    │       │               │       │               │       │       ├── new ProxyFactory() → copyFrom(this) [AbstractAutoProxyCreator:481]
    │       │               │       │               │       │       ├── 判断代理方式 [AbstractAutoProxyCreator:484]
    │       │               │       │               │       │       │   ├── proxyTargetClass=true → CGLIB [AbstractAutoProxyCreator:484]
    │       │               │       │               │       │       │   │   └── 但目标是接口/lambda → 提取接口用 JDK [AbstractAutoProxyCreator:486]
    │       │               │       │               │       │       │   └── proxyTargetClass=false → evaluateProxyInterfaces() [AbstractAutoProxyCreator:499]
    │       │               │       │               │       │       │       └── 有接口 → JDK；无接口 → CGLIB [AbstractAutoProxyCreator:499]
    │       │               │       │               │       │       ├── buildAdvisors() → 组装 Advisor 链 [AbstractAutoProxyCreator:503]
    │       │               │       │               │       │       │   └── → commonInterceptors + specificInterceptors [AbstractAutoProxyCreator]
    │       │               │       │               │       │       ├── proxyFactory.addAdvisors(advisors) [AbstractAutoProxyCreator:504]
    │       │               │       │               │       │       ├── proxyFactory.setTargetSource(new SingletonTargetSource(bean)) [AbstractAutoProxyCreator:505]
    │       │               │       │               │       │       └── proxyFactory.getProxy(classLoader) [ProxyFactory:96]
    │       │               │       │               │       │           └── createAopProxy() → DefaultAopProxyFactory.createAopProxy() [DefaultAopProxyFactory:60]
    │       │               │       │               │       │               ├── 有接口且非 proxyTargetClass → new JdkDynamicAopProxy(config) [DefaultAopProxyFactory:69]
    │       │               │       │               │       │               │   └── Proxy.newProxyInstance(classLoader, interfaces, this) [JdkDynamicAopProxy:124]
    │       │               │       │               │       │               │       └── → 生成实现了接口的代理类 → this=InvocationHandler [JdkDynamicAopProxy:124]
    │       │               │       │               │       │               │           └── invoke() → getInterceptorsAndDynamicInterceptionAdvice() [JdkDynamicAopProxy:207]
    │       │               │       │               │       │               │               └── → ReflectiveMethodInvocation.proceed() → 责任链 [JdkDynamicAopProxy:223]
    │       │               │       │               │       │               └── 无接口或 proxyTargetClass → new ObjenesisCglibAopProxy(config) [DefaultAopProxyFactory:71]
    │       │               │       │               │       │                   └── buildProxy() → Enhancer.create() [CglibAopProxy:176]
    │       │               │       │               │       │                       └── → 字节码生成子类 → DynamicAdvisedInterceptor.intercept() [CglibAopProxy]
    │       │               │       │               │       │                           └── → CglibMethodInvocation.proceed() → 责任链 [CglibAopProxy]
    │       │               │       │               │       └── 返回代理对象 [AbstractAutoProxyCreator:373]
    │       │               │       │               └── earlyBeanReferences.remove(cacheKey) == bean [AbstractAutoProxyCreator:318]
    │       │               │       │                   └── → 已提前代理过（循环依赖时）→ 跳过！避免重复代理 [AbstractAutoProxyCreator:318]
    │       │               │       │
    │       │               │       ├── ⑥ earlySingletonExposure 检查 [AbstractAutowireCapableBeanFactory:618] // L618-643
    │       │               │       │   └── getSingleton(beanName, false) → 查二级缓存（allowEarlyReference=false） [AbstractAutowireCapableBeanFactory:619]
    │       │               │       │       ├── earlySingletonReference != null（循环依赖发生过） [AbstractAutowireCapableBeanFactory:620]
    │       │               │       │       │   └── exposedObject == bean → exposedObject = earlySingletonReference [AbstractAutowireCapableBeanFactory:622]
    │       │               │       │       │       └── ★ 用早期引用替换 → 保证 B 拿到的 A 和最终暴露的 A 是同一个对象
    │       │               │       │       │   └── exposedObject != bean 且有依赖 Bean → 抛 BeanCurrentlyInCreationException [AbstractAutowireCapableBeanFactory:624]
    │       │               │       │       │       └── // 说明 initializeBean 改变了对象（如被另一个 BPP 代理）
    │       │               │       │       └── earlySingletonReference == null → 没发生循环依赖 → 正常 [AbstractAutowireCapableBeanFactory]
    │       │               │       │
    │       │               │       └── ⑦ registerDisposableBeanIfNecessary(beanName, bean, mbd) [AbstractAutowireCapableBeanFactory:647] // L647
    │       │               │           └── → 如果实现了 DisposableBean/AutoCloseable 或有 destroyMethod → 注册销毁回调 [AbstractAutowireCapableBeanFactory:647]
    │       │               │
    │       │               └── 缓存没有 → createBean 完成 → 回到 getSingleton 的回调 [DefaultSingletonBeanRegistry]
    │       │                   ├── afterSingletonCreation(beanName) → singletonsCurrentlyInCreation.remove(beanName) [DefaultSingletonBeanRegistry:554]
    │       │                   └── addSingleton(beanName, singletonObject) [DefaultSingletonBeanRegistry:160]
    │       │                       ├── singletonObjects.put(beanName, singletonObject) → 一级缓存 [DefaultSingletonBeanRegistry:160]
    │       │                       ├── singletonFactories.remove(beanName) → 清理三级 [DefaultSingletonBeanRegistry:165]
    │       │                       ├── earlySingletonObjects.remove(beanName) → 清理二级 [DefaultSingletonBeanRegistry:186]
    │       │                       └── registeredSingletons.add(beanName) [DefaultSingletonBeanRegistry:187]
    │       │
    │       └── finishRefresh() [AbstractApplicationContext]
    │           ├── clearResourceCaches() [AbstractApplicationContext]
    │           ├── LifecycleProcessor.onRefresh() → 遍历 SmartLifecycle Bean → start() [AbstractApplicationContext]
    │           │   └── WebServerStartStopLifecycle.start() [WebServerStartStopLifecycle:43] ★ 真正启动 Tomcat
    │           │       ├── webServer.start() [TomcatWebServer:226]
    │           │       │   ├── → addPreviouslyRemovedConnectors → 加回 Connector [TomcatWebServer:301]
    │           │       │   │   ★ Connector 重新加入 Service → 触发 init + start
    │           │       │   │   → ProtocolHandler.start → NioEndpoint.start
    │           │       │   │   → ServerSocketChannel.open() → bind(8080) ★ 端口绑定！
    │           │       │   │   → 创建 Worker 线程池（maxThreads=200）+ Poller 线程（2个）
    │           │       │   ├── → performDeferredLoadOnStartup → 补加载 Servlet [TomcatWebServer:326]
    │           │       │   ├── → checkThatConnectorsHaveStarted → 检查状态 [TomcatWebServer:263]
    │           │       │   ├── → this.started = true [TomcatWebServer:239]
    │           │       │   └── → "Tomcat started on port 8080 (http) with context path '/'" [TomcatWebServer:240]
    │           │       └── publishEvent(new ServletWebServerInitializedEvent) [WebServerStartStopLifecycle:46] ⚡
    │           ├── registerApplicationContextAsSingleton() → context 注册到 BeanFactory [AbstractApplicationContext]
    │           ├── publishEvent(new ContextRefreshedEvent(this)) [AbstractApplicationContext] ⚡
    │           │   └── applicationEventMulticaster.multicastEvent(event) [SimpleApplicationEventMulticaster]
    │           │       └── → 遍历 ApplicationListener → onApplicationEvent() [SimpleApplicationEventMulticaster]
    │           │           └── → 如：KeepAlive listener → 启动非守护线程 [SpringApplication:1714]
    │           └── 注册 LiveBeansView（如果启用） [AbstractApplicationContext]
    │
    ├── afterRefresh(context, applicationArguments) [SpringApplication:322] // L322 → 空实现
    │
    ├── startup.started() [SpringApplication:323] // L323
    │   └── Duration.ofMillis(System.currentTimeMillis() - startTime) [SpringApplication:1756]
    │
    ├── logStarted() [SpringApplication:324] // L324-326
    │   └── StartupInfoLogger.logStarted(applicationLog, startup) [SpringApplication:325]
    │       └── getStartedMessage(startup) [StartupInfoLogger:85]
    │           ├── startup.action() → "Started" / "Restored" [StartupInfoLogger:87]
    │           ├── appendApplicationName → "MyApplication" [StartupInfoLogger:104]
    │           ├── " in " + (timeTakenToStarted.toMillis() / 1000.0) + " seconds" [StartupInfoLogger:90]
    │           └── processUptime != null → " (process running for X.XXX)" [StartupInfoLogger:92]
    │
    ├── listeners.started(context, timeTakenToStarted) [SpringApplication:327] // L327
    │   └── EventPublishingRunListener.started() [EventPublishingRunListener:103] // L103-106
    │       ├── context.publishEvent(new ApplicationStartedEvent(application, args, context, timeTaken)) [EventPublishingRunListener:104] ⚡
    │       │   └── applicationEventMulticaster.multicastEvent(event) [SimpleApplicationEventMulticaster]
    │       │       └── → 此时用 context 的 multicaster（非 initialMulticaster）
    │       └── AvailabilityChangeEvent.publish(context, LivenessState.CORRECT) [EventPublishingRunListener:105] ⚡
    │           └── context.publishEvent(new AvailabilityChangeEvent(context, LivenessState.CORRECT)) [EventPublishingRunListener:105]
    │               └── → 标记应用存活状态为 CORRECT（健康检查用）
    │
    ├── callRunners(context, applicationArguments) [SpringApplication:328] // L328
    │   ├── beanFactory.getBeanNamesForType(Runner.class) [SpringApplication:769]
    │   │   └── → 返回所有 ApplicationRunner + CommandLineRunner 的 beanName [SpringApplication:769]
    │   ├── 遍历 beanNames → beanFactory.getBean(beanName, Runner.class) → 放入 IdentityHashMap [SpringApplication:772]
    │   ├── getOrderComparator(beanFactory) [SpringApplication:774]
    │   │   ├── DefaultListableBeanFactory.getDependencyComparator() [DefaultListableBeanFactory]
    │   │   │   └── 是 OrderComparator → 用它；否则 → AnnotationAwareOrderComparator.INSTANCE [SpringApplication:782]
    │   │   └── withSourceProvider(new FactoryAwareOrderSourceProvider) [SpringApplication:775]
    │   │       └── getOrderSource(beanName, instanceType) → 从 BeanDefinition 获取工厂方法和目标类型 [SpringApplication:1860]
    │   │           └── → 用于解析 @Order 注解（可能在工厂方法上） [SpringApplication:1864]
    │   ├── instancesToBeanNames.keySet().stream().sorted(comparator) [SpringApplication:776]
    │   │   └── → 按 @Order/Ordered 排序，ApplicationRunner 和 CommandLineRunner 混排 [SpringApplication:776]
    │   └── 遍历排序后的 Runner → callRunner(runner, args) [SpringApplication:776]
    │       ├── runner instanceof ApplicationRunner → applicationRunner.run(args) [SpringApplication:788]
    │       │   └── args 是 ApplicationArguments 对象 [SpringApplication:788]
    │       │       └── → 可通过 args.getOptionValues("--key") 获取命令行参数 [ApplicationArguments]
    │       └── runner instanceof CommandLineRunner → commandLineRunner.run(args.getSourceArgs()) [SpringApplication:791]
    │           └── args 是原始 String[] 数组 [SpringApplication:791]
    │
    ├── context.isRunning() → listeners.ready(context, startup.ready()) [SpringApplication:335] // L333-336
    │   └── EventPublishingRunListener.ready() [EventPublishingRunListener:109] // L109-112
    │       ├── context.publishEvent(new ApplicationReadyEvent(application, args, context, timeTaken)) [EventPublishingRunListener:110] ⚡
    │       │   └── applicationEventMulticaster.multicastEvent(event) [SimpleApplicationEventMulticaster]
    │       │       └── → 触发所有 ApplicationListener<ApplicationReadyEvent>
    │       └── AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC) [EventPublishingRunListener:111] ⚡
    │           └── → 标记应用就绪，可以接收流量（K8s readiness probe 用） [EventPublishingRunListener:111]
    │
    ├── return context [SpringApplication:341] // L341
    │
    └── catch: handleRunFailure(context, ex, listeners) [SpringApplication:331] // L331 / L803-828
        ├── AbandonedRunException → 直接返回 [SpringApplication:805]
        ├── handleExitCode(context, exception) [SpringApplication:881] // L881
        │   ├── getExitCodeFromException(context, exception) [SpringApplication:894]
        │   │   ├── getExitCodeFromMappedException() → ExitCodeExceptionMapper Bean [SpringApplication:902]
        │   │   │   └── ExitCodeGenerators.addAll(exception, beans) → 匹配异常类型 → 返回 exitCode [ExitCodeGenerators]
        │   │   └── getExitCodeFromExitCodeGeneratorException() → ExitCodeGenerator 接口 [SpringApplication:912]
        │   │       └── exception instanceof ExitCodeGenerator → generator.getExitCode() [SpringApplication:916]
        │   │           └── 递归 getCause() → 遍历异常链 [SpringApplication:919]
        │   ├── exitCode != 0 → context.publishEvent(new ExitCodeEvent(context, exitCode)) [SpringApplication:885] ⚡
        │   └── SpringBootExceptionHandler.registerExitCode(exitCode) [SpringApplication:889]
        ├── listeners.failed(context, exception) [SpringApplication]
        │   └── EventPublishingRunListener.failed() [EventPublishingRunListener:115] // L115-133
        │       ├── new ApplicationFailedEvent(application, args, context, exception) [EventPublishingRunListener:116]
        │       ├── context != null && context.isActive() → context.publishEvent(event) [EventPublishingRunListener:117] ⚡
        │       │   └── → 用 context 的 multicaster 广播 [EventPublishingRunListener:120]
        │       └── context 不活跃 → 遍历 context.getApplicationListeners() → 加到 initialMulticaster [EventPublishingRunListener:125]
        │           └── initialMulticaster.setErrorHandler(new LoggingErrorHandler()) [EventPublishingRunListener:130]
        │               └── initialMulticaster.multicastEvent(event) [EventPublishingRunListener:137] ⚡
        ├── reportFailure(getExceptionReporters(context), exception) [SpringApplication:816] // L842
        │   ├── getExceptionReporters(context) [SpringApplication:830]
        │   │   └── getSpringFactoriesInstances(SpringBootExceptionReporter.class, argumentResolver) [SpringApplication:835]
        │   │       └── → 默认加载 FailureAnalyzers [SpringApplication]
        │   └── 遍历 exceptionReporters → reporter.reportException(failure) [SpringApplication:844]
        │       └── → FailureAnalyzers → 匹配 FailureAnalyzer → analyze() → 输出诊断报告 [FailureAnalyzers]
        │           └── → registerLoggedException(failure) → 抑制重复堆栈打印 [SpringApplication:846]
        ├── context != null → context.close() [SpringApplication:818]
        │   └── → publishEvent(new ContextClosedEvent(context)) [AbstractApplicationContext] ⚡
        │       └── → ApplicationContextClosedListener → contexts.remove → closedContexts.add [SpringApplicationShutdownHook:242]
        │       └── → destroyBeans() → 销毁所有 DisposableBean [AbstractApplicationContext]
        │           └── → 逆序调用 Bean 的 destroy() / @PreDestroy / close() [AbstractApplicationContext]
        │       └── → closeBeanFactory() → 关闭 BeanFactory [AbstractApplicationContext]
        └── shutdownHook.deregisterFailedApplicationContext(context) [SpringApplication:819]
            └── → contexts.remove(applicationContext) → 从 ShutdownHook 注销 [SpringApplicationShutdownHook:97]

## 事件发布时间线

> **【阶段】**

| 序号 | 事件 | 触发时机 | 广播器 |
|---|---|---|---|
| 1 | ApplicationStartingEvent | run() 刚开始，Environment 还没创建 | initialMulticaster |
| 2 | ApplicationEnvironmentPreparedEvent | Environment 准备完成，Context 还没创建 | initialMulticaster |
| 3 | ApplicationContextInitializedEvent | Context 创建完成，initializers 执行完 | initialMulticaster |
| 4 | BootstrapContextClosedEvent | BootstrapContext 关闭，实例迁移到 Context | bootstrapContext.events |
| 5 | ApplicationPreparedEvent | Sources 加载完成，listener 注册到 Context | initialMulticaster（最后一次） |
| 6 | ContextRefreshedEvent | context.refresh() 完成（Spring Framework） | context multicaster |
| 7 | ApplicationStartedEvent | Context 刷新完成，Runner 还没执行 | context multicaster |
| 8 | AvailabilityChangeEvent(LivenessState.CORRECT) | 紧随 ApplicationStartedEvent | context multicaster |
| 9 | ApplicationReadyEvent | Runner 执行完，应用就绪 | context multicaster |
| 10 | AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC) | 紧随 ApplicationReadyEvent | context multicaster |
| - | ApplicationFailedEvent | 启动失败时（异常处理） | context multicaster 或 initialMulticaster |

## 分析的源码文件清单

> **【阶段】**

| 文件 | 行数 | 职责 |
|---|---|---|
| SpringApplication.java | 1876 | 核心启动类，run() 主流程 |
| ApplicationProperties.java | 175 | spring.main.* 属性绑定目标 |
| WebApplicationType.java | 116 | Web 应用类型推断（SERVLET/REACTIVE/NONE） |
| ApplicationContextFactory.java | 103 | ApplicationContext 创建策略接口 |
| DefaultApplicationContextFactory.java | 87 | 默认工厂，创建 AnnotationConfigApplicationContext |
| SpringApplicationRunListener.java | 111 | RunListener 接口，定义 7 个回调 |
| SpringApplicationRunListeners.java | 130 | RunListener 聚合器，遍历调用 |
| EventPublishingRunListener.java | 155 | 核心 RunListener，将回调转为 ApplicationEvent |
| DefaultBootstrapContext.java | 143 | 引导上下文，Context 创建前的轻量容器 |
| BeanDefinitionLoader.java | 336 | Bean 定义加载器，支持 Class/XML/Package/CharSequence |
| SpringApplicationBannerPrinter.java | 140 | Banner 打印器，找 banner.txt 或用默认 |
| SpringBootBanner.java | 59 | 默认 ASCII Spring Logo |
| DefaultApplicationArguments.java | 93 | 命令行参数包装器 |
| EnvironmentConverter.java | 132 | Environment 类型转换器 |
| ApplicationEnvironment.java | 48 | 非 Web 环境，重写 PropertyResolver |
| ApplicationInfoPropertySource.java | 86 | 提供 PID 和应用版本号 |
| StartupInfoLogger.java | 166 | 启动信息日志输出 |
| SpringApplicationShutdownHook.java | 261 | JVM 关闭钩子，优雅关闭 Context |
| Spring Framework 源码（6.2.18，从 m2 sources jar 提取） |
| AbstractAutowireCapableBeanFactory.java | 2073 | Bean 创建核心：doCreateBean / populateBean / initializeBean / getEarlyBeanReference |
| DefaultSingletonBeanRegistry.java | 841 | 三级缓存：singletonObjects / earlySingletonObjects / singletonFactories |
| AbstractBeanFactory.java | 2146 | BeanFactory 基类：getBean → doGetBean → createBean |
| AbstractAutoProxyCreator.java | 640 | AOP 自动代理创建器：getEarlyBeanReference / postProcessAfterInitialization / wrapIfNecessary / createProxy |
| DefaultAopProxyFactory.java | 78 | 代理工厂：选择 JDK 代理还是 CGLIB 代理 |
| JdkDynamicAopProxy.java | 360 | JDK 动态代理：Proxy.newProxyInstance + invoke() 拦截器链 |
| CglibAopProxy.java | 919 | CGLIB 代理：Enhancer 字节码生成子类 + DynamicAdvisedInterceptor |
| ConfigurationClassPostProcessor.java | 827 | @Configuration 类 CGLIB 增强：enhanceConfigurationClasses |

↑ 回到顶部