---
title: SpringCache源码分析(3) @CachePut、@Cacheable、@CacheEvict注解解析
date: "2016-01-23T22:18:00+08:00"
tags:
    - spring cache
url: 2016/01/23/spring-cache-3/
---

----------

&#160;&#160;&#160;&#160;
从上一篇中我们提及到，既然方法使用了@CachePut、@Cacheable、@CacheEvict这些注解，那么，执行方法是怎么知道设置了哪些注解信息的呢？ 下面我们分析一下注解的解析。但，在分析之前我们需要知道缓存操作封装类CacheOperation。

----------

### CacheOperation缓存操作封装类

&#160;&#160;&#160;&#160;
CacheOperation抽象类表示缓存的基本操作，封装了缓存基本操作的一些属性信息，也就是在@CachePut、@Cacheable、@CacheEvict中配置的属性信息。CacheOperation有三个实现类，分别是CachePutOperation、CacheableOperation、CacheEvictOperation，不用说也知道这是对注解信息的封装类。下面简单的看一下CacheOperation封装了哪些信息。

```
public abstract class CacheOperation implements BasicOperation {

	private String name = "";

	private Set<String> cacheNames = Collections.emptySet();

	private String key = "";

	private String keyGenerator = "";

	private String cacheManager = "";

	private String cacheResolver = "";

	private String condition = "";

	/* 省略了一些setter、getter方法 */

}
```

&#160;&#160;&#160;&#160;
这些属性是不是很熟悉，就是在配置@CachePut、@Cacheable、@CacheEvict使用的一些公共属性。CacheOperation没什么好分析的，我们只要知道最终注解信息都被转换成CacheOperation对象就可以了。至于是哪个具体的实现，我们现在还不用关心，后面会知道。


----------

### CacheAnnotationParser缓存注解解析器

&#160;&#160;&#160;&#160;
CacheAnnotationParser接口提供了2个方法，分别对类和方法上的注解进行解析。因为我们知道@CachePut可以在类或方法上进行注解。

&#160;&#160;&#160;&#160;
CacheAnnotationParser这是一个策略接口，它是委派给AnnotationCacheOperationSource来执行的，AnnotationCacheOperationSource内部持有一个CacheAnnotationParser集合，可以执行一系列的CacheAnnotationParser实现。也就是说我们可以自定义像@CachePut一样的注解，然后提供对应的CacheAnnotationParser解析器，达到我们自定义缓存功能的效果。下面是CacheAnnotationParser接口的源码。

```
/**
 * CacheAnnotationParser是为解析注解提供的一个策略接口，AnnotationCacheOperationSource 会委派一些解析器来解析特定的注解类型，像Spring Cache的@Cacheable、@CachePut、@CacheEvict
 */
public interface CacheAnnotationParser {

	/**
     * 对给定类型上的缓存注解进行解析，如果没有则返回null
	 * @see AnnotationCacheOperationSource#findCacheOperations(Class)
	 */
	Collection<CacheOperation> parseCacheAnnotations(Class<?> type);

	/**
     * 对给定方法上的缓存注解进行解析，如果没有则返回null
	 * @see AnnotationCacheOperationSource#findCacheOperations(Method)
	 */
	Collection<CacheOperation> parseCacheAnnotations(Method method);

}
```

&#160;&#160;&#160;&#160;
那么@CachePut、@Cacheable、@CacheEvict到底是哪个类来解析的呢？ 就是，SpringCacheAnnotationParser。下面我们看看SpringCacheAnnotationParser是如何解析注解的，我们还是看看CacheAnnotationParser提供的接口方法，代码如下：

```
@Override
public Collection<CacheOperation> parseCacheAnnotations(Class<?> type) {
	//获取@CacheConfig注解信息
	DefaultCacheConfig defaultConfig = getDefaultCacheConfig(type);
	//解析其他的缓存注解@Caching、@CachePut、@Cacheable、@CacheEvict
	return parseCacheAnnotations(defaultConfig, type);
}

@Override

public Collection<CacheOperation> parseCacheAnnotations(Method method) {
	DefaultCacheConfig defaultConfig = getDefaultCacheConfig(method.getDeclaringClass());
	return parseCacheAnnotations(defaultConfig, method);
}
```

