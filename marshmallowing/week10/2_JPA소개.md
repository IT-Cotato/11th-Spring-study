### 섹션2. JPA 소개

### SQL 중심적인 개발의 문제점

- 애플리케이션: 객체 지향 언어
- 데이터베이스: 관계형 DB

> 객체를 관계형 DB에 관리 ⇒ 수많은 SQL 필요 (INSERT INTO, UPDATE …)
>

### JPA 소개

- JPA
    - JAVA Persistence API
    - 자바 진영의 ORM 기술 표준
- ORM
    - Object-relational mapping(객체 관계 매핑)
    - 객체는 객체대로 설계
    - 관계형 데이터베이스는 관계형 데이터베이스대로 설계

      ⇒ **ORM 프레임워크가 중간에서 매핑**


---

- JPA는 애플리케이션과 JDBC 사이에서 동작
- EJB-엔티티 빈(자바 표준) → 하이버네이트(오픈 소스) → **JPA(자바 표준)**
- JPA는 표준 명세
    - JPA는 **인터페이스의 모음**
    - JPA 2.1 표준 명세를 구현한 3가지 구현체
        - **하이버네이트**(90%이상), EclipsLink, DataNucleus

---

- **JPA를 왜 사용해야 하는가?**
    - SQL 중심적인 개발에서 객체 중심으로 개발
    - 생산성
        - 저장 - jpa.persist(member)
        - 조회 - Member member = jpa.find(memberId)
        - 수정 - member.setName(”변경 이름”)
        - 삭제 - jpa.remove(member)

      ⇒ 자동으로 SQL 쿼리 날려줌

    - 유지보수
        - 기존) 필드 변경시 모든 SQL 수정

          → JPA) 필드만 추가하면 됨, SQL은 JPA가 처리

    - 패러다임의 불일치 해결
        - 상속
            - 저장
                - 개발자: `jpa.persist(album);`
                - JPA:
                    - INSERT INTO ITEM …
                    - INSERT INTO ALBUM …

              ⇒ JPA가 모든 테이블에 INSERT 쿼리 날려줌 (매핑 과정 생략)

            - 조회
                - 개발자: `Album album = jpa.find(Album.class, albumId);`
                - JPA:

                    ```sql
                    SELECT I.*, A.*
                    	FROM ITEM I
                    	JOIN ALBUM A ON I.ITEM_ID = A.ITEM_ID
                    ```

        - 연관관계, 객체 그래프 탐색
            - 연관관계 저장

                ```java
                member.setTeam(team);
                jpa.persist(member);
                ```

            - 객체 그래프 탐색

                ```java
                Member member = jpa.find(Member.class, memberId);
                Team team = member.getTeam();
                ```

        - 신뢰할 수 있는 엔티티, 계층
        - **동일한 트랜잭션에서 조회한 엔티티는 같음을 보장**
    - 성능 최적화
        - 1차 캐시와 동일성(identity) 보장
            1. 같은 트랜잭션 안에서는 같은 엔티티 반환 - 약간의 조회 성능 향상
            2. DB Isolation Level이 Read Commit이어도 애플리케이션에서 Repeatable Read 보장

            ```java
            String memberId = "100";
            Member m1 = jpa.find(Member.class, memberId); **//SQL**
            Member m2 = jpa.find(Member.class, memberId); **//캐시**
            //SQL을 또 날리지않고 캐시(메모리) 상에서 조회
            println(m1==m2) //true
            ```

        - 트랜젝션을 지원하는 쓰기 지연(transactional write-behind)
            - INSERT
                1. 트랜잭션을 커밋할 때까지 INSERT SQL을 모음
                2. JDBC BATCH SQL 기능을 사용해서 **한번에 SQL 전송**

                    ```java
                    transaction.begin(); //트랜잭션 시작
                    em.persist(memberA);
                    em.persist(memberB);
                    em.persist(memberC);
                    //여기까지 INSERT SQL을 DB에 보내지 않음
                    
                    **//커밋하는 순간 DB에 INSERT SQL을 모아서 보냄**
                    transaction.commit(); //트랜잭션 커밋
                    ```

            - UPDATE
                1. UPDATE, DELETE로 인한 로우(ROW)락 시간 최소화
                2. 트랜잭션 커밋 시 UPDATE, DELETE SQL을 실행하고, 바로 커밋

                    ```java
                    transaction.begin(); //트랜잭션 시작
                    changeMember(memberA);
                    changeMember(memberB);
                    비지니스_로직_수행(); 
                    //비지니스 로직 수행 동안 DB 로우 락이 걸리지 않는다
                    
                    //커밋하는 순간 DB에 UPDATE, DELETE SQL을 보냄
                    transaction.commit(); 
                    ```

        - 지연 로딩(Lazy Loading)
            - 지연 로딩: 객체가 실제 사용될 때 로딩
            - 즉시 로딩: JOIN SQL로 한번에 **연관된 객체까지 미리 조회**

            ```java
            //지연 로딩
            Member member = memberDAO.find(memberId);
            //SELECT * FROM MEMBER
            Team team = member.getTeam();
            //SELECT * FROM TEAM
            String teamName = team.getName();
            
            //즉시 로딩 - 멤버 조회시 팀도 같이 끌어옴
            Member member = memberDAO.find(memberId);
            **//SELECT M.*, T.* FROM MEMBER JOIN TEAM ...**
            Team team = memebr.getTeam();
            String teamName = team.getName();
            ```

    - 데이터 접근 추상화와 벤더 독립성
    - 표준