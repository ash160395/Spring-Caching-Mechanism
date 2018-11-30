# Spring-Caching-Mechanism
Explains caching mechanism adopted by Spring Framework


# @Cacheable 
This annotation has following metadata and attributes:

1. @Target -- METHOD,TYPE
2. @Inherited
3. @Retention -- RUNTIME

Attributes:

1. String[] cacheNames() default {};

    --> Names of the caches in which method invocation results are stored.
    
2. String[] value() default {};

    --> Alias for cacheNames
    
3. String key() default "";

    --> SpEL for computing the key dynamically.
        Default is {""}, meaning all method parameters are considered as a key, unless a custom #keyGenerator has been configured.
        
4. String keyGenerator() default "";

    --> Bean name of custom implementation of org.springframework.cache.interceptor.KeyGenerator
    
5. String condition() default "";

    --> SpEL expression used for making the method caching conditional.
	    Default is {""}, meaning the method result is always cached.


# SpringCacheAnnotationParser
Strategy implementation for parsing Spring's Caching, Cacheable, CacheEvict, and CachePut annotations.
implements CacheAnnotationParser

	@Override
	@Nullable
	public Collection<CacheOperation> parseCacheAnnotations(Class<?> type) {
		DefaultCacheConfig defaultConfig = getDefaultCacheConfig(type);
		return parseCacheAnnotations(defaultConfig, type);
	}

	@Override
	@Nullable
	public Collection<CacheOperation> parseCacheAnnotations(Method method) {
		DefaultCacheConfig defaultConfig = getDefaultCacheConfig(method.getDeclaringClass());
		return parseCacheAnnotations(defaultConfig, method);

# AbstractFallbackCacheOperationSource 
Abstract implementation of CacheOperationSource that caches attributes for methods and implements a fallback policy:
1. specific target method
2. target class
3. declaring method
4. declaring class/interface.

	
Canonical value held in cache to indicate no caching attribute was found for this method and we don't need to look again.
	 
	private static final Collection<CacheOperation> NULL_CACHING_ATTRIBUTE = Collections.emptyList();

Cache of CacheOperations, keyed by method on a specific target class. As this base class is not marked Serializable, the cache will be recreated after serialization - provided that the concrete subclass is Serializable.

	private final Map<Object, Collection<CacheOperation>> attributeCache = new ConcurrentHashMap<>(1024);
	
Determine a cache key for the given method and target class. Must not produce same key for overloaded methods.
Must produce same key for different instances of the same method.
@param method the method (never null)
@param targetClass the target class (may be null)
@return the cache key (never null)

	protected Object getCacheKey(Method method, @Nullable Class<?> targetClass) {
		return new MethodClassKey(method, targetClass);
	}
	
Determine the caching attribute for this method invocation.
Defaults to the class's caching attribute if no method attribute is found.
@return CacheOperation for this method, or null if the method is not cacheable

	@Override
	@Nullable
	public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

	Object cacheKey = getCacheKey(method, targetClass);
	Collection<CacheOperation> cached = this.attributeCache.get(cacheKey);

	if (cached != null) {
		return (cached != NULL_CACHING_ATTRIBUTE ? cached : null);
	}
	else {
		Collection<CacheOperation> cacheOps = computeCacheOperations(method, targetClass);
		if (cacheOps != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Adding cacheable method '" + method.getName() + "' with attribute: " + 						cacheOps);
			}
			this.attributeCache.put(cacheKey, cacheOps);
		}
		else {
			this.attributeCache.put(cacheKey, NULL_CACHING_ATTRIBUTE);
		}
		return cacheOps;
	}
	}

	@Nullable
	private Collection<CacheOperation> computeCacheOperations(Method method, @Nullable Class<?> targetClass) {
		// Don't allow no-public methods as required.
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

	// The method may be on an interface, but we need attributes from the target class.
	// If the target class is null, the method will be unchanged.
	Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

	// First try is the method in the target class.
	Collection<CacheOperation> opDef = findCacheOperations(specificMethod);
	if (opDef != null) {
		return opDef;
	}

	// Second try is the caching operation on the target class.
	opDef = findCacheOperations(specificMethod.getDeclaringClass());
	if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
		return opDef;
	}

	if (specificMethod != method) {
		// Fallback is to look at the original method.
		opDef = findCacheOperations(method);
		if (opDef != null) {
			return opDef;
		}
		// Last fallback is the class of the original method.
		opDef = findCacheOperations(method.getDeclaringClass());
		if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
			return opDef;
		}
	}

	return null;
	}


