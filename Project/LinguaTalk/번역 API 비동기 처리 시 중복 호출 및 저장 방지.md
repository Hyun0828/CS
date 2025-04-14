LinguaTalk에서 개발한 실시간 채팅 서비스의 본질은 채팅과 채팅에 포함된 현장 용어들에 대해 번역본을 제공하는 것이다. 번역을 위해 Google Cloud Translation API를 사용했는데 채팅 발생 후 번역 API를 호출했다.

**문제 : 1초 정도 딜레이가 발생해 실시간 채팅 느낌이 나질 않았다**. 

→ 그래서 번역 API는 **비동기**로 처리하기로 결정했다. 

**문제 : 그런데 이러면 번역 데이터를 어떻게 클라이언트로 전송할 수 있을까?**

-> **소켓 연결** 을 추가로 진행해서 HTTP 통신처럼 클라이언트의 요청 없이 서버에서 전송할 수 있게 만들었다.

다 해결된 것 같지만 우리는 채팅 메시지에 대한 번역 뿐만 아니라 현장 용어에 대한 번역을 진행해야 하는데 이 과정에서 문제가 생길 수 있다. 

1. 채팅이 발생하면 채팅방에 속한 사용자 숫자만큼 번역 메소드가 호출되는데, 이 때 같은 채팅 / 용어에 대해 중복 API 호출이 발생할 수 있다.
2. 이후 추가적인 채팅이 발생했을 때 이전에 번역된 현장 용어들에 대해 중복 API 호출이 발생할 수 있다.

결국 중복 API 호출을 막는 로직이 필요했다.

현재 채팅 메소드를 보자

```java
public void chat(final ChatMessageRequestDto chatRequestDto, final Long teamId, final ChatDto.ChatDetailDto chatDetailDto) {
    TermDataWithNewChatDto result = termManager.query(chatRequestDto.getMessage()); // 현장용어 추출
    chatSendService.sendChatMessage(result, chatRequestDto.getName(), chatDetailDto, teamId); // 채팅은 즉시 전송

    // 현장용어 추출 후 Term 엔티티 저장하기
    createTerms(result);

    // 채팅방에 속한 모든 사용자의 Id와 언어 가져오기
    List<UserIdAndLanguageDto> dtos = userTeamQueryService.findAllUserIdAndLanguageByTeamId(teamId);

    // 채팅방에 속한 모든 사용자에 대해 번역 데이터를 전송하고 채팅방 순서를 갱신한다.
    dtos.forEach(dto -> {
        Language language = dto.getLanguage();
        chatSendService.sendTranslatedMessage(result, language, chatDetailDto.getChatId(), teamId, dto.getUserId());
        chatSendService.sendTeamData(chatDetailDto, teamId, dto.getUserId());
    });

    // 현장용어를 위한 Local Cache 업데이트
    termMetaDataCommandService.updateTermMetaDataInLocalCache(result.getTerms(), getLanguageSet(dtos));
}
```

chatSendService의 sendTranslatedMessage 메소드를 보자

```java
 public void sendTranslatedMessage(final TermDataWithNewChatDto result, final Language language,
                                    final Long chatId, final Long teamId, final Long userId) {
    String translationCacheKey = language + ":" + chatId;
    CompletableFuture<TranslatedDataDto> future = translationAPICache.get(translationCacheKey, key ->
            termManager.translate(result.getNewChat(), result.getTerms(), language));
    future.thenAccept(dto -> {
        dto.getTtSet().forEach(data -> createTranslatedTerm(language, data));
        createTranslation(language, dto, chatId);
        messagingTemplate.convertAndSend(TRANSLATE_SUB_URL + teamId + "/" + userId,
                ChatConverter.toTranslatedTextResponseDto(dto.getTranslatedText(), dto.getTranslatedTerms(), chatId));
    });
    // TODO 비동기 처리에 대한 예외처리
}
```

translate 메소드가 비동기로 처리되고 비동기 처리가 완료되면 소켓 통신으로 데이터를 전송해 실시간 채팅에서 중요한 **채팅 응답 속도 문제**를 개선했다.

그럼 **중복 API 호출 문제**는 어떻게 해결할 수 있을까?

```java

@Service
@Slf4j
@RequiredArgsConstructor
public class ChatSendService {

    private final Cache<String, Boolean> translatedTermCache = Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.SECONDS)
            .build();
    private final Cache<String, Boolean> translationCache = Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.SECONDS)
            .build();
    private final Cache<String, CompletableFuture<TranslatedDataDto>> translationAPICache = Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.SECONDS)
            .build();

    public void sendTranslatedMessage(final TermDataWithNewChatDto result, final Language language,
                                      final Long chatId, final Long teamId, final Long userId) {
        String translationCacheKey = language + ":" + chatId;
        CompletableFuture<TranslatedDataDto> future = translationAPICache.get(translationCacheKey, key ->
                termManager.translate(result.getNewChat(), result.getTerms(), language));
        future.thenAccept(dto -> {
            dto.getTtSet().forEach(data -> createTranslatedTerm(language, data));
            createTranslation(language, dto, chatId);
            messagingTemplate.convertAndSend(TRANSLATE_SUB_URL + teamId + "/" + userId,
                    ChatConverter.toTranslatedTextResponseDto(dto.getTranslatedText(), dto.getTranslatedTerms(), chatId));
        });
        // TODO 비동기 처리에 대한 예외처리
    }

    private void createTranslatedTerm(final Language language, final TermDto.CreateTranslatedTermEntityDto data) {
        String isNewTermKey = language + ":" + data.getWord();
        if (translatedTermCache.asMap().putIfAbsent(isNewTermKey, true) == null) {
            Long termId = termQueryService.findTermIdByWord(data.getWord());
            translatedTermCommandService.createTranslatedTerm(termId, data.getLanguage(), data.getTranslatedWord());
        }
    }

    private void createTranslation(final Language language, final TranslatedDataDto dto, final Long chatId) {
        String isNewChatKey = language + ":" + chatId;
        if (translationCache.asMap().putIfAbsent(isNewChatKey, true) == null) {
            translationCommandService.createTranslation(dto.getTranslatedText(), language, chatId);
        }
    }
}

```

