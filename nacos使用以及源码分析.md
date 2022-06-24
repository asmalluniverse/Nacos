### 1.官网地址
https://nacos.io/zh-cn/docs/what-is-nacos.html

### 2.下载地址
https://github.com/alibaba/nacos/releases/tag/1.4.3

### 3. 概览
Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

**Nacos提供了服务发现和配置功能的功能**，下面基于1.4.3的版本介绍Nacos的基本使用

### 3.安装使用
##### 3.1 单节点部署
首先我们下载安装包，然后解压，下载地址：https://github.com/alibaba/nacos/releases/tag/1.4.3

- 配置修改

1. 打开conf目录，修改application.properties配置文件：
```yml
#数据库配置信息使用的是 nacos库，需要自行创建
db.url.0=db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
#数据库用户名
db.user.0=root
#数据库密码
db.password.0=123456
```

2. 表创建
在数据库中执行sql文件，执行conf目录下的nacos-mysql.sql

3. 启动服务
到bin目录下执行如下命令 
`startup.cmd -m standalone`

### 4.nacos和spring cloud集成


4. 配置优先级
Nacos的优先级和Spring的优先级相同，带环境名的配置优先

5. 共享配置
在微服务系统里，多组件之间肯定会使用公共的配置，那如果每个应用都配置一份岂不是很冗余，并且如果改动，每个应用都要修改。

为了解决这个问题，Nacos同样贴心的给我们提供了另一种配置方式：共享配置
# 共享的配置
spring.cloud.nacos.config.shared-configs[0].data-id=log.yaml
# refresh: true表示动态刷新配置
spring.cloud.nacos.config.shared-configs[0].refresh=true

6. 扩展配置
扩展配置和共享配置的使用方式相同，增加extension-configs即可
**扩展配置的优先级比共享配置优先级高**

# 扩展的配置
spring.cloud.nacos.config.extension-configs[0].data-id=extension.properties
spring.cloud.nacos.config.extension-configs[0].refresh=true


总结：Nocas常用作配置中心，实现应用的动态刷新，使用起来也比较方便。
配置的优先级：共享配置 < 扩展配置 < 服务名 < 服务名.文件扩展名 < 服务名-环境.文件扩展名

### 1.nacao整体流程分析，我们可以从如下几个方面进行思考
1.客户端启动从服务器拉取配置信息
2.客户端要缓存服务端的信息到本地
3.客户端要监听服务端配置文件的变更
4.Nacos服务端的配置是如何存储的呢
5.@Value注解是怎么实现配置的动态注入的呢？

- nacos 如何加载配置文件？
我们知道，在使用nacos的时候我们只要启动nacos服务端，并配置相关的配置文件，在客户端我们通过@value注解即可获取到，
@Value注解是spring中给我们提供的，那反推我们应该知道肯定是把所有的配置加载到Enviroment中。

那我们先分析一下Environment，Environment继承了PropertyResolver，它相当于属性配置的解决方案，而Environment是个接口，有多种实现。
Environment接口有两个关键的作用：
1. Profile， 
	可能在我们的开发中不同环境需要定义不同的配置文件，我们就可以通过profile来进行指定，例如在配置文件中进行指定：spring.profiles.active=dev
2.properties。
Spring的Environment可以方便的访问property属性，包含系统属性，环境变量和自定义的。
并且是可配置的，可以加入自定义的property，这基于spring使用PropertySources 来管理不同的PropertySource

以上都是spring中的属性源加载，那spring Cloud中属性源的加载是如何实现的呢


### 2.PropertySourceLocator加载原理
在spring boot项目启动时，有一个prepareContext的方法，它会回调所有实现了**ApplicationContextInitializer**的实例，来做一些初始化工作。

1. ApplicationContextInitializer类的作用
ApplicationContextInitializer是Spring框架原有的东西， 它的主要作用就是在,ConfigurableApplicationContext类型(或者子类型)的ApplicationContext做
refresh之前，允许我们对ConfiurableApplicationContext的实例做进一步的设置和处理。它可以用在需要对应用程序上下文进行编程初始化的web应用程序中，比如根据上下文环
境来注册propertySource，或者配置文件。而Config 的这个配置中心的需求恰好需要这样一个机制来完成。

ApplicationContextInitializer的使用有以下3中方式：
自定义实现类：
```java
 /**
  * 自定义的ApplicationContextInitializer
  *
  */
 public class MyInitializer implements ApplicationContextInitializer,Ordered {
    
     public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment ce=applicationContext.getEnvironment();
		for(PropertySource<?> propertySource:ce.getPropertySources()){
			System.out.println(propertySource);
		}
		System.out.println("--------end");
     }
 }
```

- 在启动类中使用SpringApplication.addInitializers() 手动增加Initializer
```java
@SpringBootApplication
@Slf4j
public class WebsocketApplication {

 public static void main(String[] args) {
     run(args);
 }

 private static void run(String[] args) {
     SpringApplication springApplication = new SpringApplication(WebsocketApplication.class);
     springApplication.addInitializers(new MyInitializer());
     springApplication.run(args);
 }
}
```
- application.properties添加配置方式
` context.initializer.classes=com.train.nacos.initializertest.MyInitializer`

- spi的机制实现
将MyInitializer配置在spring.factories文件中
` org.springframework.context.ApplicationContextInitializer=com.train.nacos.initializertest.MyInitializer`


