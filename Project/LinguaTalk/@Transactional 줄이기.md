요즘 IT 기업들의 기술 블로그를 열심히 보고 있는데 재밌는 포스팅을 보게 되어 프로젝트에 적용해보았다.

https://tech.kakaopay.com/post/jpa-transactional-bri/

카카오페이 온라인 결제 서비스에서 쿼리 발생 비율을 분석한 결과 

- select, commit 쿼리 : 약 5K
- update, insert 쿼리 : 약 3K 미만
- set_option 쿼리 : 약 14K

비율로 발생했다고 한다. 그렇다면 set_option 쿼리란 무엇일까?

MySQL에서 set_option은 말 그대로 option을 setting하는 모든 쿼리이다. autocommit, transaciton, sql_mode, character_set_results 같은 쿼리들이다.

생각해보면 @Transactional과 관련된 쿼리들이다. 

사실 JPA로 개발하면서 서비스 계층에 무지성으로 @Transactional을 달아주는 습관이 있었는데 스스로 돌아볼 좋은 기회가 되었던 것 같다.

카카오페이 팀에서 트랜잭션이 없는 경우 select 쿼리가 2~3배 정도 증가해 DB 사용 성능이 크게 개선되었다고 한다.

무지성으로 달아주었던 TX 어노테이션으로 인해 발생한 set_option 쿼리들이 알게 모르게 성능에 영향을 끼치고 있었던 것이다.

