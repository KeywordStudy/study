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


