# 가상 스레드의 도입

## 1. 필요성

JDK 21 이후의 오라클 공식 문서에서는 플랫폼 스레드에 대하여 다음과 같이 나와있다.

>A platform thread is implemented as a thin wrapper around an operating system (OS) thread. A platform thread runs Java code on its underlying OS thread, and the platform thread captures its OS thread for the platform thread's entire lifetime. Consequently, the number of available platform threads is limited to the number of OS threads...
>
>Platform threads typically have a large thread stack and other resources that are maintained by the operating system. Platform threads support thread-local variables...
>
>Platforms threads are suitable for running all types of tasks but may be a limited resource...
>
>*출처 : https://docs.oracle.com/en/java/javase/20/core/virtual-threads.html#GUID-15BDB995-028A-45A7-B6E2-9BA15C2E0501*

요악하자면 **커널 스레드(실제 동작하는 운영체제가 할당한 스레드)와 1대 1로 매핑된 자바 스레드 객체**인 플랫폼 스레드의 한계에 대해 설명하고 있다.<br />
스케줄러의 컨텍스트 스위치 처리 비용이 높고, 생성 과정에서 메모리 소모값이 많기 때문에 애플리케이션이 플랫폼 스레드에 의존하면 OS에 가해지는 부담으로 인해 애플리케이션의 성능에도 이슈가 생길 수 있다고 공식 문서에서 지적하고 있다.

## 2. 보완 방향

