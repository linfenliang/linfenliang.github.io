---
layout: post
title:  "spring基于redis的注解式cache"
subtitle: "spring boot annotation cache redis"
date:   2017-07-26
header-img: "img/post-bg-tech.jpg"
tags:
    - 技术学习 spring-boot cache redis
categories: study
catalog:  true
---
# spring cache 简介

在spring3.1开始，spring框架提供了显示的缓存支持，与对事务的支持类似，缓存抽象允许我们在保证对代码影响最小的情况下能够以一致的方法使用不同的工具或框架做缓存。
自行spring4.1开始，缓存抽象开始显著的提升了对JSR-107 annotations 注解式缓存以及其他自定义选项的支持。
以上是官网给出的介绍，以我的理解就是：采用缓存式注解方便多了，可以自行指定是用guava还是ehcache还是用redis，只需要配置缓存的框架就可以了，
spring对这些缓存都做了继承，大大方便了使用开发（虽然不建议但确实是让很多人在可以不怎么深入了解缓存框架的情况下就可以使用）

# 使用spring annotation cache的特性

如上简介所描述的，非常便捷的集成、开发

spring cache的特性：

应用在Java的方法上，减少方法体的执行次数，就是每次针对应用了缓存的Java的方法进行调用时，spring都会去检查该方法是否有缓存，如果有，则从缓存中取出结果，如果没有，则执行Java方法并将该方法缓存，并返回用户结果，下次再调用该方法时，直接从缓存中取出来该方法的执行结果返回给用户即可。

这样，一些性能开销较大的Java方法（比如查询数据库），就可以只运行一次，以后如果调用相同的方法，则直接返回结果即可（一般情况下前提是传入的参数都得相同，如果传入的参数不同，还想借助缓存，那么需要自己设定一些缓存读取写入的判断规则）

缓存的逻辑对调用者是透明的。

其他的缓存相关的操作如update与remove操作等在缓存的数据发生了变化时对缓存做的处理，也是很有用的。

就像其他阿德Spring框架一样，缓存服务被设计成了抽象类，而非接口，spring cache需要借助实际的存储框架来缓存数据，这时候，抽象类的价值就体现出来了，开发人员可以直接写缓存数据的逻辑，不用关心缓存框架（和上面简介一个意思），抽象类主要是两个`org.springframework.cache.Cache`和`org.springframework.cache.CacheManager`

spring cache内置的已经实现了的缓存有JDK `java.util.concurrent.ConcurrentMap` based caches, `Ehcache 2.x`, `Gemfire cache`, `Caffeine`, `Guava caches` and JSR-107 compliant caches (e.g. `Ehcache 3.x`).


注意：spring 的缓存抽象并没有对多线程与多进程环境做处理，这个一般是缓存框架实现的。如果你有多进程环境下使用的场景（如多节点），你需要据此配置你的缓存框架（如redis），基于你的多节点使用案例，相同的数据拷贝应该足够了，如果你需要在你的多节点部署的环境下修改数据（修改后需要数据同步），则需要启用相应的传播机制（缓存数据同步机制，一般分布式缓存都已实现如redis，如果采用Map之类的缓存，就麻烦些了）。


使用缓存抽象类时需要注意两点：
1. 缓存声明：确定缓存的方法与策略
2. 缓存配置：缓存数据读取写入的缓存提供方


注意一点：缓存与缓冲的区别（cache vs buffer）

简单来说，Buffer传统上是用来解决在一个速度较快的实体与一个速度较慢的实体之间进行的临时数据存储，其中较快的部分需要更待其他影响性能的部分，Buffer采用数据阻塞的方式将多个小数据块的多次转移变为一个大数据块的一次转移，不管数据从Buffer中读或者写都是一次性的，而且最少其中一方对Buffer是可见的。

而缓存Cache被定义为不可见的，当触发缓存动作时，对使用缓存的双方系统都是不可见的。缓存一般用来提升性能，他被设计成允许可以在不同的时间非常快的读取到相同的数据

详细可以参考维基百科：[Buffer vs. Cache][db10b375]

  [db10b375]: https://en.wikipedia.org/wiki/Cache_(computing)#Buffer_vs._cache "Buffer vs. Cache"


# 使用缓存的项目需求

