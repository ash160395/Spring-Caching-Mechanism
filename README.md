# Spring-Caching-Mechanism
Explains caching mechanism adopted by Spring Framework

@Cacheable annotation has following metadata:
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