### 2.Spring Cloud是如何实现的呢？
- PropertySourceBootstrapConfiguration
我们可以在`spring-cloud-context-2.2.6.RELEASE.jar`包下面的`spring.factories`文件中找到答案，是通过spi的机制进行的扩展实现
PropertySourceBootstrapConfiguration类就实现了ApplicationContextInitializer接口，并实现了initialize方法
```java 
/**
* 配置类
*/
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {
	
	//通过注解的方式装载所有实现了PropertySourceLocator接口的实现类
	//具体的实现在BootstrapImportSelector这个类具体的加载过程如下：
	/**
	 * 1. spi的机制加载监听器BootstrapApplicationListener（spring.factories文件中）
	 * 2. BootstrapApplicationListener该类实现了ApplicationListener接口，会实现onApplicationEvent方法，代表着在spring容器启动完成会进行该方法的调用
	 * 3. BootstrapImportSelectorConfiguration类中有一个@Import注解，注解中有BootstrapImportSelector类，该类中实现了DeferredImportSelector接口，并通过spi的机制进行加载
	@Autowired(required = false)
	private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();

	@Override
	public int getOrder() {
		return this.order;
	}
	
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		List<PropertySource<?>> composite = new ArrayList<>();
		//对propertySourceLocators数组进行排序，根据默认的AnnotationAwareOrderComparator，根据@Order注解进行排序
		AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
		boolean empty = true;
		//获取运行的环境上下文
		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		for (PropertySourceLocator locator : this.propertySourceLocators) {
		//回调所有实现PropertySourceLocator接口实例的locate方法，并收集到source这个集合中。
		//例如这里会调用NacosPropertySourceLocator 类的locate方法
			Collection<PropertySource<?>> source = locator.locateCollection(environment);
			if (source == null || source.size() == 0) {
				continue;
			}
			//遍历source，把PropertySource包装成BootstrapPropertySource加入到sourceList中。
			List<PropertySource<?>> sourceList = new ArrayList<>();
			for (PropertySource<?> p : source) {
				if (p instanceof EnumerablePropertySource) {
					EnumerablePropertySource<?> enumerable = (EnumerablePropertySource<?>) p;
					sourceList.add(new BootstrapPropertySource<>(enumerable));
				}
				else {
					sourceList.add(new SimpleBootstrapPropertySource(p));
				}
			}
			logger.info("Located property source: " + sourceList);
			composite.addAll(sourceList);
			//表示propertysource不为空
			empty = false;
		}
		//只有propertysource不为空的情况，才会设置到environment中
		if (!empty) {
			//获取当前Environment中的所有PropertySources.
			MutablePropertySources propertySources = environment.getPropertySources();
			String logConfig = environment.resolvePlaceholders("${logging.config:}");
			LogFile logFile = LogFile.get(environment);
			// 遍历移除bootstrapProperty的相关属性
			for (PropertySource<?> p : environment.getPropertySources()) {
				if (p.getName().startsWith(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
					propertySources.remove(p.getName());
				}
			}
			//把前面获取到的PropertySource，插入到Environment中的PropertySources中。
			insertPropertySources(propertySources, composite);
			reinitializeLoggingSystem(environment, logConfig, logFile);
			setLogLevels(applicationContext, environment);
			handleIncludedProfiles(environment);
		}
	}
}
```
上述代码逻辑说明如下。
1. 首先this.propertySourceLocators，表示所有实现了PropertySourceLocators接
口的实现类，其中就包括我们前面自定义的GpJsonPropertySourceLocator。
2. 根据默认的 AnnotationAwareOrderComparator 排序规则对
propertySourceLocators数组进行排序。
3. 获取运行的环境上下文ConfigurableEnvironment
4. 遍历propertySourceLocators时
调用 locate 方法，传入获取的上下文environment
将source添加到PropertySource的链表中
设置source是否为空的标识标量empty
5. source不为空的情况，才会设置到environment中
返回Environment的可变形式，可进行的操作如addFirst、addLast
移除propertySources中的bootstrapProperties
根据config server覆写的规则，设置propertySources
处理多个active profiles的配置信息

### 3. NacosPropertySourceLocator
NacosPropertySourceLocator就实现了PropertySourceLocator了该接口，实现了远程配置的加载，并加载到propertySource中
```java
@Override
	public PropertySource<?> locate(Environment env) {
		//构造方法中设置到nacosConfigProperties以及configService
		nacosConfigProperties.setEnvironment(env);
		ConfigService configService = nacosConfigManager.getConfigService();

		if (null == configService) {
			log.warn("no instance of config service found, can't load config from nacos");
			return null;
		}
		//获取客户端配置的超时时间，默认是3秒
		long timeout = nacosConfigProperties.getTimeout();
		nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
				timeout);
		//获取name属性 //在Spring Cloud中，默认的name=spring.application.name。
		String name = nacosConfigProperties.getName();

		String dataIdPrefix = nacosConfigProperties.getPrefix();
		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = name;
		}
        //如果以上获取都为空，获取spring.application.name的名称作为dataId
		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = env.getProperty("spring.application.name");
		}
		//创建一个Composite属性源，可以包含多个PropertySource
		CompositePropertySource composite = new CompositePropertySource(
				NACOS_PROPERTY_SOURCE_NAME);
		//加载共享配置
		loadSharedConfiguration(composite);
		//加载扩展配置
		loadExtConfiguration(composite);
		//加载自身配置
		loadApplicationConfiguration(composite, dataIdPrefix, nacosConfigProperties, env);
		return composite;
	}
```
1. 获取nacos客户端的配置属性，并生成dataId（这个很重要，要定位nacos的配置）
2. 分别调用三个方法从加载配置属性源，保存到composite组合属性源中