在链四方项目中，由于需要与很多第三方平台进行数据交互，在数据交互的过程中，需要大量的数据有效性验证，很多验证需要数据库支持，比如订单重复、车辆、人员等信息是否存在，是否有效等，都需要大量的查询数据库操作，目前数据交互量比较大，白天可以达到每秒150-300的并发请求，数据库服务器压力上来了。

# spring boot annotation + redis具体的使用

## 说明：

本工程是基于Maven的Spring-boot工程，详细源码都放到github上面了，

可查看github [spring-boot][261bb2e4]

  [261bb2e4]: https://github.com/linfenliang/springboot "spring-boot"


## 引入maven依赖：

    ```
    <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-cache</artifactId>
		</dependency>
  ```

## 配置application.properties

    ```
    spring.cache.type=REDIS
    spring.redis.host=localhost
    spring.redis.port=6379
    spring.redis.database=0
    ```

## 配置Redis支持以及主键生成策略

    ```
    package com.springboot.data.cache;

    import java.lang.reflect.Method;

    import org.springframework.cache.CacheManager;
    import org.springframework.cache.annotation.CachingConfigurerSupport;
    import org.springframework.cache.annotation.EnableCaching;
    import org.springframework.cache.interceptor.KeyGenerator;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.data.redis.cache.RedisCacheManager;
    import org.springframework.data.redis.connection.RedisConnectionFactory;
    import org.springframework.data.redis.core.RedisTemplate;

    /**
    *
    * @Author linfenliang
    * @Version 1.0
    * @Date 2017年6月29日
    */
    @Configuration
    @EnableCaching
    public class RedisConfig extends CachingConfigurerSupport {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory cf) {
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(cf);
    return redisTemplate;
    }

    @Bean
    public CacheManager cacheManager(RedisTemplate<String, Object> redisTemplate) {
    RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate);
    cacheManager.setDefaultExpiration(30);
    return cacheManager;
    }

    @Bean("keyGenerator")
    public KeyGenerator keyGenerator() {
    return new KeyGenerator() {
      @Override
      public Object generate(Object target, Method method, Object... params) {
        StringBuilder sb = new StringBuilder();
        sb.append(target.getClass().getName());
        sb.append(method.getName());
        for (Object obj : params) {
          sb.append(obj.toString());
        }
        return sb.toString();
      }
    };
    }

    }
    ```

## 编写对应的业务操作

    ```

    package com.springboot.data.cache.service;

    import java.util.ArrayList;
    import java.util.Date;
    import java.util.Iterator;
    import java.util.List;

    import org.apache.commons.lang3.RandomStringUtils;
    import org.apache.commons.lang3.RandomUtils;
    import org.apache.commons.lang3.StringUtils;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.cache.annotation.CacheConfig;
    import org.springframework.cache.annotation.CacheEvict;
    import org.springframework.cache.annotation.Cacheable;
    import org.springframework.cache.annotation.Caching;
    import org.springframework.stereotype.Service;

    import com.springboot.data.cache.entity.VehicleOnlineData;

    /**
     *
     * @Author linfenliang
     * @Version 1.0
     * @Date 2017年7月24日
     */
    @Service
    @CacheConfig(cacheNames="vehicleOnlineDataService")
    public class VehicleOnlineDataService {
    	private static final Logger logger =  LoggerFactory.getLogger(VehicleOnlineDataService.class);

    	static List<VehicleOnlineData> list = new ArrayList<>();
    	static{
    		for (int i = 0; i < 10; i++) {
    			VehicleOnlineData v = new VehicleOnlineData();
    			v.setId(RandomStringUtils.randomAlphabetic(32));
    			v.setVehicleNo("京HM"+RandomStringUtils.randomNumeric(4));
    			v.setCreateDate(new Date());
    			v.setGatherTime(new Date());
    			v.setAttitude(RandomUtils.nextInt(100, 300));
    			list.add(v);
    		}
    	}
    	@Cacheable(value="vehicleOnlineDataList")
    	public List<VehicleOnlineData> list(){
    		logger.debug("list");
    		return list;
    	}
    	@Cacheable(value="vehicleOnlineData")
    	public VehicleOnlineData get(VehicleOnlineData entity){
    		logger.debug("get");
    		for(VehicleOnlineData v:list){
    			if(StringUtils.equals(v.getId(), entity.getId())){
    				return v;
    			}
    		}
    		return null;
    	}
    	@Caching(evict={@CacheEvict(value="vehicleOnlineData",allEntries=true),@CacheEvict(value="vehicleOnlineDataList",allEntries=true)})
    	public VehicleOnlineData update(VehicleOnlineData entity){
    		logger.debug("update");
    		for(int index=0;index<VehicleOnlineDataService.list.size();index++){
    			if(StringUtils.equals(VehicleOnlineDataService.list.get(index).getId(), entity.getId())){
    				VehicleOnlineDataService.list.set(index, entity);
    				return VehicleOnlineDataService.list.get(index);
    			}
    		}
    		return null;
    	}
    //	@CachePut(value="vehicleOnlineDataList",keyGenerator = "keyGenerator")
    	@CacheEvict(value="vehicleOnlineDataList",allEntries=true)
    	public void add(VehicleOnlineData entity){
    		logger.debug("add");
    		list.add(entity);
    		list.forEach(v->{
    			System.out.println(v);
    		});
    	}


    	@Caching(evict={@CacheEvict(value="vehicleOnlineData",allEntries=true),@CacheEvict(value="vehicleOnlineDataList",allEntries=true)})
    	public void del(String id){
    		logger.debug("del");
    		Iterator<VehicleOnlineData> iter = VehicleOnlineDataService.list.iterator();
    		while(iter.hasNext()){
    			VehicleOnlineData v = iter.next();
    			if(StringUtils.equals(v.getId(), id)){
    				iter.remove();
    			}
    		}
    	}


    }
    ```

