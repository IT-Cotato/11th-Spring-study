[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발]섹션2. 프로젝트 환경설정](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%982.-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95)

### 💡 프로젝트 생성

- H2 : 간단하게 웹 애플리케이션을 데이터베이스를 내장해서 쓸 수 있는 등 간단함
- spring-boot-starter : 필요한 의존관계들을 다 가져와줌

```java
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-validation'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'com.h2database:h2'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

- 스프링부트 메인 실행하면 → Tomcat started on port 8080 → http://localhost:8080/ 접속했을 때 Whitelabel 뜨면 성공
- Settings > plugin > lombok 검색 실행
- Settings > annotation Processors > Enable annotation processing 체크

---

### 💡  View 환경 설정

- thymeleaf
    - 장점 ) Natural templates 가지고 있음
    - 단점 ) 메뉴얼을 봐야함
- 스프링 부트 thymeleaf viewName 매핑
    - resources:templates/ +{ViewName}+ .html
- spring-boot-devtools 라이브러리

    ```java
    	// gradle 에 추가
    	implementation 'org.springframework.boot:spring-boot-devtools'
    ```

    - html 파일을 컴파일만 해주면 서버 재시작 없이 View 파일 변경이 가능
    - Build > recompile 만 하면 됨

---

### 💡 H2 데이터베이스 설치

```java
> cat h2.sh
#!/bin/sh
dir=$(dirname "$0")
java -cp "$dir/h2-2.3.232.jar:$H2DRIVERS:$CLASSPATH" org.h2.tools.Console "$@"
> ./h2.sh
```

- 처음은 파일 모드로 접근 : jdbc:h2:~/jpashop
- 이후부터는 jdbc:h2:tcp://localhost/~/jpashop 이렇게 접속
    - application.yml 에 쓰여진대로

---

### 💡 JPA와 DB 설정, 동작확인

- 테스트 코드
    - 엔티티 매니저를 통한 모든 데이터 변경은 Transaction 안에서 이루어져야함
    - @Transaction 이 test에 있으면 test 끝나면 DB를 rollback 함
    - 같은 영속성 컨텍스트 안에서는 ID 값이 같으면 같은 엔티티로 식별함
- 쿼리 파라미터 로그 남기기
    - SQL 실행 파라미터를 로그로 남김

    ```java
    // applicatino.yml 에 추가
    org.hibernate.orm.jdbc.bind: trace
    
    // gradle 에 추가
    implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.10.0'
    ```


---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 엔티티 매니저를 통한 모든 데이터 변경은 Transaction 안에서 이루어져야함
    - JPA는 변경을 즉시 DB에 반영하지 않고, 트랜잭션이 커밋되는 시점에 SQL을 실행함
    - 영속성 컨텍스트에 변경 사항을 적재해두고 트랜잭션이 커밋되는 시점에 변경 감지(dirty checking)을 통해 SQL을 생성하고 실행함
    - 영속성 컨텍스트란 ? Entity를 영구 저장하는 환경으로 EntityManager를 통해 Entity를 영속성 컨텍스트에 보관 및 관리를 함
    - 엔티티 매니저란? 영속 컨텍스트에 접근하여 엔티티에 대한 DB 작업을 함
    - JpaRepository를 쓰면 EntityManager를 내부에서 대신 써주기 때문에 직접 제어할 필요가 있을 때만 EntityManager를 쓰면 됨
2. 같은 영속성 컨텍스트 안에서는 ID 값이 같으면 같은 엔티티로 식별함
    - JPA는 ID(PK)가 같은 엔티티는 같은 객체라고 간주하고, 내부적으로는 1차 캐시를 통해 해당 객체를 재사용
    - 영속성 컨텍스트는 persist()나 find() 등으로 영속 상태가 된 객체만 1차 캐시로 가지고 있음
    - find()로 조회할 때, 이미 같은 ID의 엔티티가 있다면 DB에서 새로 가져오지 않고 캐시에 있던 같은 인스턴스를 반환함