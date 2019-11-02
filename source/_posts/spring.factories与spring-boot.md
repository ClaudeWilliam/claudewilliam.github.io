---
title: spring.factories与spring-boot
date: 2019-10-22 09:21:08
tags: [java,springboot]
categories: spring
---

#### 什么是spring.factories

spring.factories是一个配置文件（spring的工厂），放到META-INF文件夹中。存放key-value关系的文件（key就是bean工厂的接口，下面的value就是对应的实现类）。主要用于Spring-Boot中扩展的问题。这个有点像Java SPI。spring.factories文件存放的是全类名，用于加载到Spring的Bean容器中。而不是通过外部声明注入。（Bean工厂实际就是生成Bean）

在spring-core的中`SpringFactoriesLoader`类去加载配置在spring.factories中的类（也就是之前定义好的装     好自定义Bean的类）。这个是spring-boot-autoconfigure的jar包中META-INF文件夹中spring.factories中的配置文件。这里面包含初始化(Initializers)的Bean Factory的key `org.springframework.context.ApplicationContextInitializer`;应用监听器（Application Listeners）的key  `org.springframework.context.ApplicationListener`；配置的过滤器（Auto Configuration Import Listeners） `org.springframework.boot.autoconfigure.AutoConfigurationImportListener`;自动配置（Auto Configure）`org.springframework.boot.autoconfigure.EnableAutoConfiguration`；可以看出用`,`分割。

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnClassCondition

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration

# Failure analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.autoconfigure.diagnostics.analyzer.NoSuchBeanDefinitionFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.DataSourceBeanCreationFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.HikariDriverConfigurationFailureAnalyzer,\
org.springframework.boot.autoconfigure.session.NonUniqueSessionRepositoryFailureAnalyzer

# Template availability providers
org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.mustache.MustacheTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.web.servlet.JspTemplateAvailabilityProvider

```

#### Spring如何处理spring.factories文件

在spring-core的包中SpringFactoriesLoader类去加载spring.factories文件。里面有描述他如何加载这些bean工厂。下面我们就看源码去分析一下，加载spring.factories文件

```java
public abstract class SpringFactoriesLoader {

	/**
	 * 存放spring.factories文件的目录，可以存在多个jar包的目录
	 */
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

	//打印日志工具类，springboot一般使用logback打印日志。现在打印日志都是异步的。
	private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
	//是缓存，用于存放类加载器和对应类和bean工厂的名字和对应的实现类
	private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();


	/**
     *从指定位置，加载指定工厂（实现bean工厂接口）的实例，使用它们给的classLoader。
     *工厂都是排序 使用的是（AnnotationAwareOrderComparator）
     *如果需要自定义实例化策略，请使用{loadFactoryNames方法}
     *获取所有注册的工厂名称。
     *@param factoryClass 工厂的抽象类和接口
     *@param classLoader 加载类的类加载器
	 */
	public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
        //对应的工厂类不能是空，也就是spring.factories中的key值
		Assert.notNull(factoryClass, "'factoryClass' must not be null");
		ClassLoader classLoaderToUse = classLoader; //传入类加载器赋值。
		if (classLoaderToUse == null) {//如果传入的类加载器为空，使用SpringFactoriesLoader类的加载器
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}
        //获取指定bean工厂（key）的所有工厂名字，即这个接口的实现
		List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);
		if (logger.isTraceEnabled()) {//打印日志
			logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
		}
        //根据指定bean工厂的class创建一个类列表
		List<T> result = new ArrayList<>(factoryNames.size());
		for (String factoryName : factoryNames) {
            //使用classLoader实例化对应类名的实现类
			result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
		}
        //排序
		AnnotationAwareOrderComparator.sort(result);
		return result;
	}

	/**
	 * 加载给定bean工厂实现类的全类名，key是抽象类或者接口。（白话一点就是是把key对应的value全部加载出来，用逗号分割成数组转成List<string>,感觉挺low）
	 */
	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        //获取bean工厂类的全类名
		String factoryClassName = factoryClass.getName();
        //获取bean工厂类的实例类的列表
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}
	//获取bean工厂接口（抽象类）与实现类的Map,Map<key,List<value>>
	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        //从缓存中获取对应关系
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {//如果缓存不为空直接返回
			return result;
		}

		try {
            //根据classLoader获取Properties文件，如果classLoader为空，那句获取系统默认的Properties文件
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>(); //创建对应关系列表
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);//构建获取Properties的路径。
                //获取Properties文件
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				//解析Properties文件
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
					List<String> factoryClassNames = Arrays.asList(
							StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));//把用都号分割的值转成数组，在转成list
                    //放到Map里面，这里是把这个properties里的所有key的value值都遍历出来
					result.addAll((String) entry.getKey(), factoryClassNames);
				}
			}
            //存到缓存
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
            //找不到文件的异常
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
	//根据提供的实例化类，返回实例化的对象
	@SuppressWarnings("unchecked")
	private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader) {
		try {//使用classLoader获取类对象
			Class<?> instanceClass = ClassUtils.forName(instanceClassName, classLoader);
			//判断类是不是可以访问的，
            if (!factoryClass.isAssignableFrom(instanceClass)) {
				throw new IllegalArgumentException(
						"Class [" + instanceClassName + "] is not assignable to [" + factoryClass.getName() + "]");
			}
            //返回通过反射构造的实例
			return (T) ReflectionUtils.accessibleConstructor(instanceClass).newInstance();
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException("Unable to instantiate factory class: " + factoryClass.getName(), ex);
		}
	}

}

