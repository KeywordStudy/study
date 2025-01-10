## JPA 에서 QueryDSL을 사용하는 이유 및 한계

<br>

![image](https://github.com/user-attachments/assets/8de07e70-a74c-4670-b144-e3e16edf00a8)


ORM(Object-Relational Mapping)
- 객체와 db간의 매핑
- JPA는 ORM 기술 표준 인터페이스로 정의

JPA(Java Persistence API)
- 표준 명세서(인터페이스): ORM을 구현하기 위한 API 명세
- JPA 자체는 구현체가 없으며, Hibernate와 같은 구현체를 통해 동작

JPA 구현체
- Hibernate : JPA 명세를 따르는 가장 대표적인 구현체
    - HQL(Hibernate Query Language) 추가 제공

JPQL(Java Persistence Query Language)
- JPA 표준 쿼리 언어

Criteria API
- JPA 표준에서 제공하는 동적 쿼리 작성 API
- 가독성이 떨어진다는 평이있다.

QueryDSL
- JPA와 함께 사용할 수 있는 외부 프레임워크
- JPQL이나 Criteria API보다 간결하고 직관적인 쿼리 작성
    - 타입안정성과 가독성
    - 실행전 간단하게 쿼리확인가능 (.toString())

JPA 제공 인터페이스 계층
- Repository → CRUDRepository → PagingAndSortingRepository → JpaRepository 순의 계층 구조
- JPA를 사용할 때 데이터 접근 계층을 추상화하는 주요 인터페이스들.



##### Querydsl 이란

> 타입안정성을 위한 queryDsl
> 가독성

1) 타입안정성을위한 querydsl
   
###### 기존 JPQL의 한계

문자열  쿼리작성 : 컴파일 시점에 잘못된 쿼리작성이 발견되지않음
유지보수 : 쿼리를 수정해야함 (ide에서 자동완성안됨)

```JAVA
String jpql = "SELECT u FROM User u WHERE u.nam = :name"; 
List<User> users = entityManager.createQuery(jpql, User.class)
    .setParameter("name", "John")
    .getResultList();
```
###### 컴파일 시점에 타입안정성을 체크하는 Querydsl

QClass라는 타입 안전한 메타 모델을 사용 
- 엔티티 변경시에 바로 변경과 관련된 쿼리생성부분의 에러를 체크가능
- 

```JAVA
QUser user = QUser.user;

List<User> users = queryFactory
    .selectFrom(user)
    .where(user.name.eq("John")) // 'name' 필드 QClass 정의
    .fetch();

```
