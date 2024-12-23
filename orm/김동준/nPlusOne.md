# N+1 문제

## 1. 개념

### (1) JPA 개념 정리
JPA는 자바 애플리케이션에서 데이터베이스와 객체와의 매핑 처리를 위한 **ORM 기술 표준**이다. 기술 표준이라 함은 인터페이스이기 때문에 이를 구현하는 프레임워크가 존재하는데 대표적인 게 Hibernate이다.
### (2) N+1 문제 원인
JPA의 기능 중 데이터베이스 테이블의 외래 키 전략을 객체 단계에서 표현하기 위한 **연관관계 매핑**과, 이 연관된 데이터를 일괄 조회하는 것이 아닌 필요할 때에만 조회함으로써 성능 효율을 추구하는 **지연 로딩**이 있는데 이 둘 때문에 N+1 문제라는 이슈가 발생한다.

## 2. 코드 기반 탐구

### 1) 테스트

#### (1) 테스트 환경 세팅 

연관관계를 설정한 두 개의 엔티티를 생성한다.

```java
@Entity
@Getter
@NoArgsConstructor
public class Team {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "team_id")
    private Long id;

    @Column(name = "team_name")
    private String name;

    @OneToMany(mappedBy = "team") // ~ToMany: LAZY
    private List<Member> members = new ArrayList<>();

    public Team(String name) {
        this.name = name;
    }

    public void addMember(Member member) {
        members.add(member);
        member.setTeam(this); // 양방향 연관 관계 동기화
    }
}
```
```java
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    @Column(name = "member_name")
    private String name;

    @ManyToOne(cascade = CascadeType.ALL) // ~ToOne: EAGER
    @JoinColumn(name = "team_id") // 외래 키(팀)를 관리하는 측(멤버)에게 부여
    private Team team;

    public Member(String name) {
        this.name = name;
    }
}
```
`Tean` 엔티티와 `Member` 엔티티는 1:N 단방향 관계를 맺는다.

이것은 데이터베이스에서도 외래키 설정을 확인할 수 있다.

<img width="667" alt="외래키설정확인" src="https://github.com/user-attachments/assets/bc83a33f-e3c8-47c5-a5d7-b5cf458fa1fe">

1:N에서 N에 해당하는 `Member`가, 1에 해당하는 `Team`의 ID를 외래 키로 가지고 있음을 확인할 수 있다.

#### (2) 테스트 코드 작성

현재 데이터베이스(MySQL)에는 다음과 같은 테이블들이 있다.

<img width="671" alt="팀,멤버필드확인" src="https://github.com/user-attachments/assets/4c66645c-1035-4d5a-80c1-49c5e815c812">

여기서 만약 이런 경우가 생길 수 있다.

>**팀의 멤버**들을 조회

현재 위의 테이블은 팀 따로, 멤버 따로 제시되어 있으니 아마 해당 경우가 원하는 테이블의 모습은 아래와 같을 것이다.

<img width="502" alt="조인문" src="https://github.com/user-attachments/assets/f24840a9-6a38-4afe-ae9f-c9bce2669790">

위의 테이블처럼 팀 정보와 그 팀에 해당하는 멤버들을 정보를 일괄 조회할 수 있다.

이것을 JPA의 데이터베이스 객체 매핑 장점을 활용해서 자바 레벨에서도 코드로 작성할 수 있다.

```java
public void findAllMembersByTeamRepo() {
    List<String> list = teamRepository.findAll().stream()
            .flatMap(team -> team.getMembers().stream()
                    .map(member -> team.getName() + ": " + member.getName()))
            .toList();

    System.out.println("결과: " + list);
}
```

이것을 실행하면 다음과 같이 hibernate 쿼리 출력 로그와 결과 로그를 확인할 수 있다.

<img width="935" alt="결과 출력 로그" src="https://github.com/user-attachments/assets/176515d3-0c06-4da9-bb38-40b1f8c33379">

### 2) 탐구