注意：无论是加载加载共享配置、扩展配置本质上都是一样的，都是去远程服务器上读取配置，加载到CompositePropertySource属性源中

### 4.loadApplicationConfiguration
我们分析下应用程序中加载的配置过程
```java
	private void loadApplicationConfiguration(
			CompositePropertySource compositePropertySource, String dataIdPrefix,
			NacosConfigProperties properties, Environment environment) {
		//默认的扩展名为： properties
		String fileExtension = properties.getFileExtension();
		//获取group
		String nacosGroup = properties.getGroup();
		// load directly once by default
		//加载`dataid=项目名称`的配置
		loadNacosDataIfPresent(compositePropertySource, dataIdPrefix, nacosGroup,
				fileExtension, true);
		// load with suffix, which have a higher priority than the default
		//加载`dataid=项目名称+扩展名`的配置
		loadNacosDataIfPresent(compositePropertySource,
				dataIdPrefix + DOT + fileExtension, nacosGroup, fileExtension, true);
		// Loaded with profile, which have a higher priority than the suffix
		// 遍历profile（可以有多个），根据profile加载配置
		for (String profile : environment.getActiveProfiles()) {
			String dataId = dataIdPrefix + SEP1 + profile + DOT + fileExtension;
			loadNacosDataIfPresent(compositePropertySource, dataId, nacosGroup,
					fileExtension, true);
		}

	}
```

- loadNacosDataIfPresent  
调用loadNacosPropertySource加载存在的配置信息。把加载之后的配置属性保存到CompositePropertySource中
```java 
	private void loadNacosDataIfPresent(final CompositePropertySource composite,
			final String dataId, final String group, String fileExtension,
			boolean isRefreshable) {
		//如果dataId为空，或者group为空，则直接跳过
		if (null == dataId || dataId.trim().length() < 1) {
			return;
		}
		if (null == group || group.trim().length() < 1) {
			return;
		}
		//从nacos中获取属性源
		NacosPropertySource propertySource = this.loadNacosPropertySource(dataId, group,
				fileExtension, isRefreshable);
		//把属性源保存到compositePropertySource中
		this.addFirstPropertySource(composite, propertySource, false);
	}
```

- loadNacosPropertySource
```java 
	private NacosPropertySource loadNacosPropertySource(final String dataId,
			final String group, String fileExtension, boolean isRefreshable) {
		if (NacosContextRefresher.getRefreshCount() != 0) {
			//是否支持自动刷新,如果不支持自动刷新配置则自动从缓存获取返回(不从远程服务器加载)
			if (!isRefreshable) {
				return NacosPropertySourceRepository.getNacosPropertySource(dataId,
						group);
			}
		}
		//构造器从配置中心获取数据
		return nacosPropertySourceBuilder.build(dataId, group, fileExtension,
				isRefreshable);
	}
```

- NacosPropertySourceBuilder.build
```java
/**
 * @param dataId Nacos dataId
 * @param group Nacos group
 */
NacosPropertySource build(String dataId, String group, String fileExtension,
		boolean isRefreshable) {
		//调用loadNacosData加载远程数据
	List<PropertySource<?>> propertySources = loadNacosData(dataId, group,
			fileExtension);
	//构造NacosPropertySource(这个是Nacos自定义扩展的PropertySource，和我们前面演示的自定义PropertySource类似)。
	//相当于把从远程服务器获取的数据保存到NacosPropertySource中
	NacosPropertySource nacosPropertySource = new NacosPropertySource(propertySources,
			group, dataId, new Date(), isRefreshable);
	//把属性缓存到本地缓存
	NacosPropertySourceRepository.collectNacosPropertySource(nacosPropertySource);
	return nacosPropertySource;
}
```

- NacosPropertySourceBuilder.loadNacosData
```java
private List<PropertySource<?>> loadNacosData(String dataId, String group,
			String fileExtension) {
		String data = null;
		try {
			////加载Nacos配置数据 *****
			data = configService.getConfig(dataId, group, timeout);
			if (StringUtils.isEmpty(data)) {
				log.warn(
						"Ignore the empty nacos configuration and get it based on dataId[{}] & group[{}]",
						dataId, group);
				return Collections.emptyList();
			}
			if (log.isDebugEnabled()) {
				log.debug(String.format(
						"Loading nacos data, dataId: '%s', group: '%s', data: %s", dataId,
						group, data));
			}
			//对加载的数据进行解析，保存到List<PropertySource>集合。
			return NacosDataParserHandler.getInstance().parseNacosData(dataId, data,
					fileExtension);
		}
		catch (NacosException e) {
			log.error("get data from Nacos error,dataId:{} ", dataId, e);
		}
		catch (Exception e) {
			log.error("parse data from Nacos error,dataId:{},data:{}", dataId, data, e);
		}
		return Collections.emptyList();
	}
```
通过上述分析，我们知道了Spring Cloud集成Nacos时的整个调用流程，并且知道在启动时，Spring Cloud会从Nacos Server中加载动态数据保存到Environment集合。从而实现动态配置的自动注入。


### 5.Nacos客户端的数据的加载流程