Subclasses need to implement this to return the caching attribute for the given class, if any.
@param clazz the class to retrieve the attribute for
@return all caching attribute associated with this class, or null if none

	@Nullable
	protected abstract Collection<CacheOperation> findCacheOperations(Class<?> clazz);

Subclasses need to implement this to return the caching attribute for the given method, if any.
@param method the method to retrieve the attribute for
@return all caching attribute associated with this method, or null if none

	@Nullable
	protected abstract Collection<CacheOperation> findCacheOperations(Method method);

Should only public methods be allowed to have caching semantics?
The default implementation returns false.

	protected boolean allowPublicMethodsOnly() {
		return false;
	}


# AnnotationCacheOperationSource
Implementation of org.springframework.cache.interceptor.CacheOperationSource interface for working with caching metadata in annotation format.

This class reads Spring's Cacheable, CachePut and CacheEvict annotations and exposes corresponding caching operation definition to Spring's cache infrastructure. This class may also serve as base class for a custom CacheOperationSource.

extends AbstractFallbackCacheOperationSource

	private final boolean publicMethodsOnly;

	private final Set<CacheAnnotationParser> annotationParsers;
	
	@Override
	@Nullable
	protected Collection<CacheOperation> findCacheOperations(Class<?> clazz) {
		return determineCacheOperations(parser -> parser.parseCacheAnnotations(clazz));
	}

	@Override
	@Nullable
	protected Collection<CacheOperation> findCacheOperations(Method method) {
		return determineCacheOperations(parser -> parser.parseCacheAnnotations(method));
	}


Determine the cache operation(s) for the given CacheOperationProvider.
This implementation delegates to configured CacheAnnotationParsers for parsing known annotations into Spring's metadata attribute class.
Can be overridden to support custom annotations that carry caching metadata.
@param provider the cache operation provider to use
@return the configured caching operations, or null if none found
	
	@Nullable
	protected Collection<CacheOperation> determineCacheOperations(CacheOperationProvider provider) {
		Collection<CacheOperation> ops = null;
		for (CacheAnnotationParser annotationParser : this.annotationParsers) {
			Collection<CacheOperation> annOps = provider.getCacheOperations(annotationParser);
			if (annOps != null) {
				if (ops == null) {
					ops = annOps;
				}
				else {
					Collection<CacheOperation> combined = new ArrayList<>(ops.size() + annOps.size());
					combined.addAll(ops);
					combined.addAll(annOps);
					ops = combined;
				}
			}
		}
		return ops;
	}
	

Callback interface providing CacheOperation instance(s) based on a given CacheAnnotationParser.
	 
	@FunctionalInterface
	protected interface CacheOperationProvider {

		/**
		 * Return the {@link CacheOperation} instance(s) provided by the specified parser.
		 * @param parser the parser to use
		 * @return the cache operations, or {@code null} if none found
		 */
		@Nullable
		Collection<CacheOperation> getCacheOperations(CacheAnnotationParser parser);
	}

# ProxyCachingConfiguration
Configuration class that registers the Spring infrastructure beans necessary to enable proxy-based annotation-driven cache management.

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor() {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.setCacheOperationSources(cacheOperationSource());
		if (this.cacheResolver != null) {
			interceptor.setCacheResolver(this.cacheResolver);
		}
		else if (this.cacheManager != null) {
			interceptor.setCacheManager(this.cacheManager);
		}
		if (this.keyGenerator != null) {
			interceptor.setKeyGenerator(this.keyGenerator);
		}
		if (this.errorHandler != null) {
			interceptor.setErrorHandler(this.errorHandler);
		}
		return interceptor;
	}
