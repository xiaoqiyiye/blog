---
title: Spring Cache (1) -- CacheManager和Cache
date: "2016-01-22T11:00:00+08:00"
tags:
    - spring cache
url: 2016/01/23/spring-cache-1/
---

这篇作为Spring Cache源码分析的起始篇，重要在于分析CacheManager和Cache。但是，在分析CacheManager和Cache之前，还是先看一个简单的例子，这样有助于理解Spring Cache的概念，知道Spring Cache在干什么，有什么作用，只有知道了Spring Cache的用处，在分析源码的时候才能知道Spring Cache的功能是怎么实现的！ 这里不讲使用的细节，如果想要了解细节请看其他质料或后面篇涨中的详细分析。

### Spring Cache Hello示例
下面直接上示例代码，一个简单的Hello程序，Hello、HelloService、HelloTest。在下面代码中我们用到了注解：@CachePut，@Cacheable，@CacheEvict。从单词意思我们就应该知道这些的作用是什么，@CachePut用于把数据存放到缓存中；@Cacheable用于从缓存中获取数据，如果缓存中不存在就执行代码得到并存放在缓存中去，以便下次从缓存中获取；@CacheEvict用于驱除缓存中的数据。在后面的章节中会详细的讲解这些注解中的每个属性。

Hello对象：
```
public class Hello {
	String name;
	public Hello(String name){
		this.name = name;
	}
	public String getName() {
		return name;
	}
	@Override
	public String toString() {
		return "Hello," + name;
	}
}
```

HelloService对象：
```
@Service
public class HelloService {
	/**
	 * 以Hello#name属性域作为缓存key（不管缓存是否存在，都会去执行）
	 * @param hello
	 * @return
	 */
	@CachePut(value="hello", key="#hello.name")
	public Hello put(Hello hello){
		System.out.println("put Hello:" + hello.getName());
		return hello;
	}

	/**
	 * 以name参数为key
	 * 如果缓存中没有，则执行代码并缓存
	 * 如果缓存中已经存在，则直接从缓存中获取
	 * @param name
	 * @return
	 */
	@Cacheable(value="hello", key="#name")
	public Hello get(String name){
		System.out.println("new Hello:" + name);
		return new Hello(name);
	}

	/**
	 * 从命名为"hello"的缓存中，删除掉name参数的key
	 * @param name
	 */
	@CacheEvict(value="hello", key="#name")
	public void remove(String name){
		System.out.println("remove Hello:" + name);
	}

	/**
	 * 从命名为"hello"的缓存中，删除所有缓存
	 */
	@CacheEvict(value="hello", allEntries=true)
	public void removeAll(){
		System.out.println("remove all!");
	}
}
```

HelloTest测试：
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="classpath:applicationContext.xml")
public class HelloTest extends AbstractJUnit4SpringContextTests{

	@Test
	public void hello(){
		HelloService service = applicationContext.getBean(HelloService.class);
		service.get("linya");
		Hello hello = service.get("linya");
		System.out.println(hello.toString());
	}
}
```
我们可以试着运行上面的测试文件，但是程序是不能运行的，为什么呢，因为需要配置文件applicationContext.xml。上面我们说过通过注解可以缓存、获取、删除数据，那么数据被缓存到了哪里呢？很显然这样需要applicationContext.xml配置文件来处理，指明数据需要缓存的地方，这个缓存的地方在Spring Cache被定义为Cache和CacheManager，下面我们来看看如何简单的配置Spring Cache。
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:cache="http://www.springframework.org/schema/cache"
	xsi:schemaLocation="
        http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
        http://www.springframework.org/schema/context 
        http://www.springframework.org/schema/context/spring-context-4.1.xsd
        http://www.springframework.org/schema/cache
        http://www.springframework.org/schema/cache/spring-cache-4.1.xsd"
	default-lazy-init="true">
	<context:component-scan base-package="seven.xiaoqiyiye.spring.cache"/>
    <context:annotation-config/>
    
	<!-- SpringCache驱动一定要配置，后面会详细讲解这个配置的作用 -->
    <cache:annotation-driven/>

	<!-- 配置CacheManager，CacheManager中管理着Cache集合，在这里配置了一个名称为"hello"的Cache-->
	<bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
        <property name="caches">
            <set>
                <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean">
                    <property name="name" value="hello"/>
                </bean>
            </set>
        </property>
    </bean>
</beans>
```

