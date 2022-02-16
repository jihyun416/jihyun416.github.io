---
layout: post
title:  Springboot - redis cache
author: Jihyun
category: springboot
tags:
- springboot
- redis
- elasticache
- cache
date: 2022-02-16 19:00 +0900
---

Redis를 이용하여 자주 호출하는 API에 대해 캐싱을 해볼 것이다.



## 1. build.gradle 의존성 추가

```yaml
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}
```



## 2. application.yml redis 정보 추가

```yaml
spring:
  redis:
    host: 호스트명
    port: 6379
```



## 3. Redis Configuration

```java
package io.willog.marine.config.redis;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jdk8.Jdk8Module;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

@EnableCaching
@Configuration
public class RedisConfig {
    @Value("${spring.redis.host}")
    private String host;
    @Value("${spring.redis.port}")
    private int port;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setHostName(host);
        redisStandaloneConfiguration.setPort(port);
        LettuceConnectionFactory lettuceConnectionFactory =
                new LettuceConnectionFactory(redisStandaloneConfiguration);
        return lettuceConnectionFactory;
    }

    @Primary
    @Bean(name = "cacheManager1Hr")
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        Duration expiration = Duration.ofHours(1);
        RedisCacheManager redisCacheManager =
                RedisCacheManager.builder(redisConnectionFactory)
                        .cacheDefaults(
                                RedisCacheConfiguration.defaultCacheConfig()
                                        .disableCachingNullValues()
                                        .serializeValuesWith(
                                                RedisSerializationContext.SerializationPair.fromSerializer(
                                                        new GenericJackson2JsonRedisSerializer()))
                                        .entryTtl(expiration))
                        .build();
        redisCacheManager.setTransactionAware(false);
        return redisCacheManager;
    }

    @Bean
    public RedisTemplate<?, ?> redisTemplateConfig(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<?, ?> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        return redisTemplate;
    }

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.registerModules(new JavaTimeModule(), new Jdk8Module());
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}

```

- redisConnectionFactory 에서 Connection을 생성한다.
- 1시간이면 캐시를 만료되게 할 cacheManager를 설정한다. (캐시 설정 시 이용)
- redisTemplateConfig 에서 serialize/deserialize 설정을 한다.
- ObjectMapper를 이용한 serialize/deserialize 도 이상이 없도록 설정한다.



## 4. 사용 예시

#### 1) 캐시 생성

```java
@Cacheable(key = "#userSeq", value = "favorite", cacheManager = "cacheManager1Hr")
@GetMapping("/favorite")
public Favorite getFavorite(@RequestParam("userSeq") Long userSeq) {
    return favoriteService.getFavorite(userSeq);
}
```

#### 2) 캐시 삭제

```json
@CacheEvict(key = "#userSeq", value = "favorite")
@PutMapping("/favorite/update")
public Favorite updateFavorite(@RequestParam("userSeq") Long userSeq, FavoriteDTO dto) {
    return favoriteService.updateFavorite(userSeq, dto);
}
```



- 테스트를 해보면 처음 호출 시는 DB를 조회하는 것으로 확인되며, 만료 시간 이내에 다시 호출 시에는 DB를 조회하지 않고 캐시에서 데이터를 불러오는 것을 확인할 수 있다.

- 캐시 삭제가 되도록 하는 API를 호출하고 캐싱하는 API를 호출하면 캐시가 없어져서 다시 DB에서 부터 불러오는 것을 확인할 수 있다.

- 캐싱을 할때 Redis에 serialize/deserialize가 잘 되어야 하므로, 캐싱하는 데이터가 저 요건을 잘 갖추도록 해야한다. (예를 들어 NoArgsConstructor가 없을 경우 jackson이 deserialize 할 수 없어서 오류가 난다.)