## spring-boot项目启动类

    ```
    package com.springboot.data.cache;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.cache.annotation.EnableCaching;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

    /**
     *
     * @Author linfenliang
     * @Version 1.0
     * @Date 2017年7月24日
     */
    @EnableCaching
    @Configuration
    @EnableAutoConfiguration
    @SpringBootApplication
    public class CacheApp extends WebMvcConfigurerAdapter {
    	public static void main(String[] args) throws Exception {
    		SpringApplication.run(CacheApp.class, args);
    	}

    }
    ```

## 编写单元测试代码

    ```
    package com.springboot.data.cache;

    import java.util.Date;
    import java.util.List;

    import org.apache.commons.lang3.RandomStringUtils;
    import org.apache.commons.lang3.RandomUtils;
    import org.junit.Before;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

    import com.springboot.data.cache.entity.VehicleOnlineData;
    import com.springboot.data.cache.service.VehicleOnlineDataService;

    /**
     *
     * @Author linfenliang
     * @Version 1.0
     * @Date 2017年7月24日
     */
    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringBootTest(classes=CacheApp.class)
    public class CacheRedisTest {
    	private static final Logger logger = LoggerFactory.getLogger(CacheRedisTest.class);
    	@Autowired
    	private VehicleOnlineDataService service;

    	String id = null;
    	@Before
    	public void listTest(){
    		if(id==null){
    			List<VehicleOnlineData> list = service.list();
    			list.forEach(v->{
    				logger.debug(v.toString());
    			});
    			id = list.get(0).getId();
    		}
    	}
    	@Test
    	public void upateTest(){
    		logger.debug("第一次根据ID：{}取数据{}",id,service.get(new VehicleOnlineData(id)).toString());
    		logger.debug("第二次根据ID：{}取数据{}",id,service.get(new VehicleOnlineData(id)).toString());
    		VehicleOnlineData v = new VehicleOnlineData();
    		v.setId(id);
    		v.setVehicleNo("测试修改后的车辆");
    		VehicleOnlineData vv = service.update(v);
    		logger.debug("{}",vv);
    		logger.debug("第三次根据ID：{}取数据{}",id,service.get(new VehicleOnlineData(id)).toString());
    		logger.debug("修改记录后获取集合数据：");
    		List<VehicleOnlineData> list = service.list();
    		list.forEach(e->{
    			logger.debug(e.toString());
    		});

    		VehicleOnlineData entity = new VehicleOnlineData();
    		entity.setId(RandomStringUtils.randomAlphabetic(32));
    		entity.setVehicleNo("京HM"+RandomStringUtils.randomNumeric(4));
    		entity.setCreateDate(new Date());
    		entity.setGatherTime(new Date());
    		entity.setAttitude(RandomUtils.nextInt(100, 300));
    		logger.debug("新增一条记录：{}",entity);
    		service.add(entity);
    		logger.debug("新增记录后获取集合数据：");
    		list = service.list();
    		list.forEach(e->{
    			logger.debug(e.toString());
    		});


    //		service.del(id);
    //		list = service.list();
    //		list.forEach(e->{
    //			logger.debug(e.toString());
    //		});
    	}

    }

    ```