-  NacosConfigService.getConfig
优先从本地缓存中加载（远程的配置会缓存到本地），如果本地没有那么进行远程获取，调用Nacos服务端提供的sdk接口进行获取
```java
@Override
public String getConfig(String dataId, String group, long timeoutMs) throws NacosException {
	return getConfigInner(namespace, dataId, group, timeoutMs);
}


private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
	//获取组名
	group = blank2defaultGroup(group);
	//参数校验
	ParamUtils.checkKeyParam(dataId, group);
	//设置响应结果
	ConfigResponse cr = new ConfigResponse();
	
	cr.setDataId(dataId);
	cr.setTenant(tenant);
	cr.setGroup(group);
	
	// 优先使用本地配置，这里本地调用就是文件流的形式加载文件，返回对应的文件内容
	String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
	if (content != null) {
		LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
				dataId, group, tenant, ContentUtils.truncateContent(content));
		cr.setContent(content);
		//获取容灾配置的encryptedDataKey
		String encryptedDataKey = LocalEncryptedDataKeyProcessor
				.getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
		//保存到cr
		cr.setEncryptedDataKey(encryptedDataKey);
		//执行过滤(目前好像没有实现),应该是用于用户自定义扩展使用
		configFilterChainManager.doFilter(null, cr);
		content = cr.getContent();
		return content;
	}
	//本地配置加载不到进行远程调用
	try {
		ConfigResponse response = worker.getServerConfig(dataId, group, tenant, timeoutMs);
		cr.setContent(response.getContent());
		cr.setEncryptedDataKey(response.getEncryptedDataKey());
		
		configFilterChainManager.doFilter(null, cr);
		content = cr.getContent();
		
		return content;
	} catch (NacosException ioe) {
		if (NacosException.NO_RIGHT == ioe.getErrCode()) {
			throw ioe;
		}
		LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
				agent.getName(), dataId, group, tenant, ioe.toString());
	}
	
	LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
			dataId, group, tenant, ContentUtils.truncateContent(content));
	content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
	cr.setContent(content);
	String encryptedDataKey = LocalEncryptedDataKeyProcessor
			.getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
	cr.setEncryptedDataKey(encryptedDataKey);
	configFilterChainManager.doFilter(null, cr);
	content = cr.getContent();
	return content;
}
```

- LocalConfigInfoProcessor.getFailover
从本地读取配置
```java
public static String getFailover(String serverName, String dataId, String group, String tenant) {
	//获取缓存路径下面的文件
	File localPath = getFailoverFile(serverName, dataId, group, tenant);
	if (!localPath.exists() || !localPath.isFile()) {
		return null;
	}
	
	try {
		//通过文件流的形式读取文件
		return readFile(localPath);
	} catch (IOException ioe) {
		LOGGER.error("[" + serverName + "] get failover error, " + localPath, ioe);
		return null;
	}
}
```

- ClientWorker.getServerConfig
ClientWorker，表示客户端的一个工作类，它负责和服务端交互。
```java
public ConfigResponse getServerConfig(String dataId, String group, String tenant, long readTimeout)
		throws NacosException {
	ConfigResponse configResponse = new ConfigResponse();
	if (StringUtils.isBlank(group)) {
		group = Constants.DEFAULT_GROUP;
	}
	
	HttpRestResult<String> result = null;
	try {
		//构建请求参数
		Map<String, String> params = new HashMap<String, String>(3);
		if (StringUtils.isBlank(tenant)) {
			params.put("dataId", dataId);
			params.put("group", group);
		} else {
			params.put("dataId", dataId);
			params.put("group", group);
			params.put("tenant", tenant);
		}
		//发起远程调用
		result = agent.httpGet(Constants.CONFIG_CONTROLLER_PATH, null, params, agent.getEncode(), readTimeout);
	} catch (Exception ex) {
		String message = String
				.format("[%s] [sub-server] get server config exception, dataId=%s, group=%s, tenant=%s",
						agent.getName(), dataId, group, tenant);
		LOGGER.error(message, ex);
		throw new NacosException(NacosException.SERVER_ERROR, ex);
	}
	//根据响应结果实现不同的处理
	switch (result.getCode()) {
		//如果响应成功，保存快照到本地，并返回响应内容
		case HttpURLConnection.HTTP_OK:
			LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, result.getData());
			configResponse.setContent(result.getData());
			String configType;
			if (result.getHeader().getValue(CONFIG_TYPE) != null) {
				configType = result.getHeader().getValue(CONFIG_TYPE);
			} else {
				configType = ConfigType.TEXT.getType();
			}
			configResponse.setConfigType(configType);
			String encryptedDataKey = result.getHeader().getValue(ENCRYPTED_DATA_KEY);
			LocalEncryptedDataKeyProcessor
					.saveEncryptDataKeySnapshot(agent.getName(), dataId, group, tenant, encryptedDataKey);
			configResponse.setEncryptedDataKey(encryptedDataKey);
			return configResponse;
			//如果返回404， 清空本地快照
		case HttpURLConnection.HTTP_NOT_FOUND:
			LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, null);
			LocalEncryptedDataKeyProcessor.saveEncryptDataKeySnapshot(agent.getName(), dataId, group, tenant, null);
			return configResponse;
		case HttpURLConnection.HTTP_CONFLICT: {
			LOGGER.error(
					"[{}] [sub-server-error] get server config being modified concurrently, dataId={}, group={}, "
							+ "tenant={}", agent.getName(), dataId, group, tenant);
			throw new NacosException(NacosException.CONFLICT,
					"data being modified, dataId=" + dataId + ",group=" + group + ",tenant=" + tenant);
		}
		case HttpURLConnection.HTTP_FORBIDDEN: {
			LOGGER.error("[{}] [sub-server-error] no right, dataId={}, group={}, tenant={}", agent.getName(),
					dataId, group, tenant);
			throw new NacosException(result.getCode(), result.getMessage());
		}
		default: {
			LOGGER.error("[{}] [sub-server-error]  dataId={}, group={}, tenant={}, code={}", agent.getName(),
					dataId, group, tenant, result.getCode());
			throw new NacosException(result.getCode(),
					"http error, code=" + result.getCode() + ",dataId=" + dataId + ",group=" + group + ",tenant="
							+ tenant);
		}
	}
}
```