#### (1) 왜 문제가 되는가?

이게 문제가 되는 이유는, **쿼리가 중복 발신**되기 때문이다.

노란색 박스는 아래처럼 팀 테이블의 팀 ID와 이름을 조회한다.

```sql
select t1_0.team_id,t1_0.team_name from team t1_0
```

초록색 박스는 아래처럼 멤버 테이블에서 외래 키에 대응되는 팀의 ID 조건부로 팀 정보와 멤버 정보를 같이 조회한다.

```sql
select m1_0.team_id,m1_0.member_id,m1_0.member_name from member m1_0 where m1_0.team_id=?
```

사실 JPA 입장으로 봤을 때는 코드 로직을 해석해보면 충분히 납득이 간다.

아래처럼 순서가 이뤄지고 이것은 머릿속으로도 충분히 그림이 그려지기 때문이다.

>1. 일단 팀들을 전부 조회한다 - 노란색 박스
>2. 1번 결과로부터 팀 엔티티별로 멤버들을 조회한다 - 초록색 박스

이렇게 JPA 입장에서 원하는 결과를 조회하는 데에는 문제가 없지만, 문제는 데이터베이스다.

위의 결과를 그저 딱 한 번의 쿼리로도 충분히 조회할 수 있기 때문이다.

```sql
select t.team_id, t.team_name, m.member_id, m.member_name from team t left join member m on t.team_id = m.team_id;
```

<img width="505" alt="leftjoin" src="https://github.com/user-attachments/assets/85aeab52-41b3-416b-a17c-54bf02ce759b">

쿼리 문법 중 **`join`** 문법을 활용하면 두 테이블의 데이터를 동시에 가져올 수 있어서 두 번의 테이블 스캔을 한 번으로 줄임으로써 한 번의 네트워크 호출로 원하는 결과를 반환할 수 있게 된다.

하지만 JPA는 **join**을 활용하지 않고 일일이 모든 테이블을 스캔하면서 `where`로 조건부 스캔을 중복하고 있다. 지금이야 팀이 3개밖에 안되지만 팀이 많아질 수록 이 중복되는 횟수 역시 비례해서 증가하게 될 것이다.

즉, N+1 문제라는 것은 **1번의 부모 조회** + **N번의 연관된 자식 조회**가 발생하는 이슈라고 할 수 있다.

#### (2) 왜 이렇게 설계됐는지 - 지연 로딩의 관점

이런 문제가 발생하게 된 배경에는 **지연 로딩**이 있다.

지연 로딩은 엔티티를 사용하는 시점까지 데이터 로드를 미루는 기법이다. 즉, 데이터베이스에서 즉시 데이터를 가져오지 않고 필요할 때 데이터를 로드하는 방식이다. 이를 통해 현재 시점에서 불필요한 데이터 로드를 방지하면서 초기 로딩 속도를 빠르게 할 수 있다.<br />

아까 `Team` 엔티티를 보면

```java
@Entity
@Getter
@NoArgsConstructor
public class Team {
    
    // ...
 
    @OneToMany(mappedBy = "team") // ~ToMany: LAZY
    private List<Member> members = new ArrayList<>();
    
    // ...
```

1:N 연관관계를 맺은 `Member` 엔티티 객체를 참조하고 있음을 알 수 있다.

즉, 만약 `TeamRepository`를 통해 `Team` 엔티티 인스턴스 객체를 조회하는 과정에서 `MemberRepository`에서 연관관계에 대응되는 해당 `Member` 엔티티 인스턴스들이 조회되면서 엔티티 내부에 있는 모든 정보들을 일괄적으로 함께 조회해야 되는 상황이 된다.<br />

이것을 구체화된 객체 참조가 아닌, **프록시 객체**로 대체해 최소한의 정보만을 제공하다가, `getter`, `toString()` 등을 통해 해당 필드의 구체적인 정보를 요구할 때 그제서야 쿼리를 발신하면서 실제 내용인 `List` 타입의 변수 내용이 반환된다.