&#160;&#160;&#160;&#160;
从上面的代码可以看出，这2个方法解析方式一样，都调用了相同的方法。第一，解析@CacheConfig注解，调用了getDefaultCacheConfig(Class<?> target)方法；第二，解析其他缓存注解，都调用了parseCacheAnnotations(DefaultCacheConfig cachingConfig, AnnotatedElement ae)这个方法。

 1. DefaultCacheConfig获取@CacheConfig注解信息
    ```
     DefaultCacheConfig getDefaultCacheConfig(Class<?> target) {
    
    	//直接获取目标类型上的@CacheConfig注解，如果有则将属性信息设置好，如果没有则给定一个默认的DefaultCacheConfig对象。
    	CacheConfig annotation = AnnotationUtils.getAnnotation(target, CacheConfig.class);
    
    	if (annotation != null) {
    		return new DefaultCacheConfig(annotation.cacheNames(), annotation.keyGenerator(),
    				annotation.cacheManager(), annotation.cacheResolver());
    	}
    
    	return new DefaultCacheConfig();
    }
    ```

 2. 获取CacheOperation集合信息
    ```
    protected Collection<CacheOperation> parseCacheAnnotations(DefaultCacheConfig cachingConfig, AnnotatedElement ae) {
    
    	Collection<CacheOperation> ops = null;
    
    	//解析@Cacheable
    	Collection<Cacheable> cacheables = getAnnotations(ae, Cacheable.class);
    	if (cacheables != null) {
    		ops = lazyInit(ops);
    		for (Cacheable cacheable : cacheables) {
    			ops.add(parseCacheableAnnotation(ae, cachingConfig, cacheable));
    		}
    	}
    
    	//解析@CacheEvict
    	Collection<CacheEvict> evicts = getAnnotations(ae, CacheEvict.class);
    	if (evicts != null) {
    		ops = lazyInit(ops);
    		for (CacheEvict evict : evicts) {
    			ops.add(parseEvictAnnotation(ae, cachingConfig, evict));
    		}
    	}
    
    	//解析@CachePut
    	Collection<CachePut> puts = getAnnotations(ae, CachePut.class);
    	if (puts != null) {
    		ops = lazyInit(ops);
    		for (CachePut put : puts) {
    			ops.add(parsePutAnnotation(ae, cachingConfig, put));
    		}
    	}
    
    	//解析@Caching
    	Collection<Caching> cachings = getAnnotations(ae, Caching.class);
    	if (cachings != null) {
    		ops = lazyInit(ops);
    		for (Caching caching : cachings) {
    			ops.addAll(parseCachingAnnotation(ae, cachingConfig, caching));
    		}
    	}
    	
    	//最后，返回解析出来的所有缓存操作对象CacheOperation集合
    	return ops;
    }
    ```

&#160;&#160;&#160;&#160;
这些代码虽然很简单，但是有些细节还是需要说明：
1. 如果存在自定义注解使用了@CachePut等这些缓存注解，该怎么解析呢？ 到底需不需解析到？
2. 前面说过@CachConfig是一个总注解配置，那么，@CacheConfig的注解信息如何优先于其他注解信息呢？
3. 在@CachePut、@Cacheable、@CacheEvict注解信息中，key和keyGenerator是排它性的，cacheManager和cacheResolve也是排它性的，为什么呢？

&#160;&#160;&#160;&#160;
答案就在我们的眼前，下面我们看看 getAnnotations(AnnotatedElement ae, Class<T> annotationType) 和 parsePutAnnotation(AnnotatedElement ae, DefaultCacheConfig defaultConfig, CachePut cachePut) （三个方法之选一个说明，其他两个都一样）

----------

#### 自定义注解中存在@CachePut等缓存注解

&#160;&#160;&#160;&#160;
假设我们自定以了一个注解类型@UserCache，如下：
```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Cacheable(value="user")
public @interface UserCache{

}
```