코드가 너무 길어 설명에 필요한 부분만 남겼다.

1번 케이스처럼 하나의 채팅에 대해 중복 API 호출을 방지하기 위해 Local Cache인 Caffeine 캐시를 사용했다. 비동기 메소드가 호출되면 스레드가 하나 생성되고 결국 멀티스레드 환경에서 제어해줄 방법이 필요했는데 생각한 후보는 여러 가지가 있었다.

**<동일 채팅 및 용어에 대한 중복 API 호출 방지>**

1. **번역 API 동기 처리**
    
    ![image](https://github.com/user-attachments/assets/73e949dc-fe75-4f9e-941b-f64700511f2c)
    
    채팅이 발생하면 번역 API를 먼저 전부 처리하고 데이터를 전송할 수도 있다. 왼쪽 그림처럼 하면 되는데 채팅방에 서로 다른 언어 사용자 N명이 있을 때, 번역 API가 최소 N번은 호출되고 나서야 번역본이 전달되기 때문에 성능 문제가 발생하여 너무 느리다.
    
2. **Lock**
    
    Lock도 좋은 방법이긴 하나 Lock은 기본적으로 성능 저하가 필연적이다. 실시간 채팅 서비스에선 부적절하다고 판단했다.
    
3. **HashMap / ConcurrentHashMap**
    
    ![image](https://github.com/user-attachments/assets/4e4ea972-0d32-40db-a013-f71194b00bb0)

    1번과 달리 최초 번역 API만 호출하는 방법도 있다. 
    
    HashMap은 멀티스레드 환경에서 Thread-Safe하지 않다. 
    
    그래서 ConcurrentHashMap을 쓰고 language와 chat이나 term 쌍을 key로 두어 처리할 수 있었다.
    
    그러나 여기서 문제는 메모리에 key를 무한히 누적 시킬 수 없다는 것이다.
    
    ConcurrentHashMap은 자체적인 TTL 기능을 제공하지 않기 때문에 수동 삭제가 필요한데 이를 구현하는 것보다 Caffeine 같은 로컬 캐시나 Redis 캐시를 쓰는 것이 낫다고 생각했다.
    
4. **Redis 캐시**
    
    Redis를 쓰면 TTL을 설정할 수 있어 충분히 사용 가능하다. 그러나 Redis와의 통신도 어쨌든 네트워크 비용이 발생하기 때문에 로컬 캐시를 두고 굳이 Redis를 사용해야 할 필요성을 느끼지 못했다.
    
5. **Caffeine**
    
    최종적으로 선택한 방법이다. TTL 기능도 제공하고 서버 로컬 캐시라 네트워크 비용도 발생하지 않으며 Thread-Safe 하기에 최적의 방법이라고 생각했다. 캐시에 Key를 채팅:언어 혹은 용어:언어로 설정해서 호출된 적이 있다면 캐시에 저장했다.
    

**<다른 채팅 발생 시 현장 용어에 대한 중복 API 호출 방지>**

메시지 번역(Translation)은 어차피 Chat과 1대1 관계기 때문에 저장해도 상관이 없다. 그러나 현장 용어는 다르다.

앞서 Caffeine 캐시를 통한 해결책은 TTL이 설정되어 있어 시간이 지나면 삭제되기 때문에 여기서 사용할 순 없었다. 그래서 생각한 방법이 **UPSERT**문이다.

```java
public void createTranslatedTerm(final Long termId, final Language language, final String word) {
    translatedTermRepository.upsertTranslatedTerm(language.name(), word, termId);
}

@Modifying
@Query(value = "INSERT INTO translated_term (language, word, term_id) "
        + "VALUES (:language, :word, :termId) "
        + "ON DUPLICATE KEY UPDATE word = VALUES(word)", nativeQuery = true)
void upsertTranslatedTerm(@Param("language") String language,
                          @Param("word") String word,
                          @Param("termId") Long termId);
```

만약 Caffeine 캐시에 Key 값이 존재하지 않으면, Upsert 구문을 통해 중복 저장을 방지했다.

여기서 **“DB에 있는 지 확인하고 없으면 추가”** 로직을 생각할 수도 있다. 나도 처음에 물론 그랬다.

그런데 이 로직의 큰 단점이 있다.

| **start tx1** |  |
| --- | --- |
| select | **start tx2** |
|  | select |
| insert |  |
|  | insert |
| commit | commit |

만약 위와 같이 동작하게 되면 tx2에선 당연히 비어있을 거라 생각하고 insert를 하게 되어 중복 저장 문제(**Race Condition**)가 생긴다.

Lock을 걸어 해결해줄 수도 있지만 성능, 복잡성 측면에서 UPSERT에 비해 유리한 점이 없다.