### CacheManager 和 Cache ###
CacheManager定义很简单，用于管理Cache集合，并提供通过Cache名称获取对应Cache对象的方法。下面是CacheManager接口定义：
```
/**
 * Spring's central cache manager SPI.
 * Allows for retrieving named {@link Cache} regions.
 * CacheManager可以通过名称来获取一个Cache对象
 * @author Costin Leau
 * @since 3.1
 */
public interface CacheManager {

	/**
	 * 通过name来获取Cache对象
	 */
	Cache getCache(String name);
	
	/**
     * 返回这个CacheManager管理的Cache集合的所有Cache名称
	 */
	Collection<String> getCacheNames();
}
```

从上面的CacheManager可以知道，每个Cache必须要指定一个name，这个name需要在CacheManager中是唯一的。另外Cache对象还需要支持一些数据操作，存放数据、获取数据、驱除数据等等。下面，我们看看Cache接口的定义：
```
/**
 * Interface that defines common cache operations.
 */
public interface Cache {

	/**
     * 获取该Cache的名称
	 */
	String getName();

	/**
     * 返回底层真是的缓存对象，接口中并不需要关系具体是怎么实现的，
     * 使用Map、Reids、Guava，Ecache等
	 */
	Object getNativeCache();

	/**
     * 返回被包装的值，主要是为了null的处理
	 */
	ValueWrapper get(Object key);

	/**
     * 返回缓存中指定key的值，并获得这个值的特定类型
	 */
	<T> T get(Object key, Class<T> type);

	/**
     * 存放key-value数据到缓存中
	 */
	void put(Object key, Object value);

	ValueWrapper putIfAbsent(Object key, Object value);

	/**
     * 从缓存中删除指定key的数据
	 */
	void evict(Object key);

	/**
     * 删除缓存中所有数据
	 * Remove all mappings from the cache.
	 */
	void clear();

	interface ValueWrapper {
		Object get();
	}

}
```

CacheManager和Cache接口定义就是这么简单，下面再看看CacheManager和Cache的实现类。它们的实现类很多，我们这里选基于ConcurrentMap的实现：ConcurrentMapCacheManager和ConcurrentMapCache。
```
/**
 * ConcurrentMapCacheManager负责管理ConcurrentMapCache对象，支持懒加载获取，也支持预先实例化对象，
 * 通过调用#setCacheNames方法可以预定义一些ConcurrentMapCache到ConcurrentMapCacheManager中，
 * 但是一旦设置过后，dynamic就会设置为false，这时就不再支持动态创建Cache功能。
 *
 * 通常是否懒加载可以通过不同的构造器来控制，new ConcurrentMapCacheManager()创建懒加载的CacheManager，
 * new ConcurrentMapCacheManager(String... cacheNames)创建预先定义Cache的CacheManager。
 */
public class ConcurrentMapCacheManager implements CacheManager {
	
	//使用ConcurrentMap来管理Cache集合对象
	private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<String, Cache>(16);
	//表示是否可以动态创建Cache缓存，如果指定过name那么就不能动态创建
	private boolean dynamic = true;
	
	/**
	 * Construct a dynamic ConcurrentMapCacheManager,
	 * lazily creating cache instances as they are being requested.
	 */
	public ConcurrentMapCacheManager() {

	}

	/**
	 * Construct a static ConcurrentMapCacheManager,
	 * managing caches for the specified cache names only.
	 */
	public ConcurrentMapCacheManager(String... cacheNames) {
		setCacheNames(Arrays.asList(cacheNames));
	}
	
	/**
     * 这里会初始化ConcurrentMapCache，并存放到ConcurrentMap中进行管理
     * 一旦初始化过，这dynamic=false，在调用getCache(name)是就不会动态创建了
	 */
	public void setCacheNames(Collection<String> cacheNames) {
		if (cacheNames != null) {
			for (String name : cacheNames) {
				this.cacheMap.put(name, createConcurrentMapCache(name));
			}
			this.dynamic = false;
		}
		else {
			this.dynamic = true;
		}
	}

	@Override
	public Collection<String> getCacheNames() {
		return Collections.unmodifiableSet(this.cacheMap.keySet());
	}

	/**
	 * 根据缓存name来获取关联的ConcurrentMapCache实例
	 * 如果ConcurrentMapCacheManager中没有获取到，则动态获取。（能否动态获取需要看dynamic是否为true）
	 */
	@Override
	public Cache getCache(String name) {
		Cache cache = this.cacheMap.get(name);
		if (cache == null && this.dynamic) {
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
					cache = createConcurrentMapCache(name);
					this.cacheMap.put(name, cache);
				}
			}
		}
		return cache;
	}

	/**
	 * Create a new ConcurrentMapCache instance for the specified cache name.
	 */
	protected Cache createConcurrentMapCache(String name) {
		return new ConcurrentMapCache(name, isAllowNullValues());
	}

}
```

