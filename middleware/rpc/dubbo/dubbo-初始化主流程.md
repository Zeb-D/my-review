本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

公司目前所在的RPC技术栈分两种，异构语言使用`GRPC`，java就使用Dubbo，

对于grpc一些流程相对于dubbo更为简单，有兴趣可以通过[Protobuf详解.md](../Protobuf详解.md)里面会有PB协议讲解，以及执行过程；

dubbo目前的复杂的点有启动时，包括元数据注册与拉取、代理生成等；运行时包括组装处理链及执行；

最近看到了`dubbo 3.0 `较之前优化了好多，虽然当前项目还是`2.7.x`;



### 源码解析

Dubbo 3.0 通过 DubboBootstrap 进行初始化逻辑  , DubboBootstrap  的启动逻辑如下 :

```markdown
// 核心原理为 ApplicationContextEvent 事件的触发 , 当 SpringApplication 启动后 ,会发布该事件

C- AbstractApplicationContext # refresh : 发布 ApplicationContextEvent 事件
C- DubboBootstrapApplicationListener # onApplicationContextEvent : 监听Application 事件
C- DubboBootstrapApplicationListener # onContextRefreshedEvent
C- DubboBootstrap # start : 开始加载操作
```

> 主要涉及类

```markdown
// 参考类 : 
DubboBeanUtils
ServiceAnnotationPostProcessor
DubboClassPathBeanDefinitionScanner
    
    
// 监听器 : 
DubboBootstrapApplicationListener
```



#### 入口详解

Dubbo 的初始化主要经过以下几个类 : DubboBootstrap -

```markdown
// Step 1 : DubboBootstrapApplicationListener
public void onApplicationContextEvent(ApplicationContextEvent event) {
	if (DubboBootstrapStartStopListenerSpringAdapter.applicationContext == null) {
		DubboBootstrapStartStopListenerSpringAdapter.applicationContext = event.getApplicationContext();
	}
        // Step 2-1 : 容器启动后初始化 Bootstrap , 看名称这里应该还可以刷新
	if (event instanceof ContextRefreshedEvent) {
		onContextRefreshedEvent((ContextRefreshedEvent) event);
        // 容器关闭逻辑  
	} else if (event instanceof ContextClosedEvent) {
		onContextClosedEvent((ContextClosedEvent) event);
	}
}

// Step 2-1 
private void onContextRefreshedEvent(ContextRefreshedEvent event) {
	if (dubboBootstrap.getTakeoverMode() == BootstrapTakeoverMode.SPRING) {
		dubboBootstrap.start();
	}
}

// Step 2-2 
private void onContextClosedEvent(ContextClosedEvent event) {
	if (dubboBootstrap.getTakeoverMode() == BootstrapTakeoverMode.SPRING) {
		DubboShutdownHook.getDubboShutdownHook().run();
	}
}
```



#### DubboBootstrap 启动流程

##### 3.1  Start 主流程

start 环节主要由以下几个环节 :

```markdown
Step 1 : 设置启动参数
Step 2 : initialize 初始化
Step 3 : exportServices 输出 Services
Step 4 : referServices 注册 refer Service
Step 5 : onStart() 启动 (异步情况下会创建 Thread 后启动)
    
// 启动 Dubbo 容器环境
dubboBootstrap.start(); 

// Step 1 ：Start 逻辑
public DubboBootstrap start() {
    if (started.compareAndSet(false, true)) {
        startup.set(false);
        initialized.set(false);
        shutdown.set(false);
        awaited.set(false);

        initialize();
        // 1. export Dubbo Services
        exportServices();

        // Not only provider register
        if (!isOnlyRegisterProvider() || hasExportedServices()) {
            // 2. export MetadataService
            exportMetadataService();
            // 3. Register the local ServiceInstance if required
            registerServiceInstance();
        }

        referServices();
        if (asyncExportingFutures.size() > 0 || asyncReferringFutures.size() > 0) {
            new Thread(() -> {
                try {
                    this.awaitFinish();
                } catch (Exception e) {
                    logger.warn(NAME + " asynchronous export / refer occurred an exception.");
                }
                startup.set(true);
                onStart();
            }).start();
        } else {
            startup.set(true);
            onStart();
        }

    }
    return this;
}
```

##### 3.2 设置启动参数

```markdown
if (started.compareAndSet(false, true)) 
    
startup.set(false);
initialized.set(false);
shutdown.set(false);
awaited.set(false);
    
// 此处有几个主要的操作 : 
1. 原子设置已经启动
2. 设置属性
    
// 主要看一下这4个属性
private AtomicBoolean initialized = new AtomicBoolean(false);
private AtomicBoolean started = new AtomicBoolean(false);
private AtomicBoolean startup = new AtomicBoolean(true);
private AtomicBoolean destroyed = new AtomicBoolean(false);
private AtomicBoolean shutdown = new AtomicBoolean(false)
```

##### 3.3  initialize 初始化内容