![image](https://github.com/user-attachments/assets/70cb9fe8-9f06-439d-9800-b54951766620)

단순 조회 메소드에 @Transactional을 사용하면 위와 같이 select 쿼리 1개를 위해 6개의 쿼리가 더 발생한다.

트랜잭션을 적절히 사용하기 위해 우선 JPA의 **OSIV**에 대해 알아보자.

## OSIV(Open Session In View)

OSIV는 영속성 컨텍스트를 뷰까지 열어두는 기능이다. 따라서 뷰에서도 엔티티가 영속 상태이기 때문에 지연 로딩을 사용할 수 있다.

![image](https://github.com/user-attachments/assets/2a911384-eeb1-4f7f-98f4-f0f771fec2fd)

OSIV를 키면 위와 같이 영속성 컨텍스트의 생명 주기가 설정되는데 이는 spring에서 기본 값으로 설정되어 있다.

- 클라이언트의 요청이 발생하면 서블릿 필터나 인터셉터에서 영속성 컨텍스트를 생성한다.
- 서비스 계층에서 @Transactional로 TX를 시작하면 이전에 생성한 영속성 컨텍스트를 찾아와 트랜잭션을 시작한다.
- TX가 커밋되면 영속성 컨텍스트를 플러시한다.
- 뷰까지 영속성 컨텍스트가 유지되기 때문에 뷰 계층에서 조회한 엔티티는 영속 상태를 유지한다.
- 서블릿 필터나 인터셉터로 요청이 돌아오면 영속성 컨텍스트를 종료한다. 이 때 플러시를 호출하지 않고 종료한다.

정리해보면 OSIV가 켜져 있다면 트랜잭션이 커밋된 뷰 계층에서도 Lazy Loading이 가능해 엔티티 조회가 가능하다. 

이것을 트랜잭션 없이 읽기라고 한다. 그러나 트랜잭션 범위 밖에서는 수정할 수 없다.

좋아 보이지만 큰 단점이 있다. OSIV 설정이 켜져 있다면 최초 DB 커넥션 시작 시점부터 API 응답이 종료될 때 까지 영속성 컨텍스트와 DB 커넥션을 유지한다. 영속성 컨텍스트는 기본적으로 DB 커넥션을 유지하기 때문인데 이렇게 되면 너무 오랜 시간동안 DB 커넥션 리소스를 사용하기 때문에 **실시간 트래픽이 중요한 어플리케이션에서는 커넥션이 모자랄 수 있다.**

예를 들어 외부 API를 호출하면 API 대기 시간 만큼 리소스를 물고 있기 때문에 커넥션 리소스 낭비로 이어진다.

따라서 실무에선 대부분 OSIV를 끄는 전략을 선택한다.

![image](https://github.com/user-attachments/assets/d1f9a6bc-f2a6-454b-ba03-9593403c890d)

위와 같이 OSIV를 끄면 영속성 컨텍스트의 라이프 사이클은 트랜잭션과 동일해지기 때문에 트랜잭션이 커밋되면 영속성 컨텍스트가 종료되고 DB 커넥션도 반환해 리소스를 낭비하지 않게 된다.

물론 이 방식은 Lazy Loading을 트랜잭션 내에서 모두 처리해야 하며 뷰에서 엔티티를 조회할 수 없다.

## 적절한 @Transactional 사용 방식

실제 프로젝트 코드에 적용해보았다!

- @Transactional이 필요 없는 경우
    - 단일 update, insert 쿼리에 대해서는 TX가 필요 없다. DB에서 자체적으로 원자성을 보장하기 때문이다.
        
        ```java
        public void batchInsertUserTeam(final List<Long> userIds, final Long teamId) {
            userTeamBatchRepository.userTeamBatchInsert(userIds, teamId);
        }
        ```
        
    - 프로젝션을 통해 DTO를 조회해서 반환하는 경우
        
        ```java
        public List<Long> findAllUserIdByTeamId(final Long teamId) {
            List<Long> userIds = userTeamRepository.findAllUserIdByTeamId(teamId);
            this.validateUserTeamData(userIds);
            return userIds;
        }
        ```
        
    - 엔티티를 조회한다고 하더라도 Caller에서 @Transactional이 이미 시작된 경우
        
        ```java
        @Transactional
        public ChatDto.ChatDetailDto createChat(final ChatMessageRequestDto chatRequestDto, final Long teamId) {
            User user = userQueryService.findByUserId(chatRequestDto.getUserId());
            Team team = teamQueryService.findByTeamId(teamId);
            Chat chat = ChatConverter.toChat(chatRequestDto, user, team);
            chatRepository.save(chat);
            return ChatConverter.toChatDetailDto(chat);
        }
        
        public User findByUserId(final Long userId) {
            return userRepository.findById(userId).orElse(null);
        }
        ```
        
- @Transactional이 필요한 경우
    - Facade 계층처럼 여러 로직이 원자적으로 처리되어야 하는 경우
        
        ```java
        @Transactional
        public void createTeam(final TeamCreateRequestDto requestDto) {
            Long teamId = teamCommandService.createTeam(requestDto.getName());
            userTeamCommandService.batchInsertUserTeam(requestDto.getUserIds(), teamId);
            eventPublisher.publishEvent(new TeamCreateEvent(this, requestDto.getUserIds(), teamId));
        }
        ```
        
    - Dirty Checking을 통해 업데이트 되는 경우, 더티 체킹은 TX가 필요하다.
        
        ```java
        @Transactional
        public void updateInRoomWhenJoin(final Long userId, final Long teamId) {
            UserTeam userTeam = userTeamQueryService.findUserTeamByUserIdAndTeamId(userId, teamId);
            userTeam.updateInRoomWhenJoin();
            userTeam.updateUnReadMessageWhenJoin();
        }
        ```
        
    - 연관 관계를 위해 영속 상태의 엔티티가 필요한 경우
        
        ```java
        @Transactional
        public void createTranslation(final String text, final Language language, final Long chatId) {
            Chat chat = chatQueryService.findChatById(chatId);
            Translation translation = Translation.builder()
                    .text(text)
                    .language(language)
                    .chat(chat)
                    .build();
            translationRepository.save(translation);
        }
        ```
        

**추가로 Redis 관련 로직은 TX에서 제외하자. 왜 그럴까?**

```java
@Transactional
public void doSomething() {
    userRepository.save(user); // DB insert
    redisTemplate.opsForValue().set("user", "값"); // Redis insert
    throw new RuntimeException(); // 예외 발생!
}
```
위 코드에서 redis는 롤백되지 않기 때문에 보통은 트랜잭션이 성공적으로 종료(커밋)된 이후에 Event를 발행해서 Redis를 처리한다고 한다.

```java
@Transactional
public void createTeam(final TeamCreateRequestDto requestDto) {
    Long teamId = teamCommandService.createTeam(requestDto.getName());
    userTeamCommandService.batchInsertUserTeam(requestDto.getUserIds(), teamId);
    eventPublisher.publishEvent(new TeamCreateEvent(this, requestDto.getUserIds(), teamId));
}
```
**이 때 @TransactionalEventListener를 사용하면 쉽게 트랜잭션 커밋 이후 이벤트를 발행할 수 있다.**

그래서 프로젝트에선 위 코드처럼 처리했다.
