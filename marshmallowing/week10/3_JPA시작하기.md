### 섹션3. JPA 시작하기

### Hello JPA - 프로젝트 생성

- 데이터베이스 방언
    - JPA는 특정 데이터베이스에 종속 X
    - 각각의 데이터베이스가 제공하는 SQL 문법과 함수는 조금씩 다름
        - 가변 문자: MySQL은 VARCHAR, Oracle은 VARCHAR2
        - 문자열을 자르는 함수: SQL 표준은 SUBSTRING(), Oracle은
          SUBSTR()
        - 페이징: MySQL은 LIMIT , Oracle은 ROWNUM
    - 방언: SQL 표준을 지키지 않는 특정 데이터베이스만의 고유한 기능

    - 하이버네이트는 40가지 이상의 데이터베이스 방언 지원

### Hello JPA - 애플리케이션 개발

```java
package hellojpa;
import jakarta.persistence.*;
public class JpaMain {
    public static void main(String[] args) {
    
        **EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");**
        //emf는 애플리케이션 로딩시점에 딱 하나만 만들어야 함
        **EntityManager em = emf.createEntityManager();**
        //em은 트랜잭션 하나마다 생성

        //jpa에서 모든 데이터 변경 작업은 트랜젝션 안에서 작업해야 함
        EntityTransaction tx = em.getTransaction();
        tx.begin();
        try{
            Member member = new Member();
            member.setId(2L);
            member.setName("HelloB");

            em.persist(member); //저장
            
            **tx.commit();**
        } catch (Exception e){
            //문제발생 시 rollback
            tx.rollback();
        } finally {
            //작업 정상 종료 시 em 닫음 - 반드시 닫아줘야 함
            **em.close();**
        }
        emf.close();
    }
}
```

- 회원 조회, 수정, 삭제

    ```java
    try{
    			 Member findMember = em.find(Member.class, 2L);
           // em.remove(findMember); 삭제
           findMember.setName("HelloJPA");
           //em.persist 할 필요 X - 변경 감지 기능(Update)
    
           tx.commit();
       } catch (Exception e){
           tx.rollback();
       } finally {
           em.close();
    }
    ```

- `Persistence.createEntityManagerFactory("hello");`
    - 설정파일의 설정 정보 읽어와서 만듦
        - `<persistence-unit name="hello">` - META-INF/persistence.xml
            - 위 name과 일치해야 함
        - persistence-unit
            - 하나의 논리적인 JPA 설정 단위
            - DB 연결 정보, 사용할 엔티티 클래스, 구현체(Hibernate 등) 설정을 묶어놓은 설정 집합

        ---

        - `Persistence.createEntityManagerFactory("hello");` 호출
        - `META-INF/persistence.xml`을 읽음
        - `<persistence-unit name="hello">`를 찾아서
        - **거기 들어 있는 설정을 기반으로 `EntityManagerFactory` 생성**

- 엔티티 매니저 팩토리는 하나만 생성해서 애플리케이션 전체에서 공유한다
- 엔티티 매니저는 쓰레드간에 공유X (사용하고 버려야 한다)
- **JPA의 모든 데이터 변경은 트랜잭션 안에서 실행**

---

- JPA를 사용하면 **엔티티 객체를 중심으로 개발**
    - 문제는 검색 쿼리
    - 검색을 할 때도 테이블이 아닌 엔티티 객체를 대상으로 검색
        - 따라서 단건 조회만 가능하다

      *→ 모든 DB 데이터를 객체로 변환해서 검색하는 것은 불가능*


<aside>
💡

애플리케이션이 필요한 데이터만 DB에서 불러오려면 결국 검색 조건이 포함된 SQL이 필요하다 ⇒ JPQL

</aside>

---

### JPQL

```java
//전체 멤버 조회 - jpql
//멤버 객체 대상으로 쿼리(테이블이 아닌 객체 대상)
List<Member> result = em.createQuery("select m from Member as m", Member.class)
                    .setMaxResults(8) //페이지네이션도 가능
                    .getResultList();

for(Member m : result){
       System.out.println("member.name = "+ m.getName());
}
```

- JPA는 SQL을 추상화한 JPQL이라는 **객체 지향 쿼리 언어** 제공
- SQL과 문법 유사
    - SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
- **JPQL은 엔티티 객체를 대상으로 쿼리**
- **SQL은 데이터베이스 테이블을 대상으로 쿼리**

<aside>
💡

JPA는 전체 ORM 기능의 표준 기술, JPQL은 그중 검색 쿼리를 작성할 때 사용하는 언어이다

</aside>

- **테이블이 아닌 객체를 대상으로 검색하는 객체 지향 쿼리**
- SQL을 추상화해서 특정 데이베이스 SQL에 의존X
- JPQL을 한마디로 정의하면 객체 지향 SQL