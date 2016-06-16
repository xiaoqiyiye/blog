---
title: SpringCache源码分析(2) @CachePut、@Cacheable、@CacheEvict、@Caching注解
date: "2016-01-23T20:00:00+08:00"
tags:
    - spring cache
url: 2016/01/23/spring-cache-2/
---

&#160;&#160;&#160;&#160;
在上一篇中我们讲解了CacheManager和Cache源码，学会了怎样使用注解进行缓存，但是对于@CachePut、@Cacheable、@CacheEvict这些注解没有提及到，这一篇中我们将对Spring Cache中提供的注解操作做一个详细的说明。
<br>


----------

### @CachePut

&#160;&#160;&#160;&#160;
@CachePut表示需要存放缓存数据，可以在类或方法上进行注解，如果注解在类上，表示这个类的所有方法都使用了@CachePut。这个注解可以设置哪些信息呢？我们直接看源代码，因为源代码能从根本说明一切。

```
/**
 * @CachePut可以注解类和方法，注解类时，表示整个类中的方法都注解了
 */	
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CachePut {

	/**
	 * 定义存放的Cache名称，和下面的cacheNames一样
	 */
	@AliasFor("cacheNames")
	String[] value() default {};

	@AliasFor("value")
	String[] cacheNames() default {};

	/**
     * 为缓存值定义的key，默认为""，表示所有的参数都加入到key的生成中（使用默认的keyGenerator）
	 */
	String key() default "";

	/**
     * 可以使用keyGenerator来自定义key的生成，但是keyGenerator和key是排它性的，也就是说key和keyGenerator只能定义其中一个
	 */
	String keyGenerator() default "";

	/**
     * 指定设置特定的cacheManager，与cacheResolver也是排它性的。
	 * {@link org.springframework.cache.CacheManager}
	 */
	String cacheManager() default "";

	/**
     * 自定义cacheResolver
	 * {@link org.springframework.cache.interceptor.CacheResolver}
	 */
	String cacheResolver() default "";

	/**
     * 定义缓存被存放的条件，只有满足条件的方法返回值才能被存放。默认为""，也意味着方法的结果都可以被缓存。
	 */
	String condition() default "";

	/**
     * unless是在方法调用后判断是否需要进行缓存更新，如果满足unless条件就不缓存。unless具有否决权！
	 */
	String unless() default "";

}
```	


----------


### @Cacheable

&#160;&#160;&#160;&#160;
@Cacheable表示从缓存中取数据，如果缓存中没有数据则执行方法，并将方法返回的结果存入到缓存中，以便下次直接从缓存中获取。@Cacheable可以在类或方法上进行注解，如果注解在类上，表示这个类的所有方法都使用了@Cacheable。由于@Cacheable和@CachePut的定义一样，就不多说明了。

```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {

	@AliasFor("cacheNames")
	String[] value() default {};

	@AliasFor("value")
	String[] cacheNames() default {};
	
	String key() default "";
	
	String keyGenerator() default "";
	
	String cacheManager() default "";
	
	String cacheResolver() default "";
	
	String condition() default "";
	
	String unless() default "";
	
}
```


----------

### @CacheEvict

&#160;&#160;&#160;&#160;
@CacheEvit表示从缓存中删除数据。和@CachePut、@Cacheable的定义基本一样，只有2个参数定义不同，allEntries和beforeInvocation。allEntries表示是否删除指定Cache名称中所有的缓存数据。beforeInvocation表示是否在方法执行前删除缓存数据，因为方法内部可能会出现异常，如果beforeInvocation=false,方法内出现异常，缓存中的数据是不会被删除的。

```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CacheEvict {

	@AliasFor("cacheNames")
	String[] value() default {};

	@AliasFor("value")
	String[] cacheNames() default {};

	String key() default "";

	String keyGenerator() default "";

	String cacheManager() default "";

	String cacheResolver() default "";

	String condition() default "";

	/**
     * 表示是否删除缓存中所有数据，默认为false,只会删除与key关联的那个缓存数据
	 */
	boolean allEntries() default false;

	/**
     * 表示是否在方法调用之前删除数据，默认为false，表示缓存删除操作是在方法调用之后的
	 */
	boolean beforeInvocation() default false;

}
```


----------

### @Caching

&#160;&#160;&#160;&#160;
@Caching是一个组合式的注解，可以组合配置@CachePut、@Cacheable、@CacheEvict集合，也可以在类或方法上进行注解。

```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Caching {

	Cacheable[] cacheable() default {};

	CachePut[] put() default {};

	CacheEvict[] evict() default {};

}
```

----------

&#160;&#160;&#160;&#160;
了解了@CachePut、@Cacheable、@CacheEvict这些注解的详细使用，那么，接下来我有一个疑惑了，这些注解到底是怎么作用到配置的方法上的呢？ 怎么能够说明一个执行方法就使用了这些注解呢？对，这些注解肯定需要被解析成操作对象！那么，@CachePut、@Cacheable、@CacheEvict会被怎么解析呢？ 这些问题我们在下一篇中做详细的分析。