- ServerHttpAgent.httpGet
```java
@Override
    public HttpRestResult<String> httpGet(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encode, long readTimeoutMs) throws Exception {
        final long endTime = System.currentTimeMillis() + readTimeoutMs;
		//注入安全信息
        injectSecurityInfo(paramValues);
		//获取当前服务器地址
        String currentServerAddr = serverListMgr.getCurrentServerAddr();
        int maxRetry = this.maxRetry;
		//配置HttpClient的属性，默认的readTimeOut超时时间是3s
        HttpClientConfig httpConfig = HttpClientConfig.builder()
                .setReadTimeOutMillis(Long.valueOf(readTimeoutMs).intValue())
                .setConTimeOutMillis(ConfigHttpClientManager.getInstance().getConnectTimeoutOrDefault(100)).build();
        do {
            try {
				//设置header
                Header newHeaders = getSpasHeaders(paramValues, encode);
                if (headers != null) {
                    newHeaders.addAll(headers);
                }
				//构建query查询条件
                Query query = Query.newInstance().initParams(paramValues);
                HttpRestResult<String> result = NACOS_RESTTEMPLATE
                        .get(getUrl(currentServerAddr, path), httpConfig, newHeaders, query, String.class);
                if (isFail(result)) {
                    LOGGER.error("[NACOS ConnectException] currentServerAddr: {}, httpCode: {}",
                            serverListMgr.getCurrentServerAddr(), result.getCode());
                } else {
                    // Update the currently available server addr
                    serverListMgr.updateCurrentServerAddr(currentServerAddr);
                    return result;
                }
            } catch (ConnectException connectException) {
                LOGGER.error("[NACOS ConnectException httpGet] currentServerAddr:{}, err : {}",
                        serverListMgr.getCurrentServerAddr(), connectException.getMessage());
            } catch (SocketTimeoutException socketTimeoutException) {
                LOGGER.error("[NACOS SocketTimeoutException httpGet] currentServerAddr:{}， err : {}",
                        serverListMgr.getCurrentServerAddr(), socketTimeoutException.getMessage());
            } catch (Exception ex) {
                LOGGER.error("[NACOS Exception httpGet] currentServerAddr: " + serverListMgr.getCurrentServerAddr(),
                        ex);
                throw ex;
            }
            //如果服务端列表有多个，并且当前请求失败，则尝试用下一个地址进行重试
            if (serverListMgr.getIterator().hasNext()) {
                currentServerAddr = serverListMgr.getIterator().next();
            } else {
                maxRetry--;
                if (maxRetry < 0) {
                    throw new ConnectException(
                            "[NACOS HTTP-GET] The maximum number of tolerable server reconnection errors has been reached");
                }
                serverListMgr.refreshCurrentServerAddr();
            }
            
        } while (System.currentTimeMillis() <= endTime);
        
        LOGGER.error("no available server");
        throw new ConnectException("no available server");
    }
```

### 6.Nacos Server端的配置获取
客户端向服务端发送请求，加载服务端配置，对应的接口地址：/nacos/v1/cs/configs，这里看Nacos服务端的地址还是比较熟悉的，跟我们平时写的业务代码结构类似。
- ConfigController.getConfig
```java 
@GetMapping
@Secured(action = ActionTypes.READ, signType = SignType.CONFIG)
public void getConfig(HttpServletRequest request, HttpServletResponse response,
		@RequestParam("dataId") String dataId, @RequestParam("group") String group,
		@RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY) String tenant,
		@RequestParam(value = "tag", required = false) String tag)
		throws IOException, ServletException, NacosException {
	// check tenant
	//这里的tenant就是namespaceid，并进行校验
	ParamUtils.checkTenant(tenant);
	tenant = NamespaceUtil.processNamespaceParameter(tenant);
	// check params
	ParamUtils.checkParam(dataId, group, "datumId", "content");
	ParamUtils.checkParam(tag);
	//获取请求的ip
	final String clientIp = RequestUtil.getRemoteIp(request);
	String isNotify = request.getHeader("notify");
	inner.doGetConfig(request, response, dataId, group, tenant, tag, isNotify, clientIp);
}
```