>`~ToMany` 어노테이션은 지연 로딩이 디폴트고, `~ToOne` 어노테이션은 즉시 로딩이 디폴트다.
> 디버깅 과정에서 찰나의 순간에 *Collecting data...* 라는 문구가 확인되는데, 프록시 객체를 실제 객체로 가져오는 과정에서 표기되는 문구로 추정
> 
> <img width="956" alt="스크린샷 2024-12-07 오후 3 38 01" src="https://github.com/user-attachments/assets/8219542a-0cb3-499e-9710-7af83fcb0d79">

#### (3) N+1 문제에 대한 고찰

**지연 로딩의 관점**<br />
N+1 문제는 JPA, 나아가 ORM 설계상의 하자가 아니다. 지연 로딩을 통해 굳이 현시점에 호출이 불필요하나, 나중에 호출의 여지가 있는 연관관계 자식 데이터는 프록시 객체로 남겨뒀다가 호출할 때 실체화된 객체 호출 코드를 짬으로써 효율을 추구한다. 이게 곧 **지연 로딩**의 존재 의의라고 볼 수 있다.

다만, 상기의 내용을 우선하다보니 부모 데이터를 부르면서 동시에 자식 데이터의 내용들을 일괄 조회하려는 경우를 챙기지 못하는 것이다. 그렇기 때문에 fetch join 등이 N+1의 설계상 하자의 해결책이라기보다는 그저 개발 환경에 맞춰서 취사선택해야하는 전략 중 하나라는 관점이 옳을 것이다. 개발 상황에 따라 데이터 조회 전략이 다르기 때문이다.

**ORM의 관점**<br />
조금 더 생각해보자면, 객체지향에는 **참조**라는 개념이 있다. 하지만 데이터베이스에는 참조라는 개념이 없다. 그리고 ORM은 객체와 데이터베이스를 매핑하는 기술이다.

객체지향 주소를 바탕으로 참조값이 가리키는 객체를 알 수 있다. 즉, 해당 객체의 내부 내용(클래스의 필드나 메소드)들을 전부 끌고오지 않아도 주소만 알고 있으면 해당 객체로 넘어가서 구체화된 정보들을 확인할 수 있게 된다.

하지만 데이터베이스에는 참조라는 개념이 없이, 단순히 외래 키를 통해 해당 스키마의 필드가 어떤 부모 스키마의 필드에게 연관관계로써 종속됐는지 직접 확인(쿼리 호출)을 해야 되기 때문에 이것을 ORM 취지에 따라 객체와 데이터베이스 매핑을 이뤄내는 과정에서 생긴 이슈라고 볼 수 있다.

## 3. 해결 방안

### 1) 로딩 시점 변경 (x)

위의 설명에 따르면 지연 로딩이 마치 N+1 문제의 원인인 것처럼 보일 수 있다. 그렇다고 즉시 로딩으로 바꿔서 해결할 수 있을까?

```java
@Entity
@Getter
@NoArgsConstructor
public class Team {
    
    // ...
 
    @OneToMany(mappedBy = "team", fetch = FetchType.EAGER)
    private List<Member> members = new ArrayList<>();
    
    // ...
```

지연 로딩을 즉시 로딩으로 설정을 바꾸고 디버깅을 해서 쿼리 출력 과정을 살펴본다.

| 지연 로딩                                                                                                | 즉시 로딩                                                                                                                                    |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| <img width="1440" alt="지연로딩" src="https://github.com/user-attachments/assets/216a937c-0af9-464f-abab-f128309fe219"> | <img width="1440" alt="즉시로딩은N+1해결책x" src="https://github.com/user-attachments/assets/f625a61a-db30-4a5b-917e-50a427c9dfac"> |
                                                                                                                                                                                                                                             
