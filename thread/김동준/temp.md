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

        try (ExecutorService virtualService = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 5; i++) {
                virtualService.submit(runnable);
            }
        }
    }

    private static Runnable getRunnable() {
        AtomicLong index = new AtomicLong();  // 인덱스 원자적 연산
        int count = 100;  // 각 태스크 당 작업 100번 실행
        CountDownLatch countDownLatch = new CountDownLatch(count);  // 100번 완료될 때까지 대기

        return () -> {
            try {
                long indexValue = index.incrementAndGet();
                Thread.sleep(1000L);
                System.out.println("\n스레드 명칭: " + Thread.currentThread().getName() + "\n인덱스: " + indexValue);

                countDownLatch.countDown();
            } catch (final InterruptedException e) {
                countDownLatch.countDown();
            }
        };
    }
}
```

플랫폼 스레드와 가상 스레드 둘 다 `ExecutorService`를 활용한 스레드 풀로 복수의 스레드를 생성한다.<br />
각 태스크는 스레드 당 100번씩 수행되며, 작업 내용에는 인덱스의 원자적 증가연산과 해당 스레드의 이름 로깅이 들어있다.<br />
코드를 작성해서 실행한 후, 로깅과 디버깅을 해본다.<br />




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