- ConfigServletInner.doGetConfig
```java
    /**
     * Execute to get config API.
     */
    public String doGetConfig(HttpServletRequest request, HttpServletResponse response, String dataId, String group,
            String tenant, String tag, String isNotify, String clientIp) throws IOException, ServletException {
        
        boolean notify = false;
        if (StringUtils.isNotBlank(isNotify)) {
            notify = Boolean.parseBoolean(isNotify);
        }
        //拼接锁，后面需要这个groupKey来获取锁，从而对文件进行操作
        final String groupKey = GroupKey2.getKey(dataId, group, tenant);
        String autoTag = request.getHeader("Vipserver-Tag");
        //获取远程ip地址
        String requestIpApp = RequestUtil.getAppName(request);
        //获取锁
        //lockResult>0 ，表示CacheItem(也就是缓存的配置项)不为空，并且已经加了读锁，意味着这个缓存数据不能被删除。
        //lockResult=0 ,表示cacheItem为空，不需要加读锁
        //lockResult=-1 , 表示加锁失败，存在冲突。（重试十次进行锁的获取）
        int lockResult = tryConfigReadLock(groupKey);
        
        final String requestIp = RequestUtil.getRemoteIp(request);
        boolean isBeta = false;
        boolean isSli = false;
        //获取锁成功，此时其他线程不能对这个文件，即cacheItem进行删除
        if (lockResult > 0) {
            // LockResult > 0 means cacheItem is not null and other thread can`t delete this cacheItem
            FileInputStream fis = null;
            try {
                String md5 = Constants.NULL;
                long lastModified = 0L;
                //从本地缓存中，根据groupKey获取CacheItem
                CacheItem cacheItem = ConfigCacheService.getContentCache(groupKey);
                //判断是否是beta发布，也就是测试版本,web界面有一个按钮进行控制
                if (cacheItem.isBeta() && cacheItem.getIps4Beta().contains(clientIp)) {
                    isBeta = true;
                }
                //获取配置文件的类型 txt,json,xml....
                final String configType =
                        (null != cacheItem.getType()) ? cacheItem.getType() : FileTypeEnum.TEXT.getFileType();
                response.setHeader("Config-Type", configType);
                //返回文件类型的枚举对象
                FileTypeEnum fileTypeEnum = FileTypeEnum.getFileTypeEnumByFileExtensionOrFileType(configType);
                String contentTypeHeader = fileTypeEnum.getContentType();
                response.setHeader(HttpHeaderConsts.CONTENT_TYPE, contentTypeHeader);
                
                File file = null;
                ConfigInfoBase configInfoBase = null;
                PrintWriter out;
                if (isBeta) {
                    md5 = cacheItem.getMd54Beta();
                    lastModified = cacheItem.getLastModifiedTs4Beta();
                    if (PropertyUtil.isDirectRead()) {
                        //如果是standalone模式 从数据库中读取
                        configInfoBase = persistService.findConfigInfo4Beta(dataId, group, tenant);
                    } else {
                        //如果是集群模式，从磁盘中获取文件，得到的是一个完整的File，因为集群模式中每个节点都会缓存一份文件在本地
                        file = DiskUtil.targetBetaFile(dataId, group, tenant);
                    }
                    response.setHeader("isBeta", "true");
                } else {
                    //是否设置了标签，可以再web界面进行设置
                    if (StringUtils.isBlank(tag)) {
                        if (isUseTag(cacheItem, autoTag)) {
                            if (cacheItem.tagMd5 != null) {
                                md5 = cacheItem.tagMd5.get(autoTag);
                            }
                            if (cacheItem.tagLastModifiedTs != null) {
                                lastModified = cacheItem.tagLastModifiedTs.get(autoTag);
                            }
                            if (PropertyUtil.isDirectRead()) {
                                configInfoBase = persistService.findConfigInfo4Tag(dataId, group, tenant, autoTag);
                            } else {
                                file = DiskUtil.targetTagFile(dataId, group, tenant, autoTag);
                            }
                            
                            response.setHeader(com.alibaba.nacos.api.common.Constants.VIPSERVER_TAG,
                                    URLEncoder.encode(autoTag, StandardCharsets.UTF_8.displayName()));
                        } else {
                            md5 = cacheItem.getMd5();
                            lastModified = cacheItem.getLastModifiedTs();
                            if (PropertyUtil.isDirectRead()) {
                                configInfoBase = persistService.findConfigInfo(dataId, group, tenant);
                            } else {
                                file = DiskUtil.targetFile(dataId, group, tenant);
                            }
                            if (configInfoBase == null && fileNotExist(file)) {
                                // FIXME CacheItem
                                // No longer exists. It is impossible to simply calculate the push delayed. Here, simply record it as - 1.
                                ConfigTraceService.logPullEvent(dataId, group, tenant, requestIpApp, -1,
                                        ConfigTraceService.PULL_EVENT_NOTFOUND, -1, requestIp, notify);
                                
                                // pullLog.info("[client-get] clientIp={}, {},
                                // no data",
                                // new Object[]{clientIp, groupKey});
                                
                                return get404Result(response);
                            }
                            isSli = true;
                        }
                        //如果tag不为空，说明配置文件设置了tag标签
                    } else {
                        if (cacheItem.tagMd5 != null) {
                            md5 = cacheItem.tagMd5.get(tag);
                        }
                        if (cacheItem.tagLastModifiedTs != null) {
                            Long lm = cacheItem.tagLastModifiedTs.get(tag);
                            if (lm != null) {
                                lastModified = lm;
                            }
                        }
                        if (PropertyUtil.isDirectRead()) {
                            configInfoBase = persistService.findConfigInfo4Tag(dataId, group, tenant, tag);
                        } else {
                            file = DiskUtil.targetTagFile(dataId, group, tenant, tag);
                        }
                        if (configInfoBase == null && fileNotExist(file)) {
                            // FIXME CacheItem
                            // No longer exists. It is impossible to simply calculate the push delayed. Here, simply record it as - 1.
                            ConfigTraceService.logPullEvent(dataId, group, tenant, requestIpApp, -1,
                                    ConfigTraceService.PULL_EVENT_NOTFOUND, -1, requestIp, notify && isSli);
                            
                            // pullLog.info("[client-get] clientIp={}, {},
                            // no data",
                            // new Object[]{clientIp, groupKey});
                            
                            response.setStatus(HttpServletResponse.SC_NOT_FOUND);
                            response.getWriter().println("config data not exist");
                            return HttpServletResponse.SC_NOT_FOUND + "";
                        }
                    }
                }
                //把获取的数据结果设置到response中返回
                response.setHeader(Constants.CONTENT_MD5, md5);
                
                // Disable cache.
                response.setHeader("Pragma", "no-cache");
                response.setDateHeader("Expires", 0);
                response.setHeader("Cache-Control", "no-cache,no-store");
                if (PropertyUtil.isDirectRead()) {
                    response.setDateHeader("Last-Modified", lastModified);
                } else {
                    fis = new FileInputStream(file);
                    response.setDateHeader("Last-Modified", file.lastModified());
                }
                //如果是单机模式，直接把数据写回到客户端
                if (PropertyUtil.isDirectRead()) {
                    Pair<String, String> pair = EncryptionHandler.decryptHandler(dataId,
                            configInfoBase.getEncryptedDataKey(), configInfoBase.getContent());
                    out = response.getWriter();
                    out.print(pair.getSecond());
                    out.flush();
                    out.close();
                } else {
					//从磁盘获取文件
                    String fileContent = IoUtils.toString(fis, Charsets.UTF_8.name());
                    String encryptedDataKey = cacheItem.getEncryptedDataKey();
                    Pair<String, String> pair = EncryptionHandler.decryptHandler(dataId, encryptedDataKey, fileContent);
                    String decryptContent = pair.getSecond();
                    out = response.getWriter();
                    out.print(decryptContent);
                    out.flush();
                    out.close();
                }
                
                LogUtil.PULL_CHECK_LOG.warn("{}|{}|{}|{}", groupKey, requestIp, md5, TimeUtils.getCurrentTimeStr());
                
                final long delayed = System.currentTimeMillis() - lastModified;
                
                // TODO distinguish pull-get && push-get
                /*
                 Otherwise, delayed cannot be used as the basis of push delay directly,
                 because the delayed value of active get requests is very large.
                 */
                ConfigTraceService.logPullEvent(dataId, group, tenant, requestIpApp, lastModified,
                        ConfigTraceService.PULL_EVENT_OK, delayed, requestIp, notify && isSli);
                
            } finally {
                //释放锁
                releaseConfigReadLock(groupKey);
                IoUtils.closeQuietly(fis);
            }
            //文件不存在返回404
        } else if (lockResult == 0) {
            
            // FIXME CacheItem No longer exists. It is impossible to simply calculate the push delayed. Here, simply record it as - 1.
            ConfigTraceService
                    .logPullEvent(dataId, group, tenant, requestIpApp, -1, ConfigTraceService.PULL_EVENT_NOTFOUND, -1,
                            requestIp, notify && isSli);
            
            return get404Result(response);
           
            //走到这里说明获取锁失败，存在冲突
        } else {
            
            PULL_LOG.info("[client-get] clientIp={}, {}, get data during dump", clientIp, groupKey);
            
            response.setStatus(HttpServletResponse.SC_CONFLICT);
            response.getWriter().println("requested file is being modified, please try later.");
            return HttpServletResponse.SC_CONFLICT + "";
            
        }
        
        return HttpServletResponse.SC_OK + "";
    }
