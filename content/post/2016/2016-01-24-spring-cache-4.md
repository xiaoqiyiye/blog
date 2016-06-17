---
title: SpringCache (4) CacheInterceptor、
date: "2016-01-24T13:00:00+08:00"
tags:
    - spring cache
url: 2016/01/24/spring-cache-4/
---


----------

### 缓存拦截器的实现

&#160;&#160;&#160;&#160;
这一篇我们讲讲SpringCache对方法的拦截器实现，也就是CacheInterceptor。在使用SpringCache我们会比较关注的问题，为什么对一个方法使用@CachePut注解后，就可以达到缓存的效果呢？下面我们将揭开这层面纱。Spring Cache实现对方法的拦截功能，是由CacheInterceptor提供的。下面直接看看CacheInterceptor是怎么做的呢？

&#160;&#160;&#160;&#160;
CacheInterceptor实现了MethodInterceptor接口，在Spring AOP中，MethodInterceptor的功能是做方法拦截。现在应该明白，为什么使用@CachePut等注解后可以实现缓存操作，因为方法被拦截处理了。CacheInterceptor的实现很简单，啥都不用啰说了，先看看代码实现。

```
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

	/*
	 *被拦截的方法都会调用invoke方法，不懂的可以先看看Spring AOP。
	 */
	@Override
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Method method = invocation.getMethod();
		
		//这里就是对执行方法调用的一次封装，主要是为了处理对异常的包装。
		CacheOperationInvoker aopAllianceInvoker = new CacheOperationInvoker() {
			@Override
			public Object invoke() {
				try {
					return invocation.proceed();
				}
				catch (Throwable ex) {
					throw new ThrowableWrapper(ex);
				}
			}
		};

		try {
			//真正地去处理缓存操作的执行，很显然这是父类的方法，所以我们要到父类CacheAspectSupport中去看看。
			return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
		}
		catch (CacheOperationInvoker.ThrowableWrapper th) {
			throw th.getOriginal();
		}
	}

}
```

&#160;&#160;&#160;&#160;
下面，我们再看看CacheAspectSupport#execute(...)这个方法中具体怎么进行缓存操作的。

```
protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {

	//标志Spring加载元素是否都准备好了，是否可以执行了
	if (this.initialized) {
		Class<?> targetClass = getTargetClass(target);
		//这里使用的就是CacheOperationSource，来获取执行方法上所有的缓存操作集合。如果有缓存操作则执行到execute(...)，如果没有就执行invoker.invoke()直接调用执行方法了。
		Collection<CacheOperation> operations = getCacheOperationSource().getCacheOperations(method, targetClass);
		if (!CollectionUtils.isEmpty(operations)) {
			return execute(invoker, new CacheOperationContexts(operations, method, args, target, targetClass));
		}
	}

	return invoker.invoke();
}
```

&#160;&#160;&#160;&#160;
在上面的代码中出现了CacheOperationContexts对象，这个对象只是为了便于获取每种具体缓存操作集合。我们知道所有的缓存操作CachePutOperation、CacheableOperation、CacheEvictOperation都存放在operations这个集合中，不便于获取具体的缓存操作，所以封装成了缓存操作上下文CacheOperationContexts这个类。接下来，我们继续看看核心代码，庐山真面目即将揭晓。先罗列一下@CachePut、@Cacheable、@CacheEvict的功能，再来看看代码是怎么实现的。

1. @CachePut  -- 执行方法后，将方法返回结果存放到缓存中。不管有没有缓存过，执行方法都会执行，并缓存返回结果（unless可以否决进行缓存）。（当然，这里说的缓存都要满足condition条件）
2. @Cacheable -- 如果没有缓存过，获取执行方法的返回结果；如果缓存过，则直接从缓存中获取，不再执行方法。
3. @CacheEvict -- 如果设置了beforeIntercepte则在方法执行前进行缓存删除操作，如果没有，则在执行方法调用完后进行缓存删除操作。

