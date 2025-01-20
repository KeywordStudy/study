# 0. 들어가기에 앞서

현재 QueryDSL은 2025년 1월 11일 기준, 8개월 전에서 **5.1 버전**을 기점으로 업데이트가 중단된 상태다. 길다면 길고 짧다면 짧은 기간의 감감무소식이어서 향후 지원이 어떻게 될 지는 모르는 상황이고 자바는 계속 버전 업데이트가 이뤄지고 있기 때문에 일단은 학습에 우선점을 더 두고 공부를 한다.

요약하자면 신속하게 공부 마무리하기.

# 1. QueryDSL 개론

## 1) 정의

Java 기반의 **타입 안정성 기반** SQL, JPQL, MongoDB, JDO 등을 작성할 수 있는 **도메인 특화 언어**로, 주로 JPA와 같이 사용되며 쿼리를 동적으로 생성하면서 컴파일 시점에 문법 검증이 가능하다.

>#### (1) 컴파일 시점의 오류 검증
>
>IDE의 문법 검증 등을 통해, 혹은 빌드 시점에서 문법 에러를 검증함으로써 런타임 시점에 에러가 발생하지 않는다.
>
>#### (2) 메소드 체이닝
>
>`select()`, `where()`, `from()` 등 SQL과 유사한 문법을 메소드 체이닝으로 구현한다.
>
>#### (3) 동적 쿼리 지원
>
>`BooleanBuilder`와 `Expressions`를 사용하여 동적 쿼리를 작성할 수 있다.
>
>#### (4) SQL 추상화를 통한 JPQL 자동 생성
>
>JPQL을 자바 코드로 표현하여 이를 SQL로 자동으로 변환한다.


JPA에서 엔티티의 `usernmae` 필드와 `email` 필드를 통해 해당하는 레코드를 테이블에서 조회해오는 것은 `findByUsernameAndEmail()` 메소드 등을 통해 간단히 해결할 수 있다. 다만 JPA만으로는 복잡한 쿼리(`join` 문법이나 복수의 테이블을 엮어 원하는 조건을 **검색**할 때)를 표현하는 데에 무리가 있다.

QueryDSL은 이 점을 보완할 수 있는데, 위에서 언급한 동적 쿼리를 지원하고 가독성이 높아지면서 직관성이 확보된다.

위의 특징들을 직접 실습을 통해 활용해볼 예정

# 2. QueryDSL 설정

## 1) 개발 환경

>- 자바 모듈 버전 : OpenJDK 21
>- 프레임워크 : Spring Boot 3.4
>- 빌드 환경 : Gradle
>- 주요 의존성 : Spring Data JPA
>- RDBMS : MySQL
>- IDE : 인텔리제이 얼티메이트 에디션

## 2) build.gradle 설정

### (1) QueryDSL 버전 변수 세팅

```groovy
buildscript {
    ext {
        queryDslVersion = "5.0.0"
    }
}
```

### (2) QueryDSL 의존성 추가

```groovy
dependencies {
    // querydsl 추가(스프링부트 3 이상에서의 QueryDSL 패키지 정의)
    implementation "com.querydsl:querydsl-jpa:${queryDslVersion}:jakarta"
    annotationProcessor "com.querydsl:querydsl-apt:${queryDslVersion}:jakarta"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}
```

스프링부트 3 이상에서는 `jakarta` 패키지를 사용한다. 또한, 어노테이션 프로세싱 툴(APT)를 통해 Q클래스를 생성해야 한다.

### (3) QueryDSL 소스 코드 경로 세팅

```groovy
def querydslDir = "$buildDir/generated/querydsl"

sourceSets {
    main.java.srcDirs += [ querydslDir ]
}

tasks.withType(JavaCompile) {
    options.annotationProcessorGeneratedSourcesDirectory = file(querydslDir)
}

clean.doLast {
    file(querydslDir).deleteDir()
}
```

Q클래스 생성 경로를 `querydslDir` 변수에 담는다. 이때 그 경로는 `build/generated/querydsl`이다. 또한 Q클래스를 포함한 코드 생성 경로룰 소스 경로에 추가하고(`sourceSets` 블록) 어노테이션 프로세서가 생성한 코드를 `querydslDir`에 저장한다(`JavaCompile` 태스크). 마지막으로 `gradle clean` 실행 시, 생성된 Q클래스를 전부 제거한다.

### (4) 설정 완료 후 작동 순서