```

经过上面的分析我们能给出，他加载Bean工厂的实例和过程。根据类名获取加载类信息。下面是具体的流程图。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/spring-factories.png)

上面就是，spring-boot在启动时候加载的Bean工厂。说到这个自动加载Bean工厂，也就是spring-boot很很核心的东西，就是自动配置，在SpringBootApplication类中去启用自动配置，那么好多之前在Jar里面的Bean就可以在Spring启动时候注入进去，而不是要在Xml中做固定的配置，他会自动去找，这样可以使用自动配置。可以使用spring-boot-starter中的jar中bean工厂，直接创建默认的bean。

#### Spring-Boot的自动配置

##### @SpringBootApplication注解

一般实现自动配置都是使用`@SpringBootApplication`注解，下面我们就分析一下这个注解的源码

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	/**
	 * 排除特定的自动配置类，使其永远不会应用。返回一个特定排除的类，作用于EnableAutoConfiguration注解
	 * @return the classes to exclude
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	/**
	 * 排除特定的自动配置类名称，以使它们永远不会被应用。作用于EnableAutoConfiguration注解
	 */
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	/**
	 *基本包的扫描组件，采用字符串安全类型的值代替。作用于ComponentScan注解
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	/**
	 *用于指定要扫描带注释的组件。 指定类别的包装将被扫描。作用于ComponentScan注解
	 *
	 */
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}

```

我们从源码可以看出来`@SpringBootApplication`注解是复合注解，包括自动配置的注解`@EnableAutoConfiguration`组件包扫描的注解`@ComponentScan`还有配置的注解`@SpringBootConfiguration`。而自动配置是依赖于`@EnableAutoConfiguration`这个注解。

##### @EnableAutoConfiguration注解

```java
/**
 *通过这个接口启动SpringContext（Spring上下文）自动配置。会推测你将要使用那些Bean来，通常会根据
 *你配置的类路径和类名去配置Bean，自动Bean配置是基于spring.factories机制实现的。如果不想配置可以去
 *exclude一些类
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	//用于判断是否开启自动配置，这里是开启的。存放到Environment中的值
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	//用于排除特定的类
	Class<?>[] exclude() default {};

	//用于排除特定的配置名称
	String[] excludeName() default {};

}
```

从上面的类中我们只是看到了注解，和一个Impor导入的Selector。这个`AutoConfigurationImportSelector`就是自动注入的核心的。`AutoConfigurationImportSelector`类方法不多，大多数都是`private`和`protected`的方法。我们说一下，几个被别人调用的`public`方法。

##### AutoConfigurationImportSelector类