```
private Object execute(CacheOperationInvoker invoker, CacheOperationContexts contexts) {

	// 处理beforeIntercepte=true的缓存删除操作
	processCacheEvicts(contexts.get(CacheEvictOperation.class), true, ExpressionEvaluator.NO_RESULT);

	// 从缓存中查找，是否有匹配@Cacheable的缓存数据
	Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

	// 如果@Cacheable没有被缓存，那么就需要将数据缓存起来，这里将@Cacheable操作收集成CachePutRequest集合，以便后续做@CachePut缓存数据存放。
	List<CachePutRequest> cachePutRequests = new LinkedList<CachePutRequest>();
	if (cacheHit == null) {
		collectPutRequests(contexts.get(CacheableOperation.class), ExpressionEvaluator.NO_RESULT, cachePutRequests);
	}

	Cache.ValueWrapper result = null;

	// 如果没有@CachePut操作，就使用@Cacheable获取的结果（可能也没有@Cableable，所以result可能为空）。
	// hasCachePut(contexts)判断是否有可缓存的操作，这里处理了@Cacheable、@CachePut中unless否决缓存的情况，关于unless后面会分析到。
	if (cachePutRequests.isEmpty() && !hasCachePut(contexts)) {
		result = cacheHit;
	}

	// 1. 既然从缓存中没有获取到数据，那么就执行方法内容吧。
	// 2. 如果有@CachePut，那么result肯定为null，所以@CachePut操作之前，肯定会先调用执行方法内容。
	// 3. 如果执行了方法，那么这里的result就是执行方法返回的结果。
	if (result == null) {
		result = new SimpleValueWrapper(invokeOperation(invoker));
	}

	// 收集@CachePut操作
	collectPutRequests(contexts.get(CachePutOperation.class), result.get(), cachePutRequests);

	// 处理@CachePut操作，将数据result数据存放到缓存中去。
	for (CachePutRequest cachePutRequest : cachePutRequests) {
		cachePutRequest.apply(result.get());
	}

	// 处理一般的@CacheEvict缓存删除操作情况，也就是beforeIntercepte=false的情况。
	processCacheEvicts(contexts.get(CacheEvictOperation.class), false, result.get());

	//返回方法执行的返回结果
	return result.get();
}
```

&#160;&#160;&#160;&#160;
上面已经分析了@CachePut、@Cacheable、@CacheEvict注解功能的具体功能实现，Spring Cache的功能基本已经了解的差不多了。但是，我们从之前的CacheOperation、CacheAnnotationParser、CacheAnnotationSource、CacheInterceptor一路分析过来，但是还是发现少了些什么。不禁会想CacheManager和Cache在哪里？唯独缺少了我们在applicationContext.xml中配置的CacheManager和Cache。我们抱着打破沙锅问到底的学习目的，下面接下来看看缓存对象Cache的获取。

----------

### 缓存对象Cache的获取

&#160;&#160;&#160;&#160;
首先我们先整理一下上面的CacheAspectSupport#execute(CacheOperationInvoker invoker, CacheOperationContexts contexts)这个方法中出现过的类。CacheOperationContexts、CachePutRequest、CacheOperationContext等等。下面我们将对这些类对象也做下详细的分析。

#### CacheOperationContexts  获取具体的缓存操作类型

&#160;&#160;&#160;&#160;
前面提到过，CacheOperationContexts是对Collection<CacheOperation>缓存操作集合做的一次封装处理，目的是为了可以获得具体缓存操作。由于很简单，我们直接分析源码：

```
private class CacheOperationContexts {

	//保存每种类型缓存操作的上下文数据，Map中的key是CacheOperation类型，也就是@CachePut、@Cacheable、@CacheEvict对应的3中CacheOperation实现类型。
	//Map中的value是CacheOperationContext。另外注意，这个Map不是普通的Map，而是一个MultiValueMap，这种Map的key是可以重复存放的。
	private final MultiValueMap<Class<? extends CacheOperation>, CacheOperationContext> contexts =
			new LinkedMultiValueMap<Class<? extends CacheOperation>, CacheOperationContext>();

	public CacheOperationContexts(Collection<? extends CacheOperation> operations, Method method,
			Object[] args, Object target, Class<?> targetClass) {
		//获取每种CacheOperation类型的缓存操作集合，然后保存到Map中去。
		for (CacheOperation operation : operations) {
			this.contexts.add(operation.getClass(), getOperationContext(operation, method, args, target, targetClass));
		}
	}
	
	//根据CacheOperation类型，直接从Map中获取对应的缓存操作上下文集合
	//比如： oprationClass为CachePutOperation，那么就是获取的所有@CachePut注解对应的缓存操作的上下文集合
	public Collection<CacheOperationContext> get(Class<? extends CacheOperation> operationClass) {
		Collection<CacheOperationContext> result = this.contexts.get(operationClass);
		return (result != null ? result : Collections.<CacheOperationContext>emptyList());
	}
}
```