&#160;&#160;&#160;&#160;
如果一个方法使用了@UserCache，那么，@UserCache中使用的@Cacheable是否也应该包含该方法的缓存注解解析中去呢？ 答案是肯定的，需要包含。但是如果注解层级关系不止一层，如果是二层注解关系，又会是怎样的呢？多层次的注解关系是不会被解析到的。下面看看getAnnotations()这个方法，分析如下：

```
private <T extends Annotation> Collection<T> getAnnotations(AnnotatedElement ae, Class<T> annotationType) {

	Collection<T> anns = new ArrayList<T>(2);

	//查找原生的注解
	T ann = ae.getAnnotation(annotationType);
	if (ann != null) {
		anns.add(AnnotationUtils.synthesizeAnnotation(ann, ae));
	}

	//扫描注解中的注解，注意：这里只包含了一层注解关系，如果出现多层次的注解关系是扫描不到的！
	for (Annotation metaAnn : ae.getAnnotations()) {
		ann = metaAnn.annotationType().getAnnotation(annotationType);
		if (ann != null) {
			anns.add(AnnotationUtils.synthesizeAnnotation(ann, ae));
		}
	}

	return (anns.isEmpty() ? null : anns);
}
```

----------

#### @CacheConfig的高优先级别

&#160;&#160;&#160;&#160;
前面早已说明过@CacheConfig的作用要高于@CachePut、@Cacheable、@CacheEvict，但是，为什么呢？这些也只能算是我们的道听途说。作为一个Coding，我们不要那些道听途说，别人说的不一定是对的。下面的代码会给你一个真理，让你也知道：哦，原来就是这么简单！ 我们就拿@CachePut的解析来说，下面是解析的方法。（@Cacheable、@CacheEvict一样就不赘述啦）

```
CacheOperation parsePutAnnotation(AnnotatedElement ae, DefaultCacheConfig defaultConfig, CachePut cachePut) {

	CachePutOperation op = new CachePutOperation();
	op.setCacheNames(cachePut.cacheNames());
	op.setCondition(cachePut.condition());
	op.setUnless(cachePut.unless());
	op.setKey(cachePut.key());
	op.setKeyGenerator(cachePut.keyGenerator());
	op.setCacheManager(cachePut.cacheManager());
	op.setCacheResolver(cachePut.cacheResolver());
	op.setName(ae.toString());

	//这里会去对op对象做一次重新的设定，到底做了什么，我想你会懂的！
	//这里会把@CacheConfig中配置的注解信息重新设置一边，也就是覆盖前面设置过的值。
	defaultConfig.applyDefault(op);
	validateCacheOperation(ae, op);

	return op;
}
```

----------

#### key和keyGenerator、cacheManager和cacheResolve排它性

&#160;&#160;&#160;&#160;
从下面的代码中，很容易知道，如果key和keyGenerator、cacheManager和cacheResolve同时存在就会抛出异常了。

```
private void validateCacheOperation(AnnotatedElement ae, CacheOperation operation) {

	if (StringUtils.hasText(operation.getKey()) && StringUtils.hasText(operation.getKeyGenerator())) {
		throw new IllegalStateException("Invalid cache annotation configuration on '" +
				ae.toString() + "'. Both 'key' and 'keyGenerator' attributes have been set. " +
				"These attributes are mutually exclusive: either set the SpEL expression used to" +
				"compute the key at runtime or set the name of the KeyGenerator bean to use.");
	}

	if (StringUtils.hasText(operation.getCacheManager()) && StringUtils.hasText(operation.getCacheResolver())) {
		throw new IllegalStateException("Invalid cache annotation configuration on '" +
				ae.toString() + "'. Both 'cacheManager' and 'cacheResolver' attributes have been set. " +
				"These attributes are mutually exclusive: the cache manager is used to configure a" +
				"default cache resolver if none is set. If a cache resolver is set, the cache manager" +
				"won't be used.");
	}

}
```

----------

&#160;&#160;&#160;&#160;
@CacheConfig、@Caching、@CachePut、@Cacheable、@CacheEvict这些缓存注解的解析分析完了，总结一句话，就是将缓存注解都解析成了Collection&lt;CacheOperation&gt;对象。