>1. `gradle build` 실행
>2. `annotationProcessor`가 엔티티 기반으로 **Q클래스**를 생성
>3. `build/generated/querydsl` 경로에 Q클래스 파일이 생성
>4. `sourceSets`에 해당 경로를 포함시켜 컴파일 시점에 QueryDSL을 사용 가능

## 3) 기타 추가 설정

### (1) 인텔리제이 어노테이션 프로세서 세팅

<img width="80%" alt="인텔리제이어노테이션프로세서세팅" src="https://github.com/user-attachments/assets/3cdb737a-eeab-4fd5-bb24-99054ef99dd3" />

이 설정은 **어노테이션 프로세서**를 Gradle 자체가 아닌 **인텔리제이 자체 빌드 시스템에서 활성화**하도록 설정한 것이다.

### (2) Gradle 기반 자바 컴파일

<img width="35%" alt="gradleQ클래스생성" src="https://github.com/user-attachments/assets/944d88d2-1bd0-4bcf-b782-d4f750b2ead7" />

`querydsl-apt`를 Gradle에서 직접 사용해서 지정했던 경로에 Q클래스를 생성한다. 새로운 엔티티 클래스가 추가될 때마다 빌드를 해주면 자동으로 경로에 맞춰 Q클래스가 추가된다.

### (3) build 패키지 내부 확인

<img width="35%" alt="Q클래스생성확인" src="https://github.com/user-attachments/assets/78be7bd9-0817-4b65-afc4-25951cb67236" />

`generated/querydsl/com/example/엔티티 패키지들` 하위에 엔티티 클래스를 기반으로 Q클래스가 생성된 것을 확인할 수 있다.

## 4) 테스트 생성 및 수정 확인

### (1) 큐클래스 생성 테스트

임시 엔티티 클래스를 생성해서, 터미널에서 `./gradlew compileJava` 명령어를 실행해서 `build` 패키지의 해당 경로에서 큐클래스가 생성되는지 확인한다.

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "post_id")
    private Long id;

    @Column(name = "title")
    private String title;

    @Column(name = "content")
    private String content;

    public Post(String title, String content) {
        this.title = title;
        this.content = content;
    }
}
```

<img width="80%" alt="터미널 :gradlew compileJava기반생성" src="https://github.com/user-attachments/assets/2c09722d-d98b-4110-bef2-98ebb9f19900" />

<img width="80%" alt="큐클래스확인" src="https://github.com/user-attachments/assets/6a412f3e-5b2a-45a3-9af5-2dcadccd570d" />

### (2) 큐클래스 수정 테스트

엔티티 클래스를 수정한 경우, `./gradlew clean compileJava` 명령어를 기반으로 큐클래스를 전부 지우고 다시 컴파일을 수행한다.

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "post_id")
    private Long id;

    @Column(name = "title")
    private String title;

    @Column(name = "category")
    private String category;
    
    @Column(name = "content")
    private String content;

    public Post(String title, String content) {
        this.title = title;
        this.content = content;
    }
}
```

<img width="80%" alt="터미널기반클린컴파일자바" src="https://github.com/user-attachments/assets/8ac083e4-34da-4324-a7c4-6a9852bb1d04" />

<img width="80%" alt="큐클래스수정" src="https://github.com/user-attachments/assets/d5d74aca-4984-405a-8aba-fd900f35a3d1" />

### (3) 실행 테스트

실행 결과, `genrated` 패키지에 엔티티 패키지 구조대로 큐클래스들이 생성\
다만 거기서 `java.lang.Exception: exception just for purpose of providing stack trace` 예외가 발생\

이것은 `UnknownNamedQueryException`인데 앱 실행에는 영향을 주지 않는다. 다만 거슬릴 경우를 대비해 로깅 레벨을 조정하는 방식으로 없앨 수 있다.

```yml
# https://mirr-coding.tistory.com/66 참고
logging:
  level:
    org:
      hibernate:
        # QueryDSL UnknownNamedQueryException 해결을 위한 로깅 조치
        resource:
          transaction:
            backend:
              jdbc:
                internal:
                  JdbcResourceLocalTransactionCoordinatorImpl: INFO
```

추가로 `src`의 `genrated` 패키지에 큐클래스가 생성됐다고 해서 달라지는 것은 없음\
엔티티 클래스의 수정이 발생하면 `./gradlew clean compileJava` 후, 재실행하면 그것에 맞춰 큐클래스도 수정

# 3. QueryDSL 예제 연습

## 1) 기본 예제

### (1) 엔티티 및 DAO 생성

