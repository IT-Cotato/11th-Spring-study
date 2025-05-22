[[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발] 섹션4. 애플리케이션 구현 준비 / 섹션5. 회원 도메인 개발]](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%984.-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B5%AC%ED%98%84-%EC%A4%80%EB%B9%84-%EC%84%B9%EC%85%985.-%ED%9A%8C%EC%9B%90-%EB%8F%84%EB%A9%94%EC%9D%B8-%EA%B0%9C%EB%B0%9C)

### 💡 구현 요구 사항

- 주요 비지니스 로직만 구현
- 예제를 단순화 하기 위해 로그인과 권한 관리는 하지 않고 파라미터 검증과 예외 처리 단순화 등

---

### 💡 애플리케이션 아키텍처

- 계층형 구조 사용
    - controller, web : 웹 계층
    - service : 비지니스 로직, 트랜잭션 처리
    - repository : JPA를 직접 사용하는 계층, 엔티티 매니저 사용
    - domain : 엔티티가 모여 있는 계층, 모든 계층에서 사용
- 패키지 구조
    - domain
    - exception
    - repository
    - service
    - web
- 개발 순서
    - 서비스, 리포지토리 계층을 개발하고 테스트 케이스를 작성해서 검증, 마지막에 웹 계층 적용