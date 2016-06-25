---
title: SpringMVC源码分析(3) HandlerMapping分析
date: "2015-10-24T20:20:51+08:00"
tags:
    - spring mvc
url: 2015/10/22/spring/handlermapping
---

----------

&#160;&#160;&#160;&#160;&#160;&#160;
在前面的篇章中我们已经提及过HandlerMapping接口，HandlerMapping接口提供的方法很简单，就是创建HandlerExecutorChain对象。而在HandlerExecutorChain对象中包含了handler对象和拦截器链对象。所以，我们在分析HandlerMapping过程中，也主要围绕这两点分析。

&#160;&#160;&#160;&#160;&#160;&#160;
由于HandlerMapping提供了最基本的抽象实现AbstractHandlerMapping，所以我们从AbstractHandlerMapping开始分析。


----------


### 1.AbstractHandlerMapping 分析

&#160;&#160;&#160;&#160;&#160;&#160;
关于AbstractHandlerMapping的分析，分为以下三个步骤：
1. 属性域说明
2. 获取拦截器
3. 创建HandlerExecutorChain对象

#### 1.1 属性域说明

```
// 配置默认的handler对象，在没有获取到handler的情况下使用
private Object defaultHandler;

// request请求路径解析辅助类
private UrlPathHelper urlPathHelper = new UrlPathHelper();

// 请求路径匹配辅助类，用于匹配哪些路径可以被MappedInterceptor来拦截处理
private PathMatcher pathMatcher = new AntPathMatcher();

// 子类或配置文件可以配置拦截器，这里的Object类型为：
// HandlerInterceptor, WebRequestInterceptor 和 MappedInterceptor
private final List<Object> interceptors = new ArrayList<Object>();

// 请求真正被适配到的拦截器(包含了HandlerInterceptor, WebRequestInterceptor)
private final List<HandlerInterceptor> adaptedInterceptors = new ArrayList<HandlerInterceptor>();

// 需要进行路径匹配的MappingInterceptor
private final List<MappedInterceptor> mappedInterceptors = new ArrayList<MappedInterceptor>();
```

#### 1.2 获取拦截器

&#160;&#160;&#160;&#160;&#160;&#160;
由于AbstractHandlerMapping继承了WebApplicationObjectSupport类，所以在Spring容器启动完成后会调用initApplicationContext()方法。

```
protected void initApplicationContext() throws BeansException {
    
    // 钩子方法，由子类重写，子类可以添加新的拦截器，
    // interceptors是由配置文件设置的，在Bean注入时调用setInterceptors(Object[] interceptors)注入。
	extendInterceptors(this.interceptors);
	
	// 检测容器中注入的MappedInterceptor，并添加到mappedInterceptors中。
	detectMappedInterceptors(this.mappedInterceptors);
	
	//初始化拦截器，也就对interceptors中的拦截器分类存放，
	// MappedInterceptor拦截器存放到mappedInterceptors中去，
	// HandlerInterceptor、WebRequestInterceptor拦截器存放到adaptedInterceptors中去。
	initInterceptors();
}
```

&#160;&#160;&#160;&#160;&#160;&#160;
下面是initInterceptors()的具体实现。

```
protected void initInterceptors() {
	if (!this.interceptors.isEmpty()) {
		for (int i = 0; i < this.interceptors.size(); i++) {
			Object interceptor = this.interceptors.get(i);
			if (interceptor == null) {
				throw new IllegalArgumentException();
			}
			// 存在到mappedInterceptors
			if (interceptor instanceof MappedInterceptor) {
				mappedInterceptors.add((MappedInterceptor) interceptor);
			}
			// 存放到adaptedInterceptors
			else {
				adaptedInterceptors.add(adaptInterceptor(interceptor));
			}
		}
	}
}
protected HandlerInterceptor adaptInterceptor(Object interceptor) {
	if (interceptor instanceof HandlerInterceptor) {
		return (HandlerInterceptor) interceptor;
	}
	// WebRequestInterceptor接口会被适配成HandlerInterceptor接口
	else if (interceptor instanceof WebRequestInterceptor) {
		return new WebRequestHandlerInterceptorAdapter((WebRequestInterceptor) interceptor);
	}
	else {
		throw new IllegalArgumentException("Interceptor type not supported: " + interceptor.getClass().getName());
	}
}
```

