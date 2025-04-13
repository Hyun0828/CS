* 채팅방에서 발생한 모든 채팅을 페이지네이션을 통해 가져오기

  ```sql
  select
      c1_0.chat_id,
      u1_0.user_id,
      u1_0.name,
      c1_0.text,
      t1_0.text,
      c1_0.created_at 
  from
      chat c1_0 
  join
      user u1_0 
          on u1_0.user_id=c1_0.user_id 
  join
      translation t1_0 
          on t1_0.chat_id=c1_0.chat_id 
          and t1_0.language=?
  where
      c1_0.team_id=?
  order by
      c1_0.created_at desc 
  limit
      ?
  ```
  ```sql
  select
      c1_0.chat_id,
      u1_0.user_id,
      u1_0.name,
      c1_0.text,
      t1_0.text,
      c1_0.created_at 
  from
      chat c1_0 
  join
      user u1_0 
          on u1_0.user_id=c1_0.user_id 
  join
      translation t1_0 
          on t1_0.chat_id=c1_0.chat_id 
          and t1_0.language=?
  where
      c1_0.team_id=?
      and c1_0.chat_id<?
  order by
      c1_0.created_at desc 
  limit
      ?
  ```
  ![image](https://github.com/user-attachments/assets/3ffab85d-29be-4554-a558-ef03d2586eee)

  나는 chat 테이블에서 team\_id + created\_at 복합 인덱스를 통해 먼저 데이터를 뽑고, user와 translation 테이블과 조인해서 데이터를 가져오는 쿼리를 기대했는데 실제로 실행계획을 보니

  1. translation 테이블에서 language로 필터링
  2. 1번 결과와 chat 테이블 조인
  3. 2번 결과와 user 테이블 조인
  4. 정렬 및 limit 조건 처리

  이렇게 실행되고 있어 매우 비효율적인 쿼리가 발생했다.
  
  내 의도대로면 드라이빙 테이블이 chat이어야 하는데 QueryDSL에서는 StraightJoin을 지원하지 않는다고 한다..
  그래서 쿼리를 2개로 나누었다.

  ```sql
  select
      c1_0.chat_id 
  from
      chat c1_0 
  where
      c1_0.team_id=?
  order by
      c1_0.created_at desc 
  limit
      ?
  ```
  ![image](https://github.com/user-attachments/assets/ad25dfde-49ad-45df-acc1-d380a1fd30d3)

  ```sql
  select
      c1_0.chat_id,
      u1_0.user_id,
      u1_0.name,
      c1_0.text,
      t1_0.text,
      c1_0.created_at 
  from
      chat c1_0 
  join
      user u1_0 
          on u1_0.user_id=c1_0.user_id 
  join
      translation t1_0 
          on t1_0.chat_id=c1_0.chat_id 
          and t1_0.language=?
  where
      c1_0.chat_id in (?, ?, ?, ?, ?)
  ```
  ![image](https://github.com/user-attachments/assets/010e691a-6477-4d5e-89f3-d19a63610c64)

  어차피 chatIds 가져오기는 커버링 인덱스로 가능해서 .. 먼저 가져온 후 where 절에 조건으로 넣어줬더니 내가 원하는대로 chat을 드라이빙 테이블로 잘 동작했다.
  쿼리 자체를 바꿔서 해결은 했는데 ,, 옵티마이저가 왜 저런 식으로 동작했는지 정확한 이유는 더 공부를 해봐야 할 것 같다.

  근데 chat 순서가 보장이 안 되는 문제가 생겼다.**
  IN 쿼리로 가져와 다시 Where절에 넣어도 순서가 보장이 안 된다고 한다. JPQL이나 Native Query면 모르겠는데 찾아봐도 QueryDSL에선 방법을 모르겠다..

  ```sql
  select
      c1_0.chat_id 
  from
      chat c1_0 
  where
      c1_0.team_id=?
  limit
      ?
  ```
  그래서 그냥 order by 절을 생략하고 애플리케이션 단에서 정렬시켰다. O(nlogn)이어도 페이지네이션이라 n=5라 성능상 영향은 없을 것이라고 판단했다.