```java
public class AutoConfigurationImportSelector
		implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
		BeanFactoryAware, EnvironmentAware, Ordered {
	//不需要自动注入时候返回的空白数组
	private static final String[] NO_IMPORTS = {};
	//打印日志
	private static final Log logger = LogFactory
			.getLog(AutoConfigurationImportSelector.class);
	//存放exclude的Properties的文件
	private static final String PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE = "spring.autoconfigure.exclude";
	//Spring的bean工厂
	private ConfigurableListableBeanFactory beanFactory;
	//环境变量
	private Environment environment;
	//类加载器
	private ClassLoader beanClassLoader;
	//资源加载器，一般用于加载Properties文件
	private ResourceLoader resourceLoader;

	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		//获取注解上是否开启自动注入，如果不是直接返回
        if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
        //从配置文件中加载 AutoConfigurationMetadata
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // 获取所有候选配置类EnableAutoConfiguration
		// 使用了内部工具使用SpringFactoriesLoader，查找classpath上所有jar包中的
		// META-INF\spring.factories，找出其中key为
		// org.springframework.boot.autoconfigure.EnableAutoConfiguration
        // 的属性定义的工厂类名称。
		// 虽然参数有annotationMetadata,attributes,但在 AutoConfigurationImportSelector 的
		// 实现 getCandidateConfigurations()中，这两个参数并未使用
        //根据Bean工厂的名称获取对应实现类的全类名。这里使用的是loadFactoryNames，不是loadFatories
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
        //去掉重复的配置
		configurations = removeDuplicates(configurations);
        //获取到排除的类信息（排除类的全类名）
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        //校验排除类的class信息，应用 exclusion 属性
		checkExcludedClasses(configurations, exclusions);
        //自动从配置的Bean工厂实现类中去除掉exclude的类名
		configurations.removeAll(exclusions);
        // 应用过滤器AutoConfigurationImportFilter，
		// 对于 spring boot autoconfigure，定义了一个需要被应用的过滤器 ：
		// org.springframework.boot.autoconfigure.condition.OnClassCondition，
		// 此过滤器检查候选配置类上的注解@ConditionalOnClass，如果要求的类在classpath
		// 中不存在，则这个候选配置类会被排除掉
		configurations = filter(configurations, autoConfigurationMetadata);
       	// 现在已经找到所有需要被应用的候选配置类
		// 广播事件 AutoConfigurationImportEvent
		fireAutoConfigurationImportEvents(configurations, exclusions);
        //转换成String数组
		return StringUtils.toStringArray(configurations);
	}

	//判断是否开启自动注入
	protected boolean isEnabled(AnnotationMetadata metadata) {
        //获取当前类是不是AutoConfigurationImportSelector类，如果是，根据存在Environment（环境）的EnableAutoConfiguration属性去判断。
		if (getClass() == AutoConfigurationImportSelector.class) {
			return getEnvironment().getProperty(
					EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
					true);
		}
		return true;
	}

	//获取注解的属性信息	
	protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
        //获取EnableAutoConfiguration注解类的属性
		String name = getAnnotationClass().getName();
		AnnotationAttributes attributes = AnnotationAttributes
				.fromMap(metadata.getAnnotationAttributes(name, true));
		Assert.notNull(attributes,
				() -> "No auto-configuration attributes found. Is "
						+ metadata.getClassName() + " annotated with "
						+ ClassUtils.getShortName(name) + "?");
		return attributes;
	}
	//获取注解类
	protected Class<?> getAnnotationClass() {
		return EnableAutoConfiguration.class;
	}

	/**
	 * 通过SpringFactoriesLoader获取对应的配置信息，从META-INF/spring.factories文件
	 * key就是getSpringFactoriesLoaderFactoryClass()提供的类
	 */
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}

	/**
	 * 返回自动配置Bean工厂类名 org.springframework.boot.autoconfigure.EnableAutoConfiguration 
	 */
	protected Class<?> getSpringFactoriesLoaderFactoryClass() {
		return EnableAutoConfiguration.class;
	}
    //教员和处理exclude类
	private void checkExcludedClasses(List<String> configurations,
			Set<String> exclusions) {
        //创建无效的exclude的集合，如果能确定集合的大小，就在创建时候指定，这样不用去扩容，提高性能
		List<String> invalidExcludes = new ArrayList<>(exclusions.size());
		for (String exclusion : exclusions) {
            //如果exclude的类，在当前类加载器可以获取到，同时不在自动配置的类的集合。如果exclude的类不在自动配置的列表，就必要exclude。后面会抛出异常
			if (ClassUtils.isPresent(exclusion, getClass().getClassLoader())
					&& !configurations.contains(exclusion)) {
                //加入到失效的exclude的列表
				invalidExcludes.add(exclusion);
			}
		}
        //如果要处理的集合不是空，开始处理集合
		if (!invalidExcludes.isEmpty()) {
			handleInvalidExcludes(invalidExcludes);
		}
	}

	/**
	 * 抛出一个异常，当一个类不是自动注入的Bean工厂实现类，就不应该被exclude掉，打印message
	 **/
	protected void handleInvalidExcludes(List<String> invalidExcludes) {
		StringBuilder message = new StringBuilder();
		for (String exclude : invalidExcludes) {
			message.append("\t- ").append(exclude).append(String.format("%n"));
		}
		throw new IllegalStateException(String
				.format("The following classes could not be excluded because they are"
						+ " not auto-configuration classes:%n%s", message));
	}

	/**
	 *
	 */
	protected Set<String> getExclusions(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		Set<String> excluded = new LinkedHashSet<>();
		excluded.addAll(asList(attributes, "exclude"));
		excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));
		excluded.addAll(getExcludeAutoConfigurationsProperty());
		return excluded;
	}
	//获取通过配置文件，配置的exclude类。spring.autoconfigure.exclude这个key对应的值
	private List<String> getExcludeAutoConfigurationsProperty() {
        //获取自动配置的环境变量属性
		if (getEnvironment() instanceof ConfigurableEnvironment) {
            //返回spring.autoconfigure.exclude下面的值
			Binder binder = Binder.get(getEnvironment());
			return binder.bind(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class)
					.map(Arrays::asList).orElse(Collections.emptyList());
		}
        //如果不是自动配置的环境，则从环境中获取这个值，如果为null,返回空list
		String[] excludes = getEnvironment()
				.getProperty(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class);
		return (excludes != null) ? Arrays.asList(excludes) : Collections.emptyList();
	}
 /**
	  * 根据autoConfigurationMetadata信息对候选配置类configurations进行过滤
	  **/
	private List<String> filter(List<String> configurations,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		long startTime = System.nanoTime();
		String[] candidates = configurations.toArray(new String[configurations.size()]);
		// 记录候选配置类是否需要被排除,skip为true表示需要被排除,全部初始化为false,不需要被排除
		boolean[] skip = new boolean[candidates.length];
		// 记录候选配置类中是否有任何一个候选配置类被忽略，初始化为false
		boolean skipped = false;
		// 获取AutoConfigurationImportFilter并逐个应用过滤
		for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
			// 对过滤器注入其需要Aware的信息
			invokeAwareMethods(filter);
			// 使用此过滤器检查候选配置类跟autoConfigurationMetadata的匹配情况
			boolean[] match = filter.match(candidates, autoConfigurationMetadata);
			for (int i = 0; i < match.length; i++) {
				if (!match[i]) {
				// 如果有某个候选配置类不符合当前过滤器，将其标记为需要被排除，
				// 并且将 skipped设置为true，表示发现了某个候选配置类需要被排除
					skip[i] = true;
					skipped = true;
				}
			}
		}
		if (!skipped) {
		// 如果所有的候选配置类都不需要被排除，则直接返回外部参数提供的候选配置类集合
			return configurations;
		}
		// 逻辑走到这里因为skipped为true，表明上面的的过滤器应用逻辑中发现了某些候选配置类
		// 需要被排除，这里排除那些需要被排除的候选配置类，将那些不需要被排除的候选配置类组成
		// 一个新的集合返回给调用者
		List<String> result = new ArrayList<String>(candidates.length);
		for (int i = 0; i < candidates.length; i++) {
			if (!skip[i]) {
				result.add(candidates[i]);
			}
		}
		if (logger.isTraceEnabled()) {
			int numberFiltered = configurations.size() - result.size();
			logger.trace("Filtered " + numberFiltered + " auto configuration class in "
					+ TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
					+ " ms");
		}
		return new ArrayList<String>(result);
	}	
	/**
	  * 使用内部工具 SpringFactoriesLoader，查找classpath上所有jar包中的
	  * META-INF\spring.factories，找出其中key为
	  * org.springframework.boot.autoconfigure.AutoConfigurationImportFilter 
	  * 的属性定义的过滤器类并实例化。
	  * AutoConfigurationImportFilter过滤器可以被注册到 spring.factories用于对自动配置类
	  * 做一些限制，在这些自动配置类的字节码被读取之前做快速排除处理。
	  * spring boot autoconfigure 缺省注册了一个 AutoConfigurationImportFilter :
	  * org.springframework.boot.autoconfigure.condition.OnClassCondition
	**/
	protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
		return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class,
				this.beanClassLoader);
	}	      
	
	private void fireAutoConfigurationImportEvents(List<String> configurations,
			Set<String> exclusions) {
		List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
		if (!listeners.isEmpty()) {
			AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this,
					configurations, exclusions);
			for (AutoConfigurationImportListener listener : listeners) {
				invokeAwareMethods(listener);
				listener.onAutoConfigurationImportEvent(event);
			}
		}
	}
	//去列表重复的字符串，这里用了hashset去重
	protected final <T> List<T> removeDuplicates(List<T> list) {
		return new ArrayList<>(new LinkedHashSet<>(list));
	}
	//根据属性值的String数组转List
	protected final List<String> asList(AnnotationAttributes attributes, String name) {
		String[] value = attributes.getStringArray(name);
		return Arrays.asList((value != null) ? value : new String[0]);
	}
            
   
	//返回自动配置的Listener的bean工厂
	protected List<AutoConfigurationImportListener> getAutoConfigurationImportListeners() {
		return SpringFactoriesLoader.loadFactories(AutoConfigurationImportListener.class,
				this.beanClassLoader);
	}
	//调用注入的方法，根据实例的不同，设置不同的到不同的变量去
	private void invokeAwareMethods(Object instance) {
		if (instance instanceof Aware) {
			if (instance instanceof BeanClassLoaderAware) {
				((BeanClassLoaderAware) instance)
						.setBeanClassLoader(this.beanClassLoader);
			}
			if (instance instanceof BeanFactoryAware) {
				((BeanFactoryAware) instance).setBeanFactory(this.beanFactory);
			}
			if (instance instanceof EnvironmentAware) {
				((EnvironmentAware) instance).setEnvironment(this.environment);
			}
			if (instance instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) instance).setResourceLoader(this.resourceLoader);
			}
		}
	}
}
```