#### 1.3 创建HandlerExecutorChain对象

很清楚，创建HandlerExecutorChain对象是接口方法 HandlerExecutionChain getHandler(HttpServletRequest request)，下面直接分析代码。

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 这是一个抽象方法，具体的handler对象，子类去提供，因为抽象类并不知道是请求需要哪种handler
	Object handler = getHandlerInternal(request);
	// 如果没有获取到，则获取默认配置的handler
	if (handler == null) {
		handler = getDefaultHandler();
	}
	// 如果没有默认配置，则返回null
	if (handler == null) {
		return null;
	}
	// 如果handler是String类型，则当作beanName从容器器获取
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = getApplicationContext().getBean(handlerName);
	}
	// 返回HandlerExecutionChain对象
	return getHandlerExecutionChain(handler, request);
}

protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    // 如果handler本身是HandlerExecutionChain类型则不用创建，
    // 否则创建一个HandlerExecutionChain对象并设置handler
	HandlerExecutionChain chain =
		(handler instanceof HandlerExecutionChain) ?
			(HandlerExecutionChain) handler : new HandlerExecutionChain(handler);
    
    // 设置拦截器
	chain.addInterceptors(getAdaptedInterceptors());

    // 根据请求路径，匹配哪些MappedInterceptor需要被拦截
	String lookupPath = urlPathHelper.getLookupPathForRequest(request);
	for (MappedInterceptor mappedInterceptor : mappedInterceptors) {
		if (mappedInterceptor.matches(lookupPath, pathMatcher)) {
			chain.addInterceptor(mappedInterceptor.getInterceptor());
		}
	}
    
    // 返回最终的HandlerExecutionChain对象
	return chain;
}
```

&#160;&#160;&#160;&#160;&#160;&#160;
上面通过三个方面分析了AbstractHandlerMapping，我们明白了拦截器是怎么获取的，我们也明白了HandlerExecutionChain的创建过程。AbstractHandlerMapping抽象类几乎做完了HandlerMapping接口的所有事情，只是handler对象的获取留给了子类去自由发挥。接下来，我们需要看看子类都是如何创建handler对象的


----------

&#160;&#160;&#160;&#160;&#160;&#160;
我们知道如果在配置文件中没有配置HandlerMapping，那么，SpringMVC会根据DispatcherServlet.properties中的配置来获取HandlerMapping的实现类。在DispatcherServlet.properties中配置了两个HandlerMapping实现类BeanNameUrlHandlerMapping和DefaultAnnotationHandlerMapping。接下来我们根据这两个类，来讲解handler的获取。

&#160;&#160;&#160;&#160;&#160;&#160;
BeanNameUrlHandlerMapping和DefaultAnnotationHandlerMapping都是基于URL的HandlerMapping，而且继承关系都是： AbstractDetectingUrlHandlerMapping --> AbstractUrlHandlerMapping。 所以我们接下来分析基于URL的HandlerMapping。

### 2.AbstractUrlHandlerMapping分析

#### 2.1 基于URL匹配handler的过程

```
protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
    // 获取请求路径
	String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
	// 查找handler对象
	Object handler = lookupHandler(lookupPath, request);
	if (handler == null) {
		Object rawHandler = null;
		// 处理根路径，获取配置的rootHandler
		if ("/".equals(lookupPath)) {
			rawHandler = getRootHandler();
		}
		if (rawHandler == null) {
			rawHandler = getDefaultHandler();
		}
		if (rawHandler != null) {
		    // 如果是String类型，则从容器中根据beanName获取
			if (rawHandler instanceof String) {
				String handlerName = (String) rawHandler;
				rawHandler = getApplicationContext().getBean(handlerName);
			}
			// 钩子方法子类去校验
			validateHandler(rawHandler, request);
			// 使用配置的rawHandler，来创建HandlerExecutionChain，把HandlerExecutionChain当作handler
			handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
		}
	}
	// 返回handler
	return handler;
}

protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
	// 直接使用路径获取handler，那么，handlerMap中的数据是怎么来的？ 后面再分析。
	Object handler = this.handlerMap.get(urlPath);
	if (handler != null) {
	    // 如果是String类型，则从容器中根据beanName获取
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = getApplicationContext().getBean(handlerName);
		}
		// 钩子方法子类去校验
		validateHandler(handler, request);
		// 使用配置的rawHandler，来创建HandlerExecutionChain，把HandlerExecutionChain当作handler
		return buildPathExposingHandler(handler, urlPath, urlPath, null);
	}
    // 模糊匹配代码省略...
	// 如果最终没有匹配到，则放回null
	return null;
}
```

#### 2.2 handler注册

&#160;&#160;&#160;&#160;&#160;&#160;
在上面分析过程中，我们不禁会问handlerMap中数据是怎么来的呢？ 这个问题也就是我们需要讲解的handler注册。

```
protected void registerHandler(String[] urlPaths, String beanName) throws BeansException, IllegalStateException {
	for (String urlPath : urlPaths) {
		registerHandler(urlPath, beanName);
	}
}

protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {

	Object resolvedHandler = handler;
	
	// 是否设置懒加载
	if (!this.lazyInitHandlers && handler instanceof String) {
		String handlerName = (String) handler;
		if (getApplicationContext().isSingleton(handlerName)) {
			resolvedHandler = getApplicationContext().getBean(handlerName);
		}
	}

    // 如果可以直接获取到，则判断和传入的resolvedHandler是否一致
	Object mappedHandler = this.handlerMap.get(urlPath);
	if (mappedHandler != null) {
		if (mappedHandler != resolvedHandler) {
			throw new IllegalStateException();
		}
	}
	else {
	    // 设置rootlHandler和defaultHandler
		if (urlPath.equals("/")) {
			setRootHandler(resolvedHandler);
		}
		else if (urlPath.equals("/*")) {
			setDefaultHandler(resolvedHandler);
		}
		else {
		    // 存放到handlerMap中
			this.handlerMap.put(urlPath, resolvedHandler);
		}
	}
}
```
&#160;&#160;&#160;&#160;&#160;&#160;
在AbstractUrlHandlerMapping中，并没有调用registerHandler()方法，也就是说，只是提供了方法。而方法修饰符为protected，所以注册handler肯定是在子类进行的。

&#160;&#160;&#160;&#160;&#160;&#160;
AbstractUrlHandlerMapping的实现很简单，可以总结为以下两点：
1. 根据请求路径urlPath从handlerMap中获取handler，首先直接匹配，然后模糊匹配。
2. 提供注册handler的方法registerHandler()供子类来注册handler，存放到handlerMap中去。

&#160;&#160;&#160;&#160;&#160;&#160;
接下来可以分析AbstractDetectingUrlHandlerMapping了，从名称上我们就可以看出，这个抽象类是用来检测获取Url的。

----------

### 3.AbstractDetectingUrlHandlerMapping分析

&#160;&#160;&#160;&#160;&#160;&#160;
AbstractDetectingUrlHandlerMapping注册handler是通过重写initApplicationContext()来进行的，在Spring容器启动好后会调用detectHandlers()方法，AbstractDetectingUrlHandlerMapping就开始检测所有的Bean，并获取URL。

```
protected void detectHandlers() throws BeansException {
	// 获取容器中所有beanNames(配置detectHandlersInAncestorContexts可以从父容器检测，默认为false)
	String[] beanNames = (this.detectHandlersInAncestorContexts ?
			BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
			getApplicationContext().getBeanNamesForType(Object.class));

	for (String beanName : beanNames) {
	    // 检测每一个beanName中的所有url(抽象方法，子类去实现如何查找urls)
		String[] urls = determineUrlsForHandler(beanName);
		if (!ObjectUtils.isEmpty(urls)) {
			// 注册handler
			registerHandler(urls, beanName);
		}
	}
}
```

&#160;&#160;&#160;&#160;&#160;&#160;
AbstractDetectingUrlHandlerMapping调用了registerHandler()来注册handler，最终都存放到AbstractUrlHandlerMapping#handlerMap中去。剩下的事情就是子类如何实现抽象方法determineUrlsForHandler(beanName)了。

### 4. BeanNameUrlHandlerMapping分析

&#160;&#160;&#160;&#160;&#160;&#160;
BeanNameUrlHandlerMapping是基于beanName来获取Url，beanName必须以/开头。代码很简单，直接看代码了。

```
protected String[] determineUrlsForHandler(String beanName) {
	List<String> urls = new ArrayList<String>();
	// beanName需要以/开头
	if (beanName.startsWith("/")) {
		urls.add(beanName);
	}
	// 获取该beanName的别名，别名也需要以/开头
	String[] aliases = getApplicationContext().getAliases(beanName);
	for (String alias : aliases) {
		if (alias.startsWith("/")) {
			urls.add(alias);
		}
	}
	// 把匹配的路径全部返回
	return StringUtils.toStringArray(urls);
}
```

### 4. DefaultAnnotationHandlerMapping分析

&#160;&#160;&#160;&#160;&#160;&#160;
虽然DefaultAnnotationHandlerMapping在3.2版本以后就设置成了@Deprecated，也就是不建议使用这个了(RequestMappingHandlerMapping代替了它)。但我现在使用的4.0.2版本中DispatcherServlet.properties默认提供的还是DefaultAnnotationHandlerMapping，所以在我们不主动切换HandlerMapping的大多数情况下，我们还是会使用DefaultAnnotationHandlerMapping。

&#160;&#160;&#160;&#160;&#160;&#160;
DefaultAnnotationHandlerMapping的功能是从配置了@Controller、@RequestMapping的类中回去请求url。

```
protected String[] determineUrlsForHandler(String beanName) {
	ApplicationContext context = getApplicationContext();
	Class<?> handlerType = context.getType(beanName);
	// 获取类型注解@RequestMapping
	RequestMapping mapping = context.findAnnotationOnBean(beanName, RequestMapping.class);
	if (mapping != null) {
		// 缓存类型上的@RequestMapping注解
		this.cachedMappings.put(handlerType, mapping);
		Set<String> urls = new LinkedHashSet<String>();
		// 获取@RequestMapping value值
		String[] typeLevelPatterns = mapping.value();
		if (typeLevelPatterns.length > 0) {
			// 获取方法上的@RequestMapping
			String[] methodLevelPatterns = determineUrlsForHandlerMethods(handlerType, true);
			// 结合类型上的@RequestMapping路径和方法上的@RequestMapping路径
			for (String typeLevelPattern : typeLevelPatterns) {
			    // 类型上@RequestMapping value值可以不用/开头，这里会检测
				if (!typeLevelPattern.startsWith("/")) {
					typeLevelPattern = "/" + typeLevelPattern;
				}
				// 方法上是否有空值的@RequestMapping
				boolean hasEmptyMethodLevelMappings = false;
				for (String methodLevelPattern : methodLevelPatterns) {
					if (methodLevelPattern == null) {
						hasEmptyMethodLevelMappings = true;
					}
					else {
					    // 获取结合后的url
						String combinedPattern = getPathMatcher().combine(typeLevelPattern, methodLevelPattern);
						addUrlsForPath(urls, combinedPattern);
					}
				}
				// 直接使用类型上的@RequestMapping，但需要时Controller对象类型
				if (hasEmptyMethodLevelMappings ||
						org.springframework.web.servlet.mvc.Controller.class.isAssignableFrom(handlerType)) {
					addUrlsForPath(urls, typeLevelPattern);
				}
			}
			return StringUtils.toStringArray(urls);
		}
		else {
			// 获取方法上的@RequestMapping
			return determineUrlsForHandlerMethods(handlerType, false);
		}
	}
	else if (AnnotationUtils.findAnnotation(handlerType, Controller.class) != null) {
		// 检测@Controller中的@RequestMapping
		return determineUrlsForHandlerMethods(handlerType, false);
	}
	else {
		return null;
	}
}
```

&#160;&#160;&#160;&#160;&#160;&#160;
上面的方法有点复杂，就上面的方法我们归纳一下：
1. 检测类@RequestMapping和@Controller，@RequestMapping优先。如果存在@RequestMapping，则不会检测@Controller。
2. 如果类@RequestMapping存在，则和方法@RequestMapping组合在一起。如果存在方法@RequestMapping存在没有设置value，则以类@RequestMapping路径设置。
3. 如果没有类@RequestMapping，则检测类@Controller。注意@Controller(value="name")，@Controller中设置的value并不会影响请求url。

&#160;&#160;&#160;&#160;&#160;&#160;
下面我们在看看方法上的@RequestMapping是如何解析的。

```
protected String[] determineUrlsForHandlerMethods(Class<?> handlerType, final boolean hasTypeLevelMapping) {
    // 这是一个空方法，返回null，提供给子类自定义
	String[] subclassResult = determineUrlsForHandlerMethods(handlerType);
	if (subclassResult != null) {
		return subclassResult;
	}

	final Set<String> urls = new LinkedHashSet<String>();
	Set<Class<?>> handlerTypes = new LinkedHashSet<Class<?>>();
	// 添加handler以及所有接口
	handlerTypes.add(handlerType);
	handlerTypes.addAll(Arrays.asList(handlerType.getInterfaces()));
	// 遍历handler及其接口方法
	for (Class<?> currentHandlerType : handlerTypes) {
	    // 递归调用类以及父类方法（接口及父接口），排除bridge方法(编译器生成的方法)和Object方法
		ReflectionUtils.doWithMethods(currentHandlerType, new ReflectionUtils.MethodCallback() {
			@Override
			public void doWith(Method method) {
			    // 获取方法@RequestMapping
				RequestMapping mapping = AnnotationUtils.findAnnotation(method, RequestMapping.class);
				if (mapping != null) {
				    // 获取路径值
					String[] mappedPatterns = mapping.value();
					if (mappedPatterns.length > 0) {
						for (String mappedPattern : mappedPatterns) {
						    //如果存在类@RequestMapping，则方法@RequestMapping必须以/开头
							if (!hasTypeLevelMapping && !mappedPattern.startsWith("/")) {
								mappedPattern = "/" + mappedPattern;
							}
							addUrlsForPath(urls, mappedPattern);
						}
					}
					// 添加null
					else if (hasTypeLevelMapping) {
						urls.add(null);
					}
				}
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
	}
	return StringUtils.toStringArray(urls);
}	
```

&#160;&#160;&#160;&#160;&#160;&#160;
关于方法@RequestMapping我们也总结一下：
1. handler类，父类，接口，父接口中所有设置有@RequestMapping注解的方法都会处理。
2. 如果有类@RequestMapping，则方法@RequestMapping设置value时，必须以/开头，否则添加到路径集合中。如果是@Controller，则方法@RequestMapping可以不用/开头设置value。


### 总结

&#160;&#160;&#160;&#160;&#160;&#160;
1. HandlerMapping接口的功能是用来创建HandlerExecutionChain对象。而在创建过程中需要获取hander对象和拦截器。
2. 拦截器的获取是通过属性interceptors在配置文件中注入的。
3. handler对象的获取是根据请求url，从handlerMap中匹配。
4. handlerMap中的数据，是在Spring容器启动完成后进行设置。设置方式由子类完成。
5. 明白了BeanNameUrlHandlerMapping的设置。
6. 明白了@Controller、@RequestMapping注解配置的url是如何被读取的。