&#160;&#160;&#160;&#160;
解析成Collection&lt;CacheOperation&gt;对象，什么时候调用呢？ 前面提及过CacheAnnotationParser提供了对缓存注解的解析策略。具体的调用并不是由CacheAnnotationParser直接处理的，而是由AnnotationCacheOperationSource委派给CacheAnnotationParser来处理。AnnotationCacheOperationSource是CacheOperationSource接口的实现类，也就是说调用工作是由CacheOperationSource来负责的。

----------

### CacheOperationSource 缓存操作调用

&#160;&#160;&#160;&#160;
下面，我们直接了当地看CacheOperationSource接口的定义：
```
/**

 * 这个接口是由CacheInterceptor使用
 * Interface used by {@link CacheInterceptor}. Implementations know how to source
 */
public interface CacheOperationSource {

	/**
     * 方法功能很明确：为一个执行方法返回所有缓存注解的缓存操作集合
	 */
	Collection<CacheOperation> getCacheOperations(Method method, Class<?> targetClass);

}
```

&#160;&#160;&#160;&#160;
现在，我们的思路应该很清楚了。那就是，CacheInterceptor调用了CacheOperationSource，CacheOperationSource委派给CacheAnnotationParser去解析执行方法上的注解，然后返回一个Collection<CacheOperation>给CacheInterceptor。就是这样！

&#160;&#160;&#160;&#160;
CacheInterceptor怎么调用了CacheOperationSource我们现在暂且不管，下一篇中分析。 下面，我们看看，CacheOperationSource如何委派CacheAnnotationParser解析注解。

----------

##### AnnotationCacheOperationSource 委派给注解解析器

&#160;&#160;&#160;&#160;
AnnotationCacheOperationSource的继承关系是： AnnotationCacheOperationSource --> AbstractFallbackCacheOperationSource --> CacheOperationSource。所以我们还是先从AbstractFallbackCacheOperationSource开始分析，然后分析AnnotationCacheOperationSource。

&#160;&#160;&#160;&#160;
首先，看看AbstractFallbackCacheOperationSource提供的属性和实现CacheOperationSource的接口方法：
```
public abstract class AbstractFallbackCacheOperationSource implements CacheOperationSource {

	/**
	 * 这里的key是AnnotatedElementKey对象，这个对象只是对执行方法method和目标类型targetClass的包装。
	 * 也就是说，保证了method+targetClass产生key的唯一性。
	 */
	private final Map<Object, Collection<CacheOperation>> attributeCache =
			new ConcurrentHashMap<Object, Collection<CacheOperation>>(1024);

	/**
	 * 为调用的方法获取缓存属性，如果方法中找不到，则从类型中找。
	 */
	@Override
	public Collection<CacheOperation> getCacheOperations(Method method, Class<?> targetClass) {

		//获取key，也就是AnnotatedElementKey对象(当作一般的保证key唯一性标志就好了)
		Object cacheKey = getCacheKey(method, targetClass);

		//从Map中获取，如果获取不到，则进行解析
		Collection<CacheOperation> cached = this.attributeCache.get(cacheKey);

		if (cached != null) {
			return (cached != NULL_CACHING_ATTRIBUTE ? cached : null);
		}
		else {
			//解析执行方法的缓存操作集合
			Collection<CacheOperation> cacheOps = computeCacheOperations(method, targetClass);
			if (cacheOps != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Adding cacheable method '" + method.getName() + "' with attribute: " + cacheOps);
				}
				this.attributeCache.put(cacheKey, cacheOps);
			}
			else {
				this.attributeCache.put(cacheKey, NULL_CACHING_ATTRIBUTE);
			}
			return cacheOps;
		}
	}

}
```

&#160;&#160;&#160;&#160;
上面的代码很容易明白：当一个方法执行时，会去解析该方法的缓存操作集合，解析后放入到attributeCache这个Map中去，以便下次直接获取。