&#160;&#160;&#160;&#160;
既然通过CacheOperationContexts#get(oprationClass)方法返回的是Collection<CacheOperationContext>，那么，我们是不是应该了解下CacheOperationContext包含哪些信息呢？ 接下来分析CacheOperationContext。

##### CacheOperationContext  封装缓存参数信息， condition、unless处理

&#160;&#160;&#160;&#160;
CacheOperationContext这个类我们要好好地去研究下，为什么呢？ 因为它包含了缓存条件(conditions)的判断，以及缓存对象Cache的获取，这些都是我们在分析源码的时候比较关心的东西。为了更好地理解CacheOperationContext的含义，我们先从属性开始了解，CacheOperationContext提供了一下的属性：

```
//缓存操作对象的源数据对象，封装了CacheOperation、method、targetClass、keyGenerator、cacheResolve
//metadata中的属性是通过CacheOperation中的属性来设置的，也就是@CachePut(keyGeneraor="kg", cacheResolve="cr")
//这样就达到了自定义keyGeneraor的效果
private final CacheOperationMetadata metadata;

//执行方法的参数
private final Object[] args;

//执行方法的目标类对象
private final Object target;

//执行方法可以获取到的缓存对象集合，也就是@CachePut等设置的value值关联的那个Cache对象
private final Collection<? extends Cache> caches;

//执行方法使用缓存注解设置的缓存名称，例如:@CachePut(value="cacheName")
private final Collection<String> cacheNames;

//表示目标类型的执行方法标识的key值
private final AnnotatedElementKey methodCacheKey;
```

&#160;&#160;&#160;&#160;
接下来，我们看看CacheOperationContext中主要的方法：

```
protected class CacheOperationContext implements CacheOperationInvocationContext<CacheOperation> {

	public CacheOperationContext(CacheOperationMetadata metadata, Object[] args, Object target) {
		this.metadata = metadata;
		this.args = extractArgs(metadata.method, args);
		this.target = target;
		//获取缓存对象Cache
		this.caches = CacheAspectSupport.this.getCaches(this, metadata.cacheResolver);
		this.cacheNames = createCacheNames(this.caches);
		this.methodCacheKey = new AnnotatedElementKey(metadata.method, metadata.targetClass);
	}

	//这个方法用来判断缓存条件condition
	protected boolean isConditionPassing(Object result) {
		//首先判断CacheOperation是否设置了conditions条件
		//如果没有设置条件，则直接通过条件检测
		//如果设置了条件，那么通过evaluator去判断（ExpressionEvaluator evaluator 会通过SpEL表达式去检测）
		if (StringUtils.hasText(this.metadata.operation.getCondition())) {
			EvaluationContext evaluationContext = createEvaluationContext(result);
			return evaluator.condition(this.metadata.operation.getCondition(),
					this.methodCacheKey, evaluationContext);
		}
		return true;
	}

	//处理@Cacheable、@CachePut中unless，如果unless通过SpEL检测，则否决存放缓存
	protected boolean canPutToCache(Object value) {
		String unless = "";
		if (this.metadata.operation instanceof CacheableOperation) {
			unless = ((CacheableOperation) this.metadata.operation).getUnless();
		}
		else if (this.metadata.operation instanceof CachePutOperation) {
			unless = ((CachePutOperation) this.metadata.operation).getUnless();
		}
		if (StringUtils.hasText(unless)) {
			EvaluationContext evaluationContext = createEvaluationContext(value);
			return !evaluator.unless(unless, this.methodCacheKey, evaluationContext);
		}
		return true;
	}

	/**
	 * Cache中的key值都是通过KeyGenerator来生成的，默认使用了SimpleKeyGenerator。
	 */
	protected Object generateKey(Object result) {
		if (StringUtils.hasText(this.metadata.operation.getKey())) {
			EvaluationContext evaluationContext = createEvaluationContext(result);
			return evaluator.key(this.metadata.operation.getKey(), this.methodCacheKey, evaluationContext);
		}
		//使用KeyGenerator生成Cache中的缓存key值
		return this.metadata.keyGenerator.generate(this.target, this.metadata.method, this.args);
	}

	//EvaluationContext对象用于SpEL表达式检测，关于SpEL不做深入分析
	private EvaluationContext createEvaluationContext(Object result) {
		return evaluator.createEvaluationContext(
				this.caches, this.metadata.method, this.args, this.target, this.metadata.targetClass, result);
	}

}
```