```markdown
public void initialize() {
    if (!initialized.compareAndSet(false, true)) {
        return;
    }

    ApplicationModel.initFrameworkExts();

    // 初始化配置
    startConfigCenter();
    loadConfigsFromProps();
    checkGlobalConfigs();

    // 初始化 Service
    startMetadataCenter();
    initMetadataService();
}
```

##### 3.3.1 ApplicationModel.initFrameworkExts()

```markdown
// C- DubboBootstrap
public static void initFrameworkExts() {
        Set<FrameworkExt> exts = ExtensionLoader.getExtensionLoader(FrameworkExt.class).getSupportedExtensionInstances();
    	// PIC21005
        for (FrameworkExt ext : exts) {
            ext.initialize();
        }
}


// Step 1 : 通过 SPI 进行插件处理
@SPI
public interface FrameworkExt extends Lifecycle {

}

// 补充 : Dubbo 中的  Lifecycle 对象
- ConfigManager
- Environment
- LifecycleAdapter
- ServiceRepository
- SpringExtensionFactory


// Step 2-1  : ConfigManager 部分
public void initialize() throws IllegalStateException {
        String configModeStr = null;
        try {
            configModeStr = (String) ApplicationModel.getEnvironment().getConfiguration().getProperty(DUBBO_CONFIG_MODE);
            if (StringUtils.hasText(configModeStr)) {
                this.configMode = ConfigMode.valueOf(configModeStr.toUpperCase());
            }
        } catch (Exception e) {
            throw new IllegalArgumentException(msg, e);
        }
}    

// Step 2-2  : Environment 部分
public void initialize() throws IllegalStateException {
        if (initialized.compareAndSet(false, true)) {
            this.propertiesConfiguration = new PropertiesConfiguration();
            this.systemConfiguration = new SystemConfiguration();
            this.environmentConfiguration = new EnvironmentConfiguration();
            this.externalConfiguration = new InmemoryConfiguration("ExternalConfig");
            this.appExternalConfiguration = new InmemoryConfiguration("AppExternalConfig");
            this.appConfiguration = new InmemoryConfiguration("AppConfig");

            loadMigrationRule();
        }
}


// Step 2-3 : ServiceRepository 部分
```

**PIC21005 : FrameworkExt  对象列表**

