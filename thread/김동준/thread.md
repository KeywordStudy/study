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

![image](https://github.com/user-attachments/assets/c701cb36-02e3-4fb1-9439-2dbd4bfec16a)

위 그림의 Carrier Thread2에서 Cont2 작업이 큐에 진입한다. 그 과정에서 Cont1 작업이 블로킹 처리됐다고 가정하자.<br />
기존의 플랫폼 스레드는 블로킹이 되면 커널 스레드 자체를 중단하는 반면, 위의 Cont1 작업이 블로킹 처리됐을 때, **yield** 처리가 되는 걸 볼 수 있다.<br />
즉, 작업이 중단되어도 다른 작업인 Cont2를 처리하고 그 동안 Cont1은 잠시 (스레드 개념의 대기가 아닌, 양보의 의미의) 대기 상태로 진입한다.

![image](https://github.com/user-attachments/assets/cd5479a7-de33-44ab-a229-558ebbdf1c78)

스레드가 대기 상태로 진입할 때 기존 플랫폼 스레드는 `Unsafe`의 네이티브 메소드인 `park()`를 호출하면서 대기상태로 진입한다.<br />
가상 스레드 로직의 메소드를 타고 들어가보면 `VirtualThread`의 `park()` 메소드 내부에서 `private` 메소드인 `yieldContinuation()`을 호출하는데, 이 내부에 다른 스레드에게 제어권을 넘기는 로직이 구성되어 있다.

```java
final class VirtualThread extends BaseVirtualThread {

    // ...

    @Hidden
    @ChangesCurrentThread
    private boolean yieldContinuation() {
        // unmount
        // 현재 스레드 상태 변경(언마운트)
        notifyJvmtiUnmount(/*hide*/true);
        unmount();
        try {
            // 가상 스레드 실행 양보(실행 범위 : VTHREAD_SCOPE)
            return Continuation.yield(VTHREAD_SCOPE);
        } finally {
            // re-mount
            // 재실행하여 상태 복원(마운트)
            mount();
            notifyJvmtiMount(/*hide*/false);
        }
    }
```

이런 과정들을 거치며 해당 가상 스레드를 실제로 실행하는 플랫폼 스레드가 바로 `carrierThread` 참조 객체에 지정되어 있다.<br />
가상 스레드가 실행될 때 어떤 플랫폼 스레드가 이를 처리하는 지에 대한 정보가 명시되어 있으며, 타입이 `Thread`로 되어있는 것을 볼 수 있다.

### (3) 결론

종합하자면, 기존의 커널 모드 전환 비용을 지불하고 컨텍스트 스위칭이 이뤄진 것과 비교했을 때, JVM의 관리 하에서 `Continuation`을 활용하여 호출 스택의 상태 복원이 이뤄지기 때문에 커널 모드 전환 비용이 절감된다. 또한, 플랫폼 스레드의 고정 스택 크기(1 MB)에 비해 가상 스레드는 약 1~2 KB밖에 되지 않는 크기의 스택을 사용해서 스레드 수가 급증해도 부담이 덜하다.

<div style="display: flex; justify-content: space-between;">
    <img src="https://github.com/user-attachments/assets/e53cd81b-b323-4ad5-80d8-a2b40c528ed7" alt="Image 1" style="width: 45%;"/>
    <img src="https://github.com/user-attachments/assets/d09f64de-8814-48a4-b81b-3525f97f1bec" alt="Image 2" style="width: 45%;"/>
</div>

또한, 플랫폼 스레드의 비동기 작업에는 `CompletableFuture` 같은 비동기 API가 필요했으나 가상 스레드는 동기식 코드로도 비동기 처리가 가능하며 입출력 같은 블로킹 작업이 발생해도 가상 스레드가 운영체제의 커널 스레드를 점유하지 않아도 되므로 운영체제의 스레드 낭비가 줄어들게 된다. 이는 플랫폼 스레드가 블로킹 처리에서 대기 상태로 강제 진입하므로 리소스 낭비가 발생되는 부분과 대비되는 점이다.

### (4) 코드 기반 실행시간 비교

```java
public class StructureTest {

    public static void main(String[] args) {
        Runnable runnable = getRunnable();

        // 기존 스레드풀 사용
        long startTime = System.nanoTime();
        try (ExecutorService platformService = Executors.newFixedThreadPool(100)) {
            for (int i = 0; i < 10_000; i++) {
                platformService.submit(runnable);
            }
        }
        long endTime = System.nanoTime();
        long durationPlatform = endTime - startTime;
        System.out.println("플랫폼 스레드풀 실행 시간: " + durationPlatform / 1_000_000 + " ms");

        System.out.println();

        // 가상 스레드풀 사용
        startTime = System.nanoTime();
        try (ExecutorService virtualService = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10_000; i++) {
                virtualService.submit(runnable);
            }
        }
        endTime = System.nanoTime();
        long durationVirtual = endTime - startTime;
        System.out.println("가상 스레드풀 실행 시간: " + durationVirtual / 1_000_000 + " ms");
    }

    private static Runnable getRunnable() {
        return () -> {
            for (long i = 0; i < 1000; i++) {
            }

            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        };
    }
}
```

<img width="715" alt="스크린샷 2024-12-01 오후 7 03 24" src="https://github.com/user-attachments/assets/91a79d6b-980b-42b0-9087-e3f1cb4b6562">


| **특징**     | **플랫폼 스레드 (`newFixedThreadPool(N)`)**                                    | **가상 스레드 (`newVirtualThreadPerTaskExecutor()`)**                    |
|--------------|-------------------------------------------------------------------------------|------------------------------------------------------------------------|
| **스레드 수** | 고정된 N개의 플랫폼 스레드를 생성하고 유지                                     | 각 작업에 대해 새로운 가상 스레드를 생성                               |
| **작업 처리** | 제출된 작업은 이미 생성된 스레드에서 처리                                      | 각 작업마다 새로운 가상 스레드가 생성되어 처리                        |
| **리소스 관리** | 운영 체제의 리소스를 직접 사용, 비용이 큼                                       | JVM이 관리하며 OS 수준의 리소스를 직접 사용하지 않음, 더 가볍게 작동 |
| **스케줄링**  | 운영 체제에서 직접 스레드 스케줄링을 처리                                       | JVM이 스레드 스케줄링 및 실행 관리를 효율적으로 처리                  |
| **대기열**    | 작업이 많으면 대기열에 추가                                                     | 각 작업에 대해 새로운 가상 스레드가 생성되므로 대기열이 없음         |

# 가상 스레드 추가 정리

## 1. 기존 코드와의 통합

가상 스레드에 대한 매커니즘은 `VirtualThread`라는 별도의 상속 불가 클래스에 정의되어 있다.<br />
그렇지만 가상 스레드를 실행하는 방법으로 주어지는 대표적인 3가지를 보면 의아한 점이 있다.

```java
// 태스크 정의
Runnable runnable = getRunnable();

// 1
Thread virtualThread1 = Thread.startVirtualThread(runnable);

// 2
Thread.Builder builder = Thread.ofVirtual();
Thread virtualThread2 = builder.start(runnable);

// 3 (ExecutorService 기반)
try (ExecutorService virtualThreads = Executors.newVirtualThreadPerTaskExecutor()) {
    virtualThreads.submit(runnable);
}
```

보면 기존에도 존재했던 `Thread` 객체 혹은 `ExecutorService`를 기반으로 가상 스레드가 생성된다.<br />
이렇게 설계된 이유는 기존 코드와의 통합성을 위해서다. 소위 말하는 **다형성**을 활용하여 기존 클래스 범위에서 충분히 커버할 수 있다.<br />

```java
sealed abstract class BaseVirtualThread extends Thread

final class VirtualThread extends BaseVirtualThread
```

## 2. 가상 스레드와 관련된 유의점, 고려할 점

#### 캐리어 스레드 블로킹
가상 스레드는 캐리어 스레드에서 실행되기 때문에, 캐리어 스레드가 블로킹되면 전체 시스템 성능에 영향을 미칠 수 있다. 예를 들어, 사용 중인 자원에 접근하는 것을 블로킹하는 `synchronized`나 공유 자원 동기화 등으로 블로킹이 발생할 수 있는 `parallelStream`이 영향을 받을 수 있는 가상 스레드를 사용할 때 주의해야 한다.<br />
그래서 `ReentrantLock`을 사용하여 동기화 문제를 해결하는 것이 권장된다. `ReentrantLock`은 더 세밀한 동기화 및 성능 최적화를 제공하며, 가상 스레드의 장점을 최대한 활용할 수 있다.

#### 풀링 사용 x
가상 스레드는 생성 비용이 낮아 매번 새로운 스레드를 생성하고, 사용 후 GC가 자동으로 처리한다. 따라서 스레드풀을 사용하여 개수에 제한을 두는 방식은 필요하지 않기 때문에 `ExecutorService` 기반 생성에서 파라미터가 필요 없다.

#### CPU 집약적인 작업
가상 스레드는 논블로킹 입출력 작업에 최적화되어 있지만, CPU 집약적인 작업에서는 성능이 떨어질 수 있다.<br /> 
만약 CPU 바운드 작업을 가상 스레드로 처리하면, Carrier Thread 위에서 실행되므로 성능 낭비가 발생할 수 있기 떄문에 CPU를 많이 사용하는 작업은 가상 스레드 대신 일반 스레드를 사용하는 것이 더 효율적일 수 있다.

> CPU 집약적인 작업 : CPU 성능에 의존하는 작업(ex) 복잡한 수학 연산, 이미지나 비디오 처리)

#### ScopedValue (Thread Local 대체)
가상 스레드에서는 Java에서 각 스레드가 독립적인 값을 저장할 수 있도록 제공되는 클래스인 `ThreadLocal`을 사용하기 어렵거나 성능 저하를 초래할 수 있어서 이를 대체하기 위해 JDK 21에서 `ScopedValue`라는 새로운 기능이 도입됐디.<br /> 
`ScopedValue`는 ThreadLocal의 대안으로, 스레드 범위에서 값의 전달을 효율적으로 처리할 수 있고 가상 스레드와 잘 통합되어, 동시성 처리에서 더욱 효율적인 상태 관리가 가능하다.

#### 가상 스레드 vs 코틀린 코루틴(Coroutine)
**가상 스레드**는 스레드 단위로 동작하고, **Coroutine**은 메소드 단위로 동작한다.<br /> 
가상 스레드는 스레드 스케줄링을 직접 처리하지만, 코루틴은 메소드 실행을 중단/재개할 수 있어 더 작은 단위에서 높은 동시성을 처리할 수 있고 특정 작업을 종료하거나 캔슬할 수 있기 때문에, 더 세밀한 제어가 가능하다.

#### 가상 스레드 vs WebFlux
**WebFlux**는 배압(BackPressure)을 지원하는데, 이는 가상 스레드에서는 지원되지 않는다.<br />
배압을 처리하는 방법은 세마포어를 사용하거나 애플리케이션 코드에서 처리해야 하는데, WebFlux는 비동기 흐름을 잘 처리하며 배압 기능을 지원하지만 가상 스레드는 이와 같은 기능이 부족하기 때문에 애플리케이션 레벨에서 배압을 처리해야 한다.

>Backpressure은 Publisher가 끊임없이 emit하는 무수히 많은 데이터를 적절하게 제어하여 데이터 처리에 과부하가 걸리지 않도록 제어하는 것

#### 가상 스레드 & MySQL JDBC Driver
MySQL JDBC 드라이버는 아직 가상 스레드를 지원하는 PR이 진행 중이라서 지원이 되지 않았으나 병합이 이뤄진 듯하다.<br />
나중에 코드로 확인해봐야겠다.

>*참고*<br />
>*https://bugs.mysql.com/bug.php?id=110512*<br />
>*https://github.com/mysql/mysql-connector-j/commit/00d43c5e8b24f1d516f93eea900b3487c15a489c*


---
*출처*<br />
*https://ride-wind.tistory.com/119*<br />
*https://perfectacle.github.io/2022/12/29/look-over-java-virtual-threads/*<br />
*https://stackoverflow.com/questions/76180420/thread-currentthread-getname-returns-empty-string-for-virtual-threads-cre*<br />
*https://velog.io/@on5949/%EC%8A%A4%ED%84%B0%EB%94%94%EC%84%B8%EB%AF%B8%EB%82%98-Java%EC%9D%98-%EB%AF%B8%EB%9E%98-Virtual-Thread*