보면 `Team` 엔티티 인스턴스의 내부 정보들을 조회해 올 때, 지연 로딩은 우선 `Team` 엔티티에 대응되는 team 스키마의 필드들을 호출하는 데에 그치지만, 즉시 로딩은 team 스키마 필드와 더불어 연관관계(데이터베이스 입장에서는 외래 키로써 부여된)를 맺고 있는 `Member` 엔티티 인스턴스들까지 한 번에 조회해 온다.

기존의 지연 로딩에서도 중복되는 쿼리 발신이 N+1 문제로 발생했었고 즉시 로딩이라고 나아지진 않는다. 결국 연관관계 필드의 로딩 시점을 조절하는 것만으로는 해결책이 될 수 없다.

### 2) 연관관계 방향 설정 (x)

이제까지 확인했던 예제는 전부 `Team` 엔티티의 시선에서 `Member` 타입 필드 리스트를 조회하는 경우였다. 이를 반대로 `Member` 엔티티 입장에서 `Team`을 조회해보자.

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Member {
    
    // ...

    @ManyToOne(cascade = CascadeType.ALL) // ~ToOne: EAGER
    @JoinColumn(name = "team_id") // 외래 키(팀)를 관리하는 측(멤버)에게 부여
    private Team team;

    // ...
```
```java
// TeamMemberService

public void findAllMembersByMemberRepo() {
    List<String> list = memberRepository.findAll().stream()
            .map(member -> member.getTeam().getName() + ": " + member.getName())
            .toList();

    System.out.println("결과: " + list);
}
```
<img width="989" alt="방향조회변경도 해결책이아니다" src="https://github.com/user-attachments/assets/ba11f2e7-dcd7-40f9-af51-34756dbc7e81">

애시당초 해결이 불가능하다. 이 방법은 오히려 쿼리의 `join` 문법 방향과 더 멀어지는 것이며, 모든 멤버를 조회하고(1) 특정 팀의 ID를 외래 키로 가진 멤버들을 조회(N)하는 방식이기 때문이다.

### 3) Fetch Join

#### (1) 적용

아까 데이터베이스에 직접 SQL을 날려서 한 번의 호출로 연결된 테이블을 조회함으로써 원하는 모든 데이터를 조회할 수 있었다. 이때 쓰인 문법이 `join` 문법이다.

```sql
select t.team_id, t.team_name, m.member_id, m.member_name from team t left join member m on t.team_id = m.team_id;
```

위의 SQL문과 유사한 JPA의 **JPQL(Java Persistence Query Language)** 를 활용해서 한 번의 호출로 예상 결과를 호출할 수 있다. 이 문법을 `Fetch Join` 이라고 한다.

```jpaql
SELECT t FROM Team t JOIN FETCH t.members
```
```java
@Repository
public interface TeamRepository extends JpaRepository<Team, Long> {
    @Query("SELECT t FROM Team t JOIN FETCH t.members")
    List<Team> findAllWithMembers();
}
```

<img width="989" alt="fetchjoin" src="https://github.com/user-attachments/assets/bbbe4638-3873-4862-aee7-631ac4a84f1a">

위처럼 쿼리 호출이 단 한 번으로 예상했던 결과가 호출돼서 로그에 찍히는 것을 확인할 수 있다.

#### (2) JDBC와의 차이점

여담이지만, JPQL을 쓰는 게 아닌 직접 SQL을 활용해서 JDBC로 해결할 수도 있다. 물론 이것은 hibernate가 작성하는 쿼리가 아니기 때문에 히카리풀 로그에 남지 않는다.

```java
@Repository
@RequiredArgsConstructor
public class TeamJdbcRepository {

    private final JdbcTemplate jdbcTemplate;

    public List<String> findAll() {
        String sql = "SELECT t.team_id, t.team_name, m.member_id, m.member_name " +
                "FROM team t " +
                "LEFT JOIN member m ON t.team_id = m.team_id";
        
        List<String> list = new ArrayList<>();

        jdbcTemplate.query(sql, rs -> {
            String teamName = rs.getString("team_name");
            String memberName = rs.getString("member_name");
            
            list.add(teamName + " :" + memberName);
        });

        return list;
    }
}
```

<img width="995" alt="jdbc 테스트" src="https://github.com/user-attachments/assets/6af18b52-4062-471c-b1f5-00de39a67a64">

다만 JPA를 활용하는 이상 JPQL을 기반으로 `Fetch Join`을 활용하는 것이 더 나아 보인다. JPA의 기능 중, 영속성 컨텍스트에 영속화하는 내용이 있는데 `Fetch Join`으로 조회된 연관관계는 영속성 컨텍스트의 1차 캐시에 저장되어 다시 엔티티 그래프를 탐색해도 조회 쿼리가 수행되지 않지만 그냥 JDBC로 `JOIN` 문법을 활용하는 것은 ORM과 무관하기 때문이다.

애시당초 ORM의 취지는 객체와 데이터베이스 간의 매핑을 통한 편의성 추구 및 패러다임 차이 간극을 줄이는 것이기 때문이다.

#### (3) 페이징 관련

`Fetch Join`이 완벽한 해결책은 아니다. 일련의 데이터들을 반환할 때 자주 사용되는 페이징을 같이 활용할 경우 문제가 발생할 수 있다.

```java
@Query("SELECT t FROM Team t JOIN FETCH t.members")
Page<Team> findAllWithMembers(Pageable pageable);
```

위와 같이 리스트가 아닌 페이징 객체를 반환하면서 JPQL을 같이 활용하는 것 자체에는 문법적으로나 실행 환경적으로나 문제가 없긴 하다. 다만 로그를 확인하면 기존의 결과에 더해 추가로 문구가 하나 더 뜬다.

<img width="958" alt="fetchjoin과 페이징 병용 - 아웃 오브 메모리 위험" src="https://github.com/user-attachments/assets/13483613-95ef-4c9a-b3bc-2b3428f1a7df">

```bash
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

이 문구의 의미는, **페이징 처리를 메모리에서 하고 있다**는 경고다. 왜 이런 문구가 발생하는지는 쿼리의 작동 의도와 실제 hibernate가 작성한 쿼리를 비교해야 한다.

먼저 일전의 그냥 `List` 반환과 `Page` 반환의 차이점부터 확인하자.

| 특징                  | List 반환                                            | 페이징(Pageable) 반환                                |
|-----------------------|-----------------------------------------------------|-----------------------------------------------------|
| **데이터 반환 방식**   | 데이터베이스에서 반환된 데이터를 그대로 메모리에 로드. | 데이터베이스에서 반환된 데이터를 메모리에 올린 후, 페이징 처리. |
| **페이징 처리 위치**   | 없음.                                               | 애플리케이션 레벨에서 메모리에 올린 데이터를 기준으로 페이징. |
| **메모리 사용량**      | 데이터베이스에서 반환된 데이터 크기와 동일.           | 데이터베이스에서 반환된 데이터 크기 + 페이징 처리로 인한 추가 메모리 사용. |
| **OOM 가능성**         | 데이터 크기가 너무 크면 발생 가능.                   | 데이터 크기가 크지 않아도 추가 연산으로 인해 발생 가능성이 높음. |
| **중복 데이터 처리**   | 데이터베이스에서 반환된 중복 데이터를 그대로 유지.     | JPA가 중복 데이터를 제거하며 메모리에 로드.            |
| **데이터 크기 제한**   | 데이터 크기를 제한하지 않으면 전체 데이터를 가져옴.    | Pageable을 적용했더라도 데이터베이스에서 전체 데이터를 가져옴(페이징 적용되지 않음). |

요약하자면, `List` 반환보다 `Page` 반환이 더 많은 메모리 사용량을 요구하는 것은 페이징 작업에 상당한 메모리 소비가 일어나기 때문이다.

애시당초 `JOIN` 작업은 상당한 메모리 소비를 일으키는데 그 이유는 부모 엔티티(`Team`) 기준만으로 조인 결과가 수행되지 않아서 그렇다. 자식 엔티티(`Member`)를 포함하기 때문에 부모 엔티티별 중복된 레코드가 반환된다.

여기서 Hibernate가 `Team`을 기준으로 중복 데이터를 제거하고 페이징을 시도하려 하는데 이때 전체 데이터를 메모리에 올려 처리하기 때문에 메모리 사용량이 급증하게 된다.

또한 페이징 사이즈에 따라 hibernate 쿼리가 둘 이상 출력되면서 N+1 문제 해결 목적과 어긋날 수 있다. 아래는 페이징 사이즈를 3 이하(부모 엔티티의 개수 이하)로 맞췄을 때의 로그다.

<img width="964" alt="엔티티수이하로페이징사이즈를정하면카운트쿼리발생" src="https://github.com/user-attachments/assets/9dacf36c-6f13-41f3-843c-aeb899679d72">

페이징 사이즈가 부모 엔티티의 개수와 같거나 작을 경우, Hibernate는 페이징 논리에 따라 `COUNT` 쿼리 + 데이터 조회 쿼리를 실행하게 된다. 그렇기에 굳이굳이 페이징을 사용한다면 부모 엔티티의 개수를 초과해서 페이징 사이즈를 정해야겠다만 원칙적으로 아웃 오브 메모리 예외가 발생할 수 있으니 권유하지 않는 것이다.

#### (3) 연속된 1:N 관계에서는 사용 불가

<img width="503" alt="연속된연관관계" src="https://github.com/user-attachments/assets/09cef5a3-e68d-42df-8462-79ff7bb366ea">

엔티티 `One`이 엔티티 `Two`와 1:N 관계를 맺고, 엔티티 `Two`가 엔티티 `Three`와 1:N 관계를 맺는다. 여기서 `One` 부모 엔티티를 기반으로 그냥 반환하면 당연히 N+1 문제가 발생하게 된다.

<img width="951" alt="연속N+1문제" src="https://github.com/user-attachments/assets/93a31fe9-21f9-4eea-adde-705fa24ae625">

연속된 연관관계에서 각각의 부모 엔티티의 개수만큼 해당 엔티티 조회 쿼리가 생기게 되는 것을 볼 수 있는데, 이걸 JPQL의 `Fetch Join`을 여러 번 활용하면 되겠다고 생각할 수 있지만, 연속된 1:N 관계가 맺어진 최상위 부모 엔티티에 대하여 사용할 경우 예외를 반환하게 된다.

```java
@Repository
public interface OneRepository extends JpaRepository<One, Long> {

    @Query("SELECT DISTINCT o FROM One o " +
            "LEFT JOIN FETCH o.twoList t " +
            "LEFT JOIN FETCH t.threeList th " +
            "ORDER BY o.id, t.id, th.id")
    List<One> findAllWithTwoAndThree();
}
```

<img width="979" alt="연속N+1페치조인문제발생" src="https://github.com/user-attachments/assets/862e23d9-69c8-42c6-9a37-0bcfd066c69d">

위의 사진처럼 JPA가 내부적으로 `MultipleBagFetchException`을 감싸서 `InvalidDataAccessApiUsageException`으로 변환하게 된다. 즉, 래퍼 예외를 반환시킨다. Hibernate가 두 개 이상의 컬렉션(`twoList`, `threeList`)을 동시에 조인하려고 하면 SQL의 Cartesian Product(데카르트 곱) 형태로 생성된 `ResultSet`에서 어떤 데이터가 `twoList`에 속하고 어떤 데이터가 `threeList`에 속하는지 명확히 구분할 수 없다.

#### (4) 복수의 1:N 관계에서는 사용 불가

<img width="501" alt="복수N+1" src="https://github.com/user-attachments/assets/08dcec0a-6b0f-467e-8570-86915a015c8e">

엔티티 `A`가 엔티티 `B`와 1:N 관계를 맺고, 엔티티 `A`가 엔티티 `C`와 1:N 관계를 맺는다. 여기서 `A` 부모 엔티티를 기반으로 그냥 반환하면 당연히 N+1 문제가 발생하게 된다.

<img width="963" alt="복수일대다관계문제발생" src="https://github.com/user-attachments/assets/bc0182f9-17fb-4098-b4a9-39a7761a1e3f">

이런 경우에도 JPQL의 `Fetch Join`을 여러 번 활용하면 되겠다고 생각할 수 있지만, 복수의 1:N 관계가 맺어진 부모 엔티티에 대하여 사용할 경우 예외를 반환하게 된다.

```java
@Query("SELECT a FROM A a " +
        "LEFT JOIN FETCH a.bList b " +   // A와 B를 FetchJoin
        "LEFT JOIN FETCH a.cList c")     // A와 C를 FetchJoin
List<A> findAllWithBAndC();
```

<img width="956" alt="복수의1대다관계의페치조인은사용불가" src="https://github.com/user-attachments/assets/c4c936ac-746c-45fb-b8a7-f8368e12528a">

위의 사진처럼 JPA가 내부적으로 `MultipleBagFetchException`을 감싸서 `InvalidDataAccessApiUsageException`으로 변환하게 된다. 즉, 래퍼 예외를 반환시킨다. Hibernate가 두 개 이상의 컬렉션(`bList`, `cList`)을 동시에 조인하려고 하면 중복된 결과가 생성될 수 있어서 이를 일부러 막는 것이다.

결국 `Fetch Join`이 N+1 문제의 완벽한 대응책은 아니다.

### 4) EntityGraph

#### (1) 적용

N+1 문제를 해결할 수 있는 다른 방법으로 `@EntityGraph` 어노테이션이 있다. 코드와 실행 결과를 확인해보자.

```java
@Repository
public interface TeamRepository extends JpaRepository<Team, Long> {
    @EntityGraph(attributePaths = "members") // 함께 조회하려는 연관관계 필드 명시
    List<Team> findAll();
}
```

<img width="954" alt="entitygraph" src="https://github.com/user-attachments/assets/1fa65bf3-9180-4fb1-a6db-12696d6932d2">

`@EntityGraph`의 경우 페치 타입을 Eager로 변환, 즉 즉시 로딩하는 방식으로 `outer left join`을 수행하여 데이터를 가져오지만, `Fetch Join`의 경우 따로 `outer join`으로 명시하지 않는 경우 `inner join`을 수행한다는 점에서 차이가 있다.

#### (2) 연속된 1:N 관계에서의 사용 불가

연속된 1:N 관계에서 사용할 수 없다.

<img width="965" alt="스크린샷 2024-12-08 오전 4 15 40" src="https://github.com/user-attachments/assets/eb17f26f-e5ef-4788-8a9d-bf8c7cbea87d">

#### (3) 복수의 1:N 관계에서는 사용 불가

역시 복수의 1:N 관계에서 사용할 수는 없다.

```java
@Repository
public interface ARepository extends JpaRepository<A,Long> {
    @EntityGraph(attributePaths = {"bList", "cList"})
    List<A> findAll();
}
```

<img width="985" alt="복수의일대다는엔티티그래프도불가능" src="https://github.com/user-attachments/assets/36e1f16f-6862-4ca1-8f4d-529e95a07bdf">

### 5) Batch Size 지정

부모 엔티티를 조회할 때 연관된 자식 데이터들까지 N번 조회하면서 추가적으로 발생하는 쿼리를 한 번에 묶어 실행하는 방식으로 동작한다. 여기서 `Batch Size`가 100으로 지정되면 최대 100개의 자식 엔티티를 한 번의 쿼리로 묶어서 조회한다.

<img width="960" alt="배치사이즈테스트" src="https://github.com/user-attachments/assets/8e84c05c-5ac3-48ae-8da7-094a83cde7dc">

위의 사진을 보면 쿼리가 2번 찍히는데, `Batch Size` 조정은 N에 해당하는 쿼리를 한 번으로 묶어주는 방식으로 동작하기 때문이다. 즉, 부모 엔티티를 조회하는 쿼리와 연관된 자식 엔티티들을 한 번에 불러모으는 쿼리를 실행해서 2번 찍히게 되는 것이다.

배치 사이즈 크기가 너무 크면 메모리 부담이, 너무 작으면 효율성이 떨어지기 때문에 적당한 값 선택이 필요하며 기본적으로 지연 로딩에 영향을 미치므로 즉시 로딩 전략에서는 효과를 기대하기 어렵다.

### 6) 트래픽 비교 테스트

아무런 조치를 취하지 않은 케이스, Fetch Join 케이스, EntityGraph 케이스, Batch Size 케이스까지 총 4가지에 대하여 일전의 `Team` 엔티티 기반으로 team 스키마와 member 스키마에서 리스트 내용을 조회하는 트래픽 테스트를 수행해본다. 테스트 조건 및 환경은 아래와 같다.

>- 테스트 툴: Apache JMeter
>- 가상 사용자수: 10000
>- 램프 업 타임: 10
>- 루프 카운트: 1

#### (1) 기본 케이스

<img width="836" alt="그냥" src="https://github.com/user-attachments/assets/e05cbdcc-76bc-4c0b-bd29-07ec5eccb9e2" />
<img width="49%" src="https://github.com/user-attachments/assets/28751609-33a1-46d6-9106-c601a39a695d" />
<img width="49%" src="https://github.com/user-attachments/assets/5257a1b4-bd04-42d4-9a22-cfd4285d72c4" />

#### (2) Fetch Join 케이스

<img width="840" alt="페치조인" src="https://github.com/user-attachments/assets/f427a62b-6b3f-4f6c-9a71-70f4d6323298" />
<img width="49%" src="https://github.com/user-attachments/assets/02f4e9a4-9914-47c4-93b4-98ff0ac8f9fd" />
<img width="49%" src="https://github.com/user-attachments/assets/e7d5c710-9d19-49d0-a2a6-244ccf71114c" />

#### (3) EntityGraph 케이스

<img width="839" alt="엔티티그래프" src="https://github.com/user-attachments/assets/b88d4d06-78af-4051-99ae-3c5b4ce50daa" />
<img width="49%" src="https://github.com/user-attachments/assets/9f4f64f6-ffbe-4a82-8d7e-e581008959e7" />
<img width="49%" src="https://github.com/user-attachments/assets/b52f73f2-2da0-472a-82da-cce378fdf709" />

#### (4) Batch Size 케이스

<img width="836" alt="배치사이즈" src="https://github.com/user-attachments/assets/b4ee6f3c-e4c0-4f4b-b29d-0bd386c557b9" />
<img width="49%" src="https://github.com/user-attachments/assets/9fca8b3f-e336-4f37-b814-ba3d408a4ab1" />
<img width="49%" src="https://github.com/user-attachments/assets/ebc0f064-bf18-4fea-bdc4-6e5a41a5da28" />


## 4. 결론

단일 부모 및 자식 엔티티 관계에서의 N+1 문제 해결책으로써는 적정하나, 복합적인 1:N 관계를 일괄 조회 처리하는 데에는 무리가 있는 해결책들로 보이므로 이 경우에는 JDBC를 활용하거나 QueryDSL 등을 활용하여 커스텀 쿼리를 작성해 활용하는 것이 올바른 해결책으로 작동할 것으로 보인다.

트래픽 제어의 관점에서는 N+1 문제 역시 대용량 트래픽 상황에서 성능 저하를 일으키는 요소 중 하나인 것을 확인했으며, 비즈니스 로직이 간편한 케이스에서 Fetch Join이 가장 높은 처리량을 보였다. 다만 지연시간 관점에서는 유의미한 결과가 보이지 않아서 실제 복잡한 케이스에서의 세부적인 비교가 필요한 것으로 보인다.

---

*스터디 레코드라 틀린 부분이 있을 수 있음*