CacheManager和Cache是不是很简单？是的，非常简单！ 可是在上面的applicationContext.xml并没有配置ConcurrentMapCacheManager和ConcurrentMapCache呀，在回顾一下applicationContext.xml中是怎么配置的吧，配置如下：
```
<bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
    <property name="caches">
        <set>
            <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean">
                <property name="name" value="hello"/>
            </bean>
        </set>
    </property>
</bean>
```

在applicationContext.xml中配置了SimpleCacheManager和ConcurrentMapCacheFactoryBean，SimpleCacheManager也是一个CacheManager的实现类，一个比ConcurrentMapCacheManager更简单的实现类，它需要外部指定Cache集合对象，而这个Cache对象正是使用ConcurrentMapCacheFactoryBean来注入到Spring的。虽然这两个类很简单，但是还是看一下部分源码吧（去掉了一些方法）。
```
public class ConcurrentMapCacheFactoryBean implements FactoryBean<ConcurrentMapCache>, BeanNameAware, InitializingBean {

	//定义Cache的名称
	private String name = "";

	//注入到Spring时缓存存储的数据
	private ConcurrentMap<Object, Object> store;

	//是否允许null值
	private boolean allowNullValues = true;

	//真实的Cache实现类

	private ConcurrentMapCache cache;


	/**
	 * Specify the name of the cache.
	 * <p>Default is "" (empty String).
	 */
	public void setName(String name) {
		this.name = name;
	}

	/**
	 * Specify the ConcurrentMap to use as an internal store
	 * (possibly pre-populated).
	 * <p>Default is a standard {@link java.util.concurrent.ConcurrentHashMap}.
	 */
	public void setStore(ConcurrentMap<Object, Object> store) {
		this.store = store;
	}

	/**
	 * Set whether to allow {@code null} values
	 * (adapting them to an internal null holder value).
	 * <p>Default is "true".
	 */
	public void setAllowNullValues(boolean allowNullValues) {
		this.allowNullValues = allowNullValues;
	}

	/**
	 * 以Spring注入的beanName作为Cache的名称
	 */
	@Override
	public void setBeanName(String beanName) {
		if (!StringUtils.hasLength(this.name)) {
			setName(beanName);
		}
	}

	/**
	 * 对象注入到Spring的时候就初始化好了Cache对象（ConcurrentMapCache）
	 */
	@Override
	public void afterPropertiesSet() {
		this.cache = (this.store != null ? new ConcurrentMapCache(this.name, this.store, this.allowNullValues) :
				new ConcurrentMapCache(this.name, this.allowNullValues));
	}

	@Override
	public ConcurrentMapCache getObject() {
		return this.cache;
	}
}
```

其实，在applicationContext.xml中就负责配置了CacheManager，告诉Spring Cache使用什么要的CacheManager实现，接下来我们使用ConcurrentMapCacheManager来配置，可以达到同样的效果。
```
<bean id="cacheManager" class="org.springframework.cache.concurrent.ConcurrentMapCacheManager">
    <!-- 可以不设置cacheNames哦，设置之后就不能动态创建Cache了。前面代码已经分析过，明白了吗！ -->
    <property name="cacheNames">
    	<set>
    		<value>hello</value>
    		<value>world</value>
    	</set>
    </property>
</bean>
```

通过上面的分析，我们已经清楚地了解了CacheManager和Cache接口的作用，以及基于ConcurrentMap的实现。但是，分析了这么久，那Spring到底是怎么缓存数据的呢？ @CachePut、@Cacheable、@CacheEvict是怎么产生作用的呢？ 莫急，莫急，这个在后续章节中详细说明。至少我们现在知道，数据被存储到哪里去了！ 对，数据被存储在Cache是实现类里，就这么简单！

CacheManager和Cache还有基于Redis、Guava、EhCache、JCache的实现，这里就不分析了。哈哈，其实原理都一样！