![image](https://github.com/user-attachments/assets/507c44d4-68cf-4e8f-a228-2bcf9a093fb7)
플랫폼 스레드의 근본적인 문제점은 **운영체제의 실재하는 커널 스레드에 종속**된다는 점이다. 정확히는 1대 1로 매핑되는 점이다.<br />
운영체제가 각각 관리하는 커널 스레드의 비용이 그대로 플랫폼 스레드에도 전이되면서 비례적으로 급증하는 것을 확인할 수 있다.

다만 자바에서의 스레드 모델을 활용하려면 커널 스레드를 생성해야 된다는 점은 변함이 없기 때문에 보완 방향을 다음처럼 잡을 수 있다.

>1. 커널 스레드로 인해 전이되는 생성 소모 비용을 줄인다.
>2. 1대 1 매핑 구조를 극복한다.
>3. 커널 스레드 1개에 여러 개의 자바 스레드 모델을 매핑한다. 즉, **1대 N 매핑 구조**를 채택한다.

이렇게 보완을 한다면 생성되고 관리되는 커널 스레드 비용당 자바 스레드 모델 비용을 줄임으로써 성능을 향상시킬 수 있게 된다.


# 가상 스레드의 구조

## 1. 가상 스레드의 구성

그럼 어떻게 1대 1에서 1대 N으로 극복할 수 있을까? 핵심은 바로 **스케줄러**다.<br />
기존 플랫폼 스레드와 매핑된 커널 스레드의 스케줄링은 운영체제에서 담당했다. <br />
그리고 앞서 말했듯, 스케줄러의 컨텍스트 스위칭 비용이 성능 저하의 큰 축을 차지하고 있다.<br />

이것을 가상 스레드에서는 **JVM이 스케줄링 책임**에 대하여 비중을 부여하고 높이는 방향으로 보완됐다.<br />
물론 OS의 스케줄링 역시 커널 스레드를 관리하기 위해 존재하지만, JDK 21에서는 가상 스레드의 실행 기반으로 책임이 약화됐다.<br />
즉, 가상 스레드의 스케줄링은 JVM의 `ForkJoinPool` 같은 스케줄러가 담당하도록 보완됐다.<br />

![image](https://github.com/user-attachments/assets/31a65a3f-c84c-4536-81c9-e077b4231069)

## 2. 가상 스레드의 메커니즘

실제 코드를 작성하며 플랫폼 스레드와 가상 스레드의 애플리케이션 레벨에서의 구조를 파악해보자.<br />

```java
public class StructureTest {

    public static void main(String[] args) {

        Runnable runnable = getRunnable();

        try (ExecutorService platformService = Executors.newFixedThreadPool(5)) {
            for (int i = 0; i < 5; i++) {
                platformService.submit(runnable);
            }
        }

        System.out.println();

        try (ExecutorService virtualService = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 5; i++) {
                virtualService.submit(runnable);
            }
        }
    }

    private static Runnable getRunnable() {
        AtomicLong index = new AtomicLong();  // 인덱스 원자적 연산

        return () -> {
                long indexValue = index.incrementAndGet();
                try {
                    Thread.sleep(1000L);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } finally {
                    System.out.println("스레드 명칭: " + Thread.currentThread().getName() + "\n인덱스: " + indexValue);
                }
        };
    }
}
```

플랫폼 스레드와 가상 스레드 둘 다 `ExecutorService`를 활용한 스레드 풀로 복수의 스레드를 생성한다.<br />
각 태스크는 스레드 당 100번씩 수행되며, 작업 내용에는 인덱스의 원자적 증가연산과 해당 스레드의 이름 로깅이 들어있다.<br />
코드를 작성해서 실행한 후, 로깅과 디버깅을 해본다.<br />

### (1) 로깅

<img width="1116" alt="로깅" src="https://github.com/user-attachments/assets/07227c1a-7a69-46eb-9de4-a2c1b25889d7">

시도할 때마다 인덱스의 순서는 다르지만 원자적 연산으로 안전하게 10까지 인덱스가 증가하게 된다.<br />
여담으로 인덱스 변수가 메소드 내의 지역변수임에도 스레드 안전하게 동작할 수 있는 이유는 `main()` 메소드에서 하나의 `Runnable` 객체를 스레드들이 공유하기 때문이다.<br />
즉, 모든 스레드가 동일한 `AtomicLong` 인스턴스 변수를 사용하기 때문에 스레드 개수에 맞춰서 인덱스 증가연산이 이뤄질 수 있다.

아무튼 로깅을 확인해보면 `newFixedThreadPool(5)`로 생성한 스레드는 스레드 이름이 찍히지만, `newVirtualThreadPerTaskExecutor()`는 이름이 없다.<br />
그 이유는 자바 공식문서에 나와있는데...

>Virtual threads do not have a thread name by default. The getName method returns the empty string if a thread name is not set.
>
>Virtual threads are daemon threads and so do not prevent the shutdown sequence from beginning. Virtual threads have a fixed thread priority that cannot be changed...
>
>*출처 : https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html*

그냥 디폴트로 이름을 세팅하지 않는다고 한다.

### (2) 디버깅

이제 디버깅을 해본다. 우선 플랫폼 스레드풀 서비스부터 디버깅을 해보자.

<img width="1312" alt="스크린샷 2024-12-01 오후 4 19 09" src="https://github.com/user-attachments/assets/974ce5ca-aa50-4a53-a460-993d0824efd2">

로깅에서 봤던 스레드 이름은 클래스 내부 필드 참조 객체인 `threadFactory`에서 명명되는 것을 확인할 수 있다.<br />
또한, `HashSet` 구조를 가진 `workers` 객체에서 각 스레드들이 생명주기를 가지고 있다.

<img width="1328" alt="스크린샷 2024-12-01 오후 4 34 08" src="https://github.com/user-attachments/assets/d730db81-4094-43b3-bfb6-aa1f293a7123">

가상 스레드풀 서비스 디버깅에서 확인할 참조 객체는 `scheduler`, `runContinuation`, `carrierThread`이다

아까 가상 스레드는 JVM의 스케줄러에 의해 관리된다고 했는데 디버깅에 따르면 `ForkJoinPool` 타입의 객체를 참조하고 있다.<br />
`scheduler`가 가상 스레드의 작업을 실행하기 위한 스케줄러 참조값이며, JVM에서 관리되는(스레드 매핑 역할을 맡는) 플랫폼 스레드가 담당한다.

`runContinuation` 객체는 중단 작업의 재개 시점을 관리하고 있다. 구체적으로 가상 스레드의 실행 흐름을 캡슐화한다.<br />
객체의 내부에는 실제 가상 스레드가 수행할 작업 내용이 정의되어 있고 재개점을 호출하기 때문에 재귀 구조로 객체 참조가 이뤄진다.

그리고 해당 가상 스레드를 실제로 실행하는 플랫폼 스레드가 바로 `carrierThread` 참조 객체에 지정되어 있다.<br />
가상 스레드가 실행될 때 어떤 플랫폼 스레드가 이를 처리하는 지에 대한 정보가 명시되어 있다.

### (3) 결론

종합하자면, 기존의 커널 모드 전환 비용을 지불하고 컨텍스트 스위칭이 이뤄진 것과 비교했을 때, JVM의 관리 하에서 `Continuation`을 활용하여 호출 스택의 상태 복원이 이뤄지기 때문에 커널 모드 전환 비용이 절감된다. 또한, 플랫폼 스레드의 고정 스택 크기(1 MB)에 비해 가상 스레드는 약 1~2 KB밖에 되지 않는 크기의 스택을 사용해서 스레드 수가 급증해도 부담이 덜하다.




<div style="display: flex; justify-content: space-between;">
    <img src="https://github.com/user-attachments/assets/e53cd81b-b323-4ad5-80d8-a2b40c528ed7" alt="Image 1" style="width: 45%;"/>
    <img src="https://github.com/user-attachments/assets/d09f64de-8814-48a4-b81b-3525f97f1bec" alt="Image 2" style="width: 45%;"/>
</div>

또한, 플랫폼 스레드의 비동기 작업에는 `CompletableFuture` 같은 비동기 API가 필요했으나 가상 스레드는 동기식 코드로도 비동기 처리가 가능하며 입출력 같은 블로킹 작업이 발생해도 가상 스레드가 운영체제의 커널 스레드를 점유하지 않아도 되므로 운영체제의 스레드 낭비가 줄어들게 된다. 이는 플랫폼 스레드가 블로킹 처리에서 대기 상태로 강제 진입하므로 리소스 낭비가 발생되는 부분과 대비되는 점이다.



비교

플랫폼 스레드
- `newFixedThreadPool(5)`은 5개의 고정된 플랫폼 스레드를 생성하고 유지
- 작업이 제출될 때, 스레드풀에서 이미 생성된 스레드 중 하나가 작업을 처리
- 만약 작업이 많아 5개 이상의 작업이 제출되면, 대기열에 추가
- 스레드는 운영 체제의 리소스를 직접 사용하며, 비용이 높음

가상 스레드
- `newVirtualThreadPerTaskExecutor()`는 작업당 새로운 가상 스레드를 생성
- 가상 스레드는 JVM이 관리하며, OS 수준의 리소스를 직접 사용하지 않고 더 가볍게 작동
- 작업이 제출되면 각 작업마다 새로운 가상 스레드가 생성되어 실행(스레드풀처럼 보이지만 실제로는 가상 스레드 하나당 하나의 작업을 처리)
- 내부적으로 가상 스레드는 JVM이 스레드 스케줄링 및 실행 관리를 효율적으로 처리



*출처*<br />
*https://ride-wind.tistory.com/119*<br />
*https://perfectacle.github.io/2022/12/29/look-over-java-virtual-threads/*<br />
*https://stackoverflow.com/questions/76180420/thread-currentthread-getname-returns-empty-string-for-virtual-threads-cre*<br />
*https://velog.io/@on5949/%EC%8A%A4%ED%84%B0%EB%94%94%EC%84%B8%EB%AF%B8%EB%82%98-Java%EC%9D%98-%EB%AF%B8%EB%9E%98-Virtual-Thread*