```

总体来说，服务端获取配置主要有如下步骤:
1.先对传递的参数，dataId,tenant(namespaceid),tag,group进行校验
2.调用ConfigServletInner.doGetConfig方法获取服务端的文件
3.首先会对dataId,group,tenant进行拼接，他们作为锁的key来获取锁
	a.lockResult>0表示获取锁成功 ，表示CacheItem(也就是缓存的配置项)不为空，并且已经加了读锁，意味着这个缓存数据不能被删除。
	然后判断是否配置了beta,tag等配置，默认是不进行配置，获取文件类型设置在响应头中，如果是standalone模式 从数据库中读取，
	如果是集群模式，从磁盘中获取文件，得到的是一个完整的File，因为集群模式中每个节点都会缓存一份文件在本地，最终把数据返回给客户端
	b.lockResult=0表示文件不存在返回404
	c.lockResult<0表示取锁失败，存在冲突
	
### 7.客户端配置缓存更新
我们知道`NacosPropertySourceLocator.locate()`该方法会对配置文件加载，即获取服务端的配置，这里会初始化ConfigService，NacosConfigService实现了ConfigService接口，
这里是利用反射机制实例化 NacosConfigService 对象
ConfigService是客户端对NacosServer配置操作顶层抽象接口，并提供唯一实现类NacosConfigService，
通过分析NacosConfigService源码可以理解Nacos作为配置中心和客户端交互机制。


- NacosConfigService
```java
public NacosConfigService(Properties properties) throws NacosException {
	ValidatorUtils.checkInitParam(properties);
	String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
	if (StringUtils.isBlank(encodeTmp)) {
		this.encode = Constants.ENCODE;
	} else {
		this.encode = encodeTmp.trim();
	}
	initNamespace(properties);
	this.configFilterChainManager = new ConfigFilterChainManager(properties);
	
	this.agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
	this.agent.start();
	this.worker = new ClientWorker(this.agent, this.configFilterChainManager, properties);
}


@SuppressWarnings("PMD.ThreadPoolCreationRule")
    public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager,
            final Properties properties) {
        this.agent = agent;
        this.configFilterChainManager = configFilterChainManager;
        
        // Initialize the timeout parameter
        
        init(properties);
        
        this.executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });
        
        this.executorService = Executors
                .newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = new Thread(r);
                        t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                        t.setDaemon(true);
                        return t;
                    }
                });
        
        this.executor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                try {
                    checkConfigInfo();
                } catch (Throwable e) {
                    LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
                }
            }
        }, 1L, 10L, TimeUnit.MILLISECONDS);
    }
	
    public void checkConfigInfo() {
        // Dispatch tasks.
        int listenerSize = cacheMap.size();
        // Round up the longingTaskCount. 分片的思想进行处理（3000个配置文件作为一个批次）
        int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
        if (longingTaskCount > currentLongingTaskCount) {
            for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
				//每个批次的任务提交到线程池进行处理
                executorService.execute(new LongPollingRunnable(i));
            }
            currentLongingTaskCount = longingTaskCount;
        }
    }
