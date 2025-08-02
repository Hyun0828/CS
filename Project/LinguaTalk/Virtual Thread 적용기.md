기존에는 ThreadPoolTaskExecutor를 Bean으로 등록하여 Async 동작 시 스레드 풀에서 플랫폼 스레드를 생성해서 사용하고 있었다.

그런데 이 방식은 비동기 요청이 순간적으로 몰릴 때 플랫폼 스레드 개수가 급증하는 문제가 있다. 물론 스레드 풀 설정 값을 크게 잡으면 가능하긴 하지만 플랫폼 스레드는 메모리를 많이 잡아먹기 때문에 사실상 한계가 있다.

또한 플랫폼 스레드가 많아지면서 컨텍스트 스위칭 비용도 무시할 수 없다.

그래서 RabbitMQ와 같은 "메시지 브로커" 도입을 고려했으나 서버 인스턴스를 추가해야 된다는 점이 발목을 잡았다. 지금 당장은 플젝이 끝나기도 했고 여러 대 가용할 여건이 안 되니..

스레드 풀 설정 값을 늘리는 방법을 제외하고 단일 인스턴스에서 해결할 수 있는 방법으로 뭐가 있을까 생각해보다가 WebFlux는 러닝커브가 너무 크고 간단하게 적용 가능한 Virtual Thread를 선택했다.

기존 Platform Thread 사용 로직들을 Virtual Thread로 변경해보자.

최신 Spring Boot 버전과 JDK21 이후 버전을 사용하고 있다면 단순 설정만으로 사용할 수 있다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

우선 yml 파일에 위와 같이 설정을 해주고

```java
@Configuration
public class AsyncConfig {

    @Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    public AsyncTaskExecutor asyncTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }

    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}

```

Config 설정을 해주면 끝난다. 그런데 스케줄러 같은 경우에도 가상 스레드를 사용할 수 있지만 큰 효과가 없을 것 같아 굳이 사용하진 않고 기존 스레드 풀을 이용해 동작하게 만들었다.

우선 내가 @Async 및 스레드를 어디서 사용하고 있었을까?

## Scheduler를 통해 Hot Term 정보를 Redis에 업데이트

```java
@Scheduled(fixedRate = 36000000) // 1시간
public void hotTermUpdate() {
    Thread currentThread = Thread.currentThread();
    log.info("Thread info: name = {}, isVirtual = {}", currentThread.getName(), currentThread.isVirtual());

    // 1. Local Cache에 저장된 누적 호출 횟수를 가져온다.
    Map<Object, Object> cacheEntriesForCount = termCacheQueryService.getAllKeyAndValueInCache(
            CacheType.TERM_FIND_COUNT.getCacheName());

    // 2. 누적 호출 횟수는 Hash를 이용해 Redis에 값을 업데이트 한다.
    updateTermFindCountHash(cacheEntriesForCount);

    // 3. Local Cache에 저장했던 값은 삭제한다.
    deleteTermFindCountHashInLocalCache(cacheEntriesForCount);

    // 4. Redis Publish
    redisMessagePublisher.publish("HotTerm", "Ready for Hot Term");
}
```

위 메소드에서 사실 @Async와 @Scheduled를 동시에 사용했었는데 어차피 @Scheduled 어노테이션을 사용하면 자체적으로 스케줄링 Thread Pool을 사용한다고 한다. 중복 적용이 의미가 없었던거지.

스케줄링 같은 경우에 어차피 서버 1대에서 순간적인 중복 스레드 발생이 일어나지 않으니 굳이 가상 스레드니 스레드 풀이니 커스텀 해줄 필요가 없을 것 같다.

```java
Thread info: name = MessageBroker-1, isVirtual = false
```

## Redis Pub/Sub으로 Redis의 Hot Term 정보를 받아와 Hot Term Warming

```java
@Async
@Override
public void onMessage(Message message, byte[] pattern) {
    log.info("Redis Subscribe !!");
    Thread currentThread = Thread.currentThread();
    log.info("Thread info: name = {}, isVirtual = {}", currentThread.getName(), currentThread.isVirtual());
    Map<String, Double> hotTermMap = calculateHotTerm();
    Map<String, List<Language>> wordLanguageMap = getWordLanguageMap(hotTermMap);
    List<TermIdAndWordDto> termIdAndWordDtos = getAllTermIdAndWords(hotTermMap);
    updateHotTermInLocalCache(wordLanguageMap, termIdAndWordDtos);
    log.info("Hot Term Warming !!");
}
```

이 메소드도 어차피 스케줄링이라 위와 같은 이유도 있는데 Spring에서는 Redis 메시지 처리할 때 자체적인 스레드 풀을 관리한다고 한다. 그래서 굳이 커스텀 스레드 풀을 사용할 필요가 없어 보인다.

```java
Thread info: name = redisMessageListenerContainer-1, isVirtual = false
```

## 번역 API 호출

```java
@Async
public CompletableFuture<TranslatedDataDto> translate(final String text, final List<TermDataDto> termDataDtos,
                                                      final Language language) {
    try {
        log.info("채팅 번역");
        Translation translation = translate.translate(text,
                Translate.TranslateOption.sourceLanguage(SOURCE_LANGUAGE_CODE),
                Translate.TranslateOption.targetLanguage(language.getCode()));
        String translatedText = translation.getTranslatedText().replaceAll("&#39;", "'");
        TermPairDto result = translateTerms(translate, termDataDtos, language);
        return CompletableFuture.completedFuture(
                createdTranslatedDataDto(translatedText, result.getTranslatedTerms(), result.getTtSet())
        );
    } catch (Exception e) {
        log.error("Google Translate API 호출 중 오류 발생", e);
        return CompletableFuture.completedFuture(createdTranslatedDataDto("번역 실패", Map.of(), Set.of()));
    }
}
```

```java
Thread info: name = , isVirtual = true
```

## 현장용어 데이터 Local Cache 저장

```java
@Async
public void updateTermMetaDataInLocalCache(final List<TermDataDto> terms, final Set<Language> languageSet) {
    log.info("Local Cache에 현장 용어 호출 횟수 저장");
    Thread currentThread = Thread.currentThread();
    log.info("Thread info: name = {}, isVirtual = {}", currentThread.getName(), currentThread.isVirtual());
    terms.forEach(dto -> {
        languageSet.forEach(language -> {
            String word = dto.getTerm();
            termCacheCommandService.updateFindCount(word, language);
        });
    });
}
```

```java
Thread info: name = , isVirtual = true
```

## 결론

내가 가상 스레드의 사용을 생각했던 이유는 기존에 플랫폼 스레드에서 처리하고 있던 비동기 처리 메소드들 중에 번역 API 호출처럼 채팅이 순간적으로 몰릴 때 플랫폼 스레드 개수가 급증하는 현상을 방지하기 위해서 였다.

왜냐면 플랫폼 스레드는 병렬처리 될 수 있는 스레드 생성 개수에 제한이 어느정도 있기 때문이다. 메모리 부담이든 뭐든.

이 문제가 발발될 수 있는 케이스는 3, 4번의 경우이며 1, 2번처럼 스케줄링되는 경우에는 스레드 생성이 급증하는 경우가 없기 때문에 자체적인 Pool에서 플랫폼 스레드를 생성해 실행되게 하자.