엔티티는 다음과 같으며, 추후 연관관계 세팅 역시 이뤄질 예정이다.

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "post_id")
    private Long id;

    @Column(name = "title")
    private String title;

    @Column(name = "content")
    private String content;

    public Post(String title, String content) {
        this.title = title;
        this.content = content;
    }
}
```

그리고 이에 맞춰서 dao를 생성하는데, JPA 기본 CRUD에 맞춘 `JpaRepository` 상속 인터페이스 dao와 QueryDSL 전용 dao를 생성한다.

```bash
.
├── PostRepository.java               # JpaRepository 인터페이스 (기본 CRUD 제공)
├── CustomPostRepository.java         # 사용자 정의 쿼리 메서드 인터페이스
└── PostRepositoryImpl.java           # CustomPostRepository 구현체 (QueryDSL 사용)
```

위와 같이 dao를 구분 구현한 이유는 다음과 같다. 공통적으로 애플리케이션 기능에 특화된 메소드들을 `CustomPostRepository` 인터페이스에 공통 메소드들을 정의하고 이것을 JPA dao 구현 메소드와 QueryDSL dao 구현 메소드로 분류 구현한다.

이렇게 구현하는 이유는, JPA는 `PostRepository`(즉, JPA 관련 `Repository` 인터페이스 상속 구현)에 대해서는 알고 있지만, QueryDSL 관련 dao에 대해서는 모르기 때문에 QueryDSL 전용 코드를 별개로 구현해야 하는 것이다. JPA만 사용했을 때, `Repository`에 별 신경 안 써도 웬만한 기능들을 쓸 수 있었던 것에 대해 생각해 보자.

### (2) 컨트롤러 생성 및 확인

dao 상속 및 활용 구조는 다음과 같다.

```bash

CustomPostRepository # 추상 사용자 정의 메소드
    ├── PostRepository # CustomPostRepository 구현 - QueryDSL 기반 메소드 사용은 얘를 통해 이뤄짐
    └── PostRepositoryImpl # CustomPostRepository 구현 - JPAQueryFactory 활용한 QueryDSL 메소드
```

위의 구조에 따라서 실제 서비스 코드에서 사용할 dao는 `PostRepository`가 된다. 예제 코드를 짜고 실제 쿼리 출력이 어떻게 되는지 확인해본다.

```java
@RestController
@RequiredArgsConstructor
public class QueryDslController {

    private final PostRepository postRepository;

    @GetMapping("/find/{id}")
    public Post findPostById(@PathVariable("id") Long id) {
        return postRepository.getQslPost(id);
    }
}
```

<img width="70%" alt="스크린샷 2025-01-12 오후 7 26 09" src="https://github.com/user-attachments/assets/7548bd30-376d-4675-9266-fe6e60c0614f" />

히카리풀에 출력된 쿼리문을 확인해보면 다음과 같다.

```bash
2025-01-12T19:16:26.516+09:00 DEBUG 8815 --- [jpa] [nio-8080-exec-1] org.hibernate.SQL                        : /* select post
from Post post
where post.id = ?1 */ select p1_0.post_id,p1_0.content,p1_0.title from post p1_0 where p1_0.post_id=?
Hibernate: /* select post
from Post post
where post.id = ?1 */ select p1_0.post_id,p1_0.content,p1_0.title from post p1_0 where p1_0.post_id=?
```

현재 QueryDSL을 JPA와 결합한 상태에서 히카리풀을 확인하면 쿼리가 두 번 로깅되는 것을 확인할 수 있다.

QueryDSL은 **쿼리 작성**을 맡으며 hibernate가 실제 데이터베이스와의 통신(즉, **쿼리 실행**)을 맡는다. 기존에는 hibernate가 쿼리 작성과 실행을 전부 도맡았다면, 현재는 QueryDSL이 **복잡한 기능(where, group by, join)** 을 포함해서 쿼리 작성을 맡고 hibernate가 이해할 수 있는 SQL로 변환되어 실행된다. 이때 이 SQL 변환을 `JPAQueryFactory`가 맡게 된다.

즉, 저 두 개의 쿼리 출력은 각각 QueryDSL이 작성한 JPQL 쿼리와 hibernate가 실행하는 쿼리다.

## 2) 복잡한 예제 - 연관관계를 기반으로

*참조*\
*https://sjh9708.tistory.com*

연관관계 세팅을 위한 테이블 간의 관계는 다음과 같다.

<img width="80%" alt="Image" src="https://github.com/user-attachments/assets/1a7fbc61-5220-4aac-bff8-7d0174b83492" />


### (1) JOIN 이슈 및 N+1 이슈

### (2) 서브쿼리 이슈