上面是讲如何通过这个注解获取到，spring.fatories中的bean工厂的配置。然后Springboot应用启动过程中使用ConfigurationClassParser分析配置类时，如果发现注解中存在@Import(ImportSelector)的情况，就会创建一个相应的ImportSelector对象， 并调用其方法 public String[] selectImports(AnnotationMetadata annotationMetadata), 这里 AutoConfigurationImportSelector的导入@Import(AutoConfigurationImportSelector.class) 就属于这种情况,所以ConfigurationClassParser会实例化一个 AutoConfigurationImportSelector并调用它的 selectImports() 方法。实现Bean的注册和注入。关于ConfigurationClassParser的实现，这里有点长，我们后续在扩展（埋坑）。

##### 自动注入的流程图

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/enable-auto-configuration.png)

#### 总结

在日常工作中，我们可能需要实现一些 Spring Boot Starter 给被人使用，这个时候我们就可以使用这个 Factories 机制，将自己的 Starter 注册到 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 命名空间下。这样用户只需要在服务中引入我们的 jar 包即可完成自动加载及配置。Factories 机制可以让 Starter 的使用只需要很少甚至不需要进行配置。

本质上，Spring Factories 机制与 Java SPI 机制原理都差不多：都是通过指定的规则，在规则定义的文件中配置接口的各种实现，通过 key-value 的方式读取，并以反射的方式实例化指定的类型。

了解关于自动注入，也是依赖了几个注解。后面我会写一个关于自己如何实现一个spring-boot-starter。

#### 参考

[Spring Boot 自动配置及 Factories 机制总结](https://qidawu.github.io/2019/01/20/spring-factories/)

[Spring EnableAutoConfigurationImportSelector 是如何工作的](<https://blog.csdn.net/andy_zhang2007/article/details/78580980>)