## 效果

redis中实际存储效果：

```
    172.17.0.1:6379> keys *
    1) "vehicleOnlineData~keys"
    2) "\xac\xed\x00\x05t\x00\xd9com.springboot.data.cache.service.VehicleOnlineDataServicegetVehicleOnlineData[id=OrJmJEVcItahupVjzfdgEAQwzEcAJlIT,vehicleNo=<null>,gatherTime=<null>,latitude=<null>,longitude=<null>,attitude=<null>,createDate=<null>]"
    3) "vehicleOnlineDataList~keys"
    4) "\xac\xed\x00\x05t\x00>com.springboot.data.cache.service.VehicleOnlineDataServicelist"
```


![    缓存实现效果](/img/post-images/2017-07-26/redis-spring-boot-cache-result.jpeg)

解读：
- 第一次取到数据前调用service方法执行了一次get操作
- 第二次再按照同样的条件取数据时，由于有`@Cacheable`注解直接从缓存中取，不再调用get方法
- 然后执行update方法，修改之前的那一条记录，在update方法上加了`@CacheEvict`注解对相关联缓存中的集合以及对象做了清空处理，然后再次取数据，发现执行了get方法，表明evict注解生效了
- 最后获取集合信息，发现集合也被更新了

注意：由于我自定义了主键生成策略keyGenerator，在做缓存时按照我的自定义策略执行，本来在做update操作时，我应该是更新缓存中的数据，而非清空，但是在更新缓存的注解中需要我指明对应的key，而通过redis中的数据可以看到，我的key实际上是一个对象的序列化后得到的，这样，我就很难明确的将其定义出来（如果自定义缓存key是可以的，这时候不要用对象信息，而用对象对应的属性如ID信息，作为key来缓存），所以我这里没有用`@CachePut`注解。

其他注解及作用：
- @Cacheable triggers cache population
- @CacheEvict triggers cache eviction
- @CachePut updates the cache without interfering with the method execution
- @Caching regroups multiple cache operations to be applied on a method
- @CacheConfig shares some common cache-related settings at class-level

本例子除了CachePut未测试外，其他都采用了，其他的条件缓存、同步缓存，以及缓存用的Spel可以参考文末的官网文档链接。

另外一点是缓存全局设置<类设置<method设置

# 缓存的实现原理

缓存一个指定的item操作等同于`get-if-not-found-then-proceed-and-put-eventually`代码块直接与缓存进行交互（实际上，缓存框架就是这么干的），代码块中应是：无锁模式下多线程并发的去加载相同的item，与此相同的缓存移除操作（如果多个线程并发的去更新或移除缓存，你获取到的数据可能是过时的数据），一些缓存供应商在这一点上会提供高级的特性（如redis）,这些需要研读缓存供应商提供的文档了。

缓存只能应用于public方法上，官方给出的解释是：

<kbd>
When using proxies, you should apply the cache annotations only to methods with public visibility. If you do annotate protected, private or package-visible methods with these annotations, no error is raised, but the annotated method does not exhibit the configured caching settings. Consider the use of AspectJ (see below) if you need to annotate non-public methods as it changes the bytecode itself.
</kbd>

另外，官方建议缓存注解只放到实际的Class中，而非接口，用接口也不是不可以，这就涉及到了代理问题，spring annotation 只能工作在基于接口的代理中。

还有一点就是缓存时间的设定完全依赖于缓存框架本身，比如redis、guava，而其他如ConcurrentHashMap则不支持。

# spring-annotation的优缺点

# 参考资料

[spring官方文档][b3e67e48]

  [b3e67e48]: http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html "spring官方文档"