```


- LongPollingRunnable.run方法



```java
@Override
        public void run() {
            
            List<CacheData> cacheDatas = new ArrayList<CacheData>();
            List<String> inInitializingCacheList = new ArrayList<String>();
            try {
                // check failover config
				//遍历缓存中的数据
                for (CacheData cacheData : cacheMap.values()) {
					//任务id相同的数据放在一起，利用分片的思想进行数据的处理
                    if (cacheData.getTaskId() == taskId) {
                        cacheDatas.add(cacheData);
                        try {
							//检查磁盘数据和缓存中数据是否一致，不一致就更新
                            checkLocalConfig(cacheData);
							//这里为true说明缓存发生变化，触发一个监听
                            if (cacheData.isUseLocalConfigInfo()) {
                                cacheData.checkListenerMd5();
                            }
                        } catch (Exception e) {
                            LOGGER.error("get local config info error", e);
                        }
                    }
                }
                
                // check server config
				//把当前批次发生变化的文件，发送到服务端进行比较
                List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);
                if (!CollectionUtils.isEmpty(changedGroupKeys)) {
                    LOGGER.info("get changedGroupKeys:" + changedGroupKeys);
                }
                
                for (String groupKey : changedGroupKeys) {
                    String[] key = GroupKey.parseKey(groupKey);
                    String dataId = key[0];
                    String group = key[1];
                    String tenant = null;
                    if (key.length == 3) {
                        tenant = key[2];
                    }
                    try {
                        ConfigResponse response = getServerConfig(dataId, group, tenant, 3000L);
                        CacheData cache = cacheMap.get(GroupKey.getKeyTenant(dataId, group, tenant));
                        cache.setContent(response.getContent());
                        cache.setEncryptedDataKey(response.getEncryptedDataKey());
                        if (null != response.getConfigType()) {
                            cache.setType(response.getConfigType());
                        }
                        LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}, type={}",
                                agent.getName(), dataId, group, tenant, cache.getMd5(),
                                ContentUtils.truncateContent(response.getContent()), response.getConfigType());
                    } catch (NacosException ioe) {
                        String message = String
                                .format("[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s",
                                        agent.getName(), dataId, group, tenant);
                        LOGGER.error(message, ioe);
                    }
                }
                for (CacheData cacheData : cacheDatas) {
                    if (!cacheData.isInitializing() || inInitializingCacheList
                            .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                        cacheData.checkListenerMd5();
                        cacheData.setInitializing(false);
                    }
                }
                inInitializingCacheList.clear();
                
                executorService.execute(this);
                
            } catch (Throwable e) {
                
                // If the rotation training task is abnormal, the next execution time of the task will be punished
                LOGGER.error("longPolling error : ", e);
                executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
            }
        }
    }
```

```java 
    private void checkLocalConfig(CacheData cacheData) {
        final String dataId = cacheData.dataId;
        final String group = cacheData.group;
        final String tenant = cacheData.tenant;
        File path = LocalConfigInfoProcessor.getFailoverFile(agent.getName(), dataId, group, tenant);
        
		//缓存没有，本地路径有，更新缓存
        if (!cacheData.isUseLocalConfigInfo() && path.exists()) {
            String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
            final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
            cacheData.setUseLocalConfigInfo(true);
            cacheData.setLocalConfigInfoVersion(path.lastModified());
            cacheData.setContent(content);
            String encryptedDataKey = LocalEncryptedDataKeyProcessor
                    .getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
            cacheData.setEncryptedDataKey(encryptedDataKey);
            
            LOGGER.warn(
                    "[{}] [failover-change] failover file created. dataId={}, group={}, tenant={}, md5={}, content={}",
                    agent.getName(), dataId, group, tenant, md5, ContentUtils.truncateContent(content));
            return;
        }
        
		//本地缓存有，文件不存在，本地缓存置为false，以路径为主
        // If use local config info, then it doesn't notify business listener and notify after getting from server.
        if (cacheData.isUseLocalConfigInfo() && !path.exists()) {
            cacheData.setUseLocalConfigInfo(false);
            LOGGER.warn("[{}] [failover-change] failover file deleted. dataId={}, group={}, tenant={}", agent.getName(),
                    dataId, group, tenant);
            return;
        }
        
		//缓存和本地磁盘数据不一致，进行更新
        // When it changed.
        if (cacheData.isUseLocalConfigInfo() && path.exists() && cacheData.getLocalConfigInfoVersion() != path
                .lastModified()) {
            String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
            final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
            cacheData.setUseLocalConfigInfo(true);
            cacheData.setLocalConfigInfoVersion(path.lastModified());
            cacheData.setContent(content);
            String encryptedDataKey = LocalEncryptedDataKeyProcessor
                    .getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
            cacheData.setEncryptedDataKey(encryptedDataKey);
            LOGGER.warn(
                    "[{}] [failover-change] failover file changed. dataId={}, group={}, tenant={}, md5={}, content={}",
                    agent.getName(), dataId, group, tenant, md5, ContentUtils.truncateContent(content));
        }
    }
```

Nacos的长轮询机制