##### CacheResolver 获取缓存对象Cache

&#160;&#160;&#160;&#160;
在CacheOperationContext的构造方法中，使用了this.caches = CacheAspectSupport.this.getCaches(this, metadata.cacheResolver)来设置caches。我们看看CacheResolver是如何解析出Cache对象的。我们直接到AbstractCacheResolver#resolveCaches(CacheOperationInvocationContext<?> context)这个方法中去。

```
public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {

	//获取缓存名称，默认使用SimpleCacheResolver，也就是获取@CachePut(value={"cacheName1", "cacheName2"})注解中的值。
	Collection<String> cacheNames = getCacheNames(context);
	if (cacheNames == null) {
		return Collections.emptyList();
	}
	else {
		//通过缓存名称，从CacheManager中去找到关联的Cache对象。
		Collection<Cache> result = new ArrayList<Cache>();
		for (String cacheName : cacheNames) {
			Cache cache = this.cacheManager.getCache(cacheName);
			if (cache == null) {
				throw new IllegalArgumentException("Cannot find cache named '" +
						cacheName + "' for " + context.getOperation());
			}
			result.add(cache);
		}
		return result;
	}
}
```

&#160;&#160;&#160;&#160;
缓存对象Cache我们已经获取到了，接下来就应该可以调用缓存操作了吧，比如put(key,value)方法。 接下来我们继续分析CachePutRequest对象。

##### CachePutRequest  缓存存放操作请求

&#160;&#160;&#160;&#160;
CachePutRequest很简单，就是请求将方法返回结果result以key存放到缓存对象Cache中去，调用了doPut方法， 这是CacheInterceptor父类AbstractCacheInvoker提供的，很简单，就不贴源码了。

```
private class CachePutRequest {

	private final CacheOperationContext context;

	private final Object key;

	public CachePutRequest(CacheOperationContext context, Object key) {
		this.context = context;
		this.key = key;
	}

	public void apply(Object result) {
		if (this.context.canPutToCache(result)) {
			for (Cache cache : this.context.getCaches()) {
				doPut(cache, this.key, result);
			}
		}
	}
}
```


----------


### 总结

&#160;&#160;&#160;&#160;
用了好几篇来讲述SpringCache是怎么处理@CachePut、@Cacheable、@CacheEvict操作的，按照我们之前的思路一路分析过来：

1. CacheOperation封装了@CachePut、@Cacheable、@CacheEvict的属性信息，以便于能够获得被拦截方法的缓存操作集合。
2. CacheAnnotationParser将@CachePut、@Cacheable、@CacheEvict注解解析成CacheOperation集合。(也包含了对@Caching、@CacheConfig的解析)
3. CacheAnnotationSource获取执行方法的缓存操作集合，这个获取的过程是委派给CacheAnnotationParser去做的。CacheAnnotationParser充当了解析注解的策略接口。
4. CacheInterceptor实现了MethodInterceptor接口，在Spring AOP中实现对执行方法的拦截。在调用invoke方法时，是通过调用CacheAnnotationSource来获取缓存操作集合的。
5. CacheInterceptor的父类CacheAspectSupport实现了@CachePut、@Cacheable、@CacheEvict的缓存功能。
6. 每一个缓存操作CacheOperation最后被封装成了CacheOperationContext，CacheOperationContext通过CacheResolver解析出缓存对象Cache。
7. 最后CacheInterceptor调用了超级父类AbstractCacheInvoker提供的缓存对象Cache的基本方法doPut、doGet、doEvict等方法来缓存数据。