[![Dubbo 3.0 : BootStrap初始化主流程  第1张](https://www.shouxicto.com/zb_users/upload/2021/10/20211002132222163315214230722.png)](https://www.shouxicto.com/zb_users/upload/2021/10/20211002132222163315214230722.png)

##### 3.3.2 startConfigCenter 简述

这个阶段总共可以分为3步 :

1. 分别加载  ApplicationConfig 和 ConfigCenterConfig
2. 对 configCenterConfig 进行配置
3. 对 Application , Monitor , Modules 等多个组件进行最后的刷新

```markdown
private void startConfigCenter() {

    // Step 1 : 处理对应的 Config , 此处简单一点说分为2个类型 , 很好的思路 , 后面深入了解一下
    
    // > 类型一 : 存在class 对应配置
    // 1. 通过 class 类型去 newInstance 实例化对应类型的 Class 对象
    // 2. 调用对应的 refush 进行刷新处理 (TODO : 后续分析 , 就是配置的重写流程)
    // 3. 缓存到 configManager 中
    
    // > 类型二 : 不存在对应的配置
    // 1. 直接从 ApplicationModel 中获取 Environment
    // 2. newInstance 实例化后放在 configManager 中
    loadConfigs(ApplicationConfig.class);
    loadConfigs(ConfigCenterConfig.class);
 
 
    useRegistryAsConfigCenterIfNecessary();

    // check Config Center -> PS:332001
    Collection<ConfigCenterConfig> configCenters = configManager.getConfigCenters();
    if (CollectionUtils.isEmpty(configCenters)) {
        // 通常配置了注册中心这里都会有值 , 如果为空 , 此处 new 了一个基础实现
    } else {
        for (ConfigCenterConfig configCenterConfig : configCenters) {
            // Step 2 : 对 configCenterConfig 进行配置 , 后续深入
            configCenterConfig.refresh();
            ConfigValidationUtils.validateConfigCenterConfig(configCenterConfig);
        }
    }

    if (CollectionUtils.isNotEmpty(configCenters)) {
        CompositeDynamicConfiguration compositeDynamicConfiguration = new CompositeDynamicConfiguration();
        
        // 这里要对 environment 环节进行配置
        for (ConfigCenterConfig configCenter : configCenters) {
            // Pass config from ConfigCenterBean to environment
            environment.updateExternalConfigMap(configCenter.getExternalConfiguration());
            environment.updateAppExternalConfigMap(configCenter.getAppExternalConfiguration());

            // Fetch config from remote config center
            compositeDynamicConfiguration.addConfiguration(prepareEnvironment(configCenter));
        }
        environment.setDynamicConfiguration(compositeDynamicConfiguration);
    }
    
    // Step 3 : 此处对 Application , Monitor , Modules 等多个组件进行最后的刷新 TODO
    configManager.refreshAll();
}
```

> 补充 : ConfigCenterConfig 对象的作用

该对象中包含连接的基础配置信息 , 例如 protocol , address , port , cluster , namespace 等

**灵活利用该 ConfigCenterConfig 对象可以集成多种不同的注册中心**

> 补充 : PS:332001 结构

[![Dubbo 3.0 : BootStrap初始化主流程  第2张](https://www.shouxicto.com/zb_users/upload/2021/10/20211002132222163315214214046.png)](https://www.shouxicto.com/zb_users/upload/2021/10/20211002132222163315214214046.png)

##### 3.3.3 loadConfigsFromProps 简述

**这里和上面其实差不多 , loadConfigs 是通过特殊的前缀对配置文件的配置进行读取**

之前看过多个开源项目对于配置文件的加载思路 , **基本上殊途同归 , 都是以类区分 , 通过前缀加载** , 后面会单独分析

```markdown
private void loadConfigsFromProps() {

    // application config has load before starting config center
    // load dubbo.applications.xxx
    loadConfigs(ApplicationConfig.class);

    // load dubbo.modules.xxx
    loadConfigs(ModuleConfig.class);

    // load dubbo.monitors.xxx
    loadConfigs(MonitorConfig.class);

    // load dubbo.metricses.xxx
    loadConfigs(MetricsConfig.class);

    // load multiple config types:
    // load dubbo.protocols.xxx
    loadConfigs(ProtocolConfig.class);

    // load dubbo.registries.xxx
    loadConfigs(RegistryConfig.class);

    // load dubbo.providers.xxx
    loadConfigs(ProviderConfig.class);

    // load dubbo.consumers.xxx
    loadConfigs(ConsumerConfig.class);

    // load dubbo.metadata-report.xxx
    loadConfigs(MetadataReportConfig.class);

    // config centers has bean loaded before starting config center
    //loadConfigs(ConfigCenterConfig.class);

}
```

##### 3.3.4 checkGlobalConfigs 简述

在这个环节中会通过 ConfigValidationUtils  进行配置校验 , 以及端口处理 , 最后通过 refresh 刷新

```markdown
private void checkGlobalConfigs() {
    // check config types (ignore metadata-center)
    List<Class<? extends AbstractConfig>> multipleConfigTypes = Arrays.asList(
        ApplicationConfig.class,
        ProtocolConfig.class,
        RegistryConfig.class,
        MetadataReportConfig.class,
        ProviderConfig.class,
        ConsumerConfig.class,
        MonitorConfig.class,
        ModuleConfig.class,
        MetricsConfig.class,
        SslConfig.class);

    for (Class<? extends AbstractConfig> configType : multipleConfigTypes) {
        // 通过 ConfigValidationUtils 对配置进行校验
        checkDefaultAndValidateConfigs(configType);
    }

    // check port conflicts
    // 
    Map<Integer, ProtocolConfig> protocolPortMap = new LinkedHashMap<>();
    for (ProtocolConfig protocol : configManager.getProtocols()) {
        Integer port = protocol.getPort();
        if (port == null || port == -1) {
            continue;
        }
        
        // 此处如果存在端口冲突 , 这里会直接抛出异常
        ProtocolConfig prevProtocol = protocolPortMap.get(port);
        if (prevProtocol != null) {
            throw new IllegalStateException
        }
        protocolPortMap.put(port, protocol);
    }

    // check reference and service
    // 这里正式开始处理 reference 和 DubboService , TODO : 后面单章分析
    for (ReferenceConfigBase<?> reference : configManager.getReferences()) {
        reference.refresh();
    }
    for (ServiceConfigBase service : configManager.getServices()) {
        service.refresh();
    }
}
```

##### 3.3.5 startMetadataCenter 简述

此处是元数据的处理　, 元数据是新特性 , 可以在配置文件中使用前缀“dubbo.metadata-report”设置MetadataReportConfig的属性

> 什么是元数据 :

- 元数据是为了减轻注册中心的压力，将部分存储在注册中心的内容放到元数据中心。
- 元数据中心的数据只是给自己使用的 , 改动无需刷新
- 不需要监听和通知 , 极大的减少了性能的消耗

```markdown
// 参考类 : 
DubboBeanUtils
ServiceAnnotationPostProcessor
DubboClassPathBeanDefinitionScanner
    
    
// 监听器 : 
DubboBootstrapApplicationListener0
```

##### 3.3.6 initMetadataService 简述

```markdown
// 参考类 : 
DubboBeanUtils
ServiceAnnotationPostProcessor
DubboClassPathBeanDefinitionScanner
    
    
// 监听器 : 
DubboBootstrapApplicationListener1
```

##### 3.4  exportServices

详见服务注册体系

##### 3.5  referServices

详见服务发现体系

##### 3.6 onStart 调用监听器扩展

```markdown
// 参考类 : 
DubboBeanUtils
ServiceAnnotationPostProcessor
DubboClassPathBeanDefinitionScanner
    
    
// 监听器 : 
DubboBootstrapApplicationListener2
```

### 总结

这一篇主要是为了总结,  以及对全部的概念有个初步的理解 , 后面会逐步完善整个流程中的细节点 , 以及对其代码的学习和使用。