&#160;&#160;&#160;&#160;
但是，是怎么计算一个执行方法的缓存操作集合呢？ 在computeCacheOperations(method, targetClass)这个类中，下面跟进去瞧瞧。
```
private Collection<CacheOperation> computeCacheOperations(Method method, Class<?> targetClass) {

	// 判断no-public方法是否可以获取缓存操作集合，子类可以重写allowPublicMethodsOnly()方法，默认返回的是false。
	if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
		return null;
	}

	//方法可能是接口上的，所以需要去找到一个特定的方法。
	Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
	specificMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

	//首先，试着从方法上找缓存操作集合（这是一个抽象方法）
	Collection<CacheOperation> opDef = findCacheOperations(specificMethod);
	if (opDef != null) {
		return opDef;
	}

	//其次，再试着从类型上找缓存操作集合（这也是一个抽象方法）
	opDef = findCacheOperations(specificMethod.getDeclaringClass());
	if (opDef != null) {
		return opDef;
	}

	//如果前面都没有找到，并且specificMethod != method，则从原始执行方法method上去找
	if (specificMethod != method) {
		//先从方法上找
		opDef = findCacheOperations(method);
		if (opDef != null) {
			return opDef;
		}

		//最后，再从类型上找
		return findCacheOperations(method.getDeclaringClass());
	}
	
	return null;
}
```

&#160;&#160;&#160;&#160;
从上面的分析可以知道，AbstractFallbackCacheOperationSource做了2件事情：第一，增加了Map作为缓存，以便后续可以直接获取。第二，处理了方法fallback（备援）的情况。

&#160;&#160;&#160;&#160;
接下来，我们需要看看findCacheOperations(specificMethod)和findCacheOperations(specificMethod.getDeclaringClass())这两个抽象方法是怎么实现的了，不用到说，实现类就是AnnotationCacheOperationSource了。我们再想想，有一个接口提供了和这两个类似的方法，那就是CacheAnnotationParser接口。到这里，我们终于把CacheOperationSource和CacheAnnotationParser联系起来了。下面再看看代码是如何实现的：
```
public class AnnotationCacheOperationSource extends AbstractFallbackCacheOperationSource implements Serializable {

	//定义是否只用public方法可以被缓存操作
	private final boolean publicMethodsOnly;

	//缓存注解解析器集合
	private final Set<CacheAnnotationParser> annotationParsers;

	@Override
	protected Collection<CacheOperation> findCacheOperations(final Class<?> clazz) {
	
		//实际上就是遍历annotationParsers
		return determineCacheOperations(new CacheOperationProvider() {
			@Override
			public Collection<CacheOperation> getCacheOperations(CacheAnnotationParser parser) {
				return parser.parseCacheAnnotations(clazz);
			}
		});

	}

	@Override
	protected Collection<CacheOperation> findCacheOperations(final Method method) {

		return determineCacheOperations(new CacheOperationProvider() {
			@Override
			public Collection<CacheOperation> getCacheOperations(CacheAnnotationParser parser) {
				return parser.parseCacheAnnotations(method);
			}
		});

	}

	/**
	 * 这里就是在遍历CacheAnnotationParser集合了，最后把所有的Collection<CacheOperation>返回
	 */
	protected Collection<CacheOperation> determineCacheOperations(CacheOperationProvider provider) {

		Collection<CacheOperation> ops = null;
		for (CacheAnnotationParser annotationParser : this.annotationParsers) {
			Collection<CacheOperation> annOps = provider.getCacheOperations(annotationParser);
			if (annOps != null) {
				if (ops == null) {
					ops = new ArrayList<CacheOperation>();
				}
				ops.addAll(annOps);
			}
		}

		return ops;
	}

	/**
	 * 重写了父类的方法，可以配置no-public缓存操作
	 */
	@Override
	protected boolean allowPublicMethodsOnly() {
		return this.publicMethodsOnly;
	}

	/**
	 * 只是为了便于遍历调用CacheAnnotationParser
	 */
	protected interface CacheOperationProvider {
		Collection<CacheOperation> getCacheOperations(CacheAnnotationParser parser);
	}

}
```

### 总结

1. CacheAnnotationParser将@CachePut等注解解析成CacheOperation集合对象
2. CacheOperationSource委派CacheAnnotationParser来获取解析到的CacheOperation集合
3. CacheInterceptor调用CacheOperationSource

&#160;&#160;&#160;&#160;
在下一篇中我们继续分析CacheInterceptor的调用。

