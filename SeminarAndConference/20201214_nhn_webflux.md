# 내가 만든 WebFlux가 느렸던 이유

> NHN FORWARD 2020 - 김병부
>
> WebFlux로 만든 API 서버의 성능 측정 결과와 성능 향상에 관한 이야기
>
> Spring WebFlux와 Lettuce 성능 측정 결과

## 시작하기 전에

* 일반적으로 WebFlux로 만든 애플리케이션이 웹 MVC로 만든 애플리케이션 보다 처리량이 높은 것으로 알려져 있다.
* 하지만, 코딩을 잘못하게 되면 WebFlux가 오히려 처리량이 낮아진다.
* 비동기, Non-blocking으로 개발해야지 높은 처리량을 기대할 수 있다는 것은 누구나 아는 사실이다.
  * 비동기, Non-blocking으로 인한 성능저하 케이스들과 추가적인 성능 저하 케이스를 알아 본다.

## 소개

* 필자는 NHN Global에서 패션고라는 B2B서비스를 운영하고 있고, 그중 광고 시스템을 개발하고 있다.
* 이 광고 서비스는 빠른 응답과 다수의 요청을 처리할 수 있는 높은 처리량을 요구한다. 사용자에게 100ms이상 응답이 걸리는 것은 효과적이지 않은 광고 노출로 판단하여 내부적으로 timeout 처리한다.
* 패션고 곳곳에서 광고가 노출되고 있으므로 패션고 전체 request보다 요청량이 많다. 대략 3배정도..
* 광고 서버는 빠른 응답과 높은 처리량이 필요했으므로 저장소는 redis, API는 Spring Webflux가 좋은 선택지로 보였다. 차차 알아보자.

## WebFlux를 선택한 이유

### Spring MVC

* **Thread Per Request Model**
  * 서버는 request를 처리하기 위해서 ThreadPool에 있는 스레드 하나를 이 request에 할당한다.
  * 할당된 스레드는 요청을 받고 응답할 때까지 모든 처리를 담당하게 된다.
  * 응답을 마치면 스레드는 다시 ThreadPool에 들어간다.
* SpringBoot의 Auto Configuration에서는 Tomcat의 기본 스레드 개수 설정이 200개이다.
* 분산시스템으로 설계된 서비스는 다른 API 서버의 Rest API를 호출하여 데이터를 통합하는 경우가 매우 빈번하다.
* 내 서비스에서 호출한 다른 서비스의 API의 처리시간이 길어진다면, 그 시간만큼 내 서비스는 Block 된다.
* 부하가 적은 상태에서는 문제없이 동작하겠지만, 시스템의 부하가 많은 상태에서 스레드의 상태가 Wating에서 Runnable로 바뀌는 것. 즉, Context Switching이 되고, 스레드에 계속해서 데이터를 로딩하는 오버헤드가 문제가 된다.
* 또한, 톰캣의 기본 스레드 개수는 200개인데, 200개의 스레드가 얼마되지 않는 CPU 코어를 점유하기 위해서 경합하는 현상도 발생한다. 대부분의 개발자들이 사용하는 머신 코어의 수는 2개 또는 4개 많아봐야 8개 정도 된다. 200개의 스레드가 8개의 코어를 점유하기 위해 경합한다면 이것 또한 큰 부하가 될 수 있다.

### Spring WebFlux

* **EventLoop Model**
  * 사용자들의 요청이나 애플리케이션 내부에서 처리해야 하는 작업들은 모두 이벤트라는 단위로 관리되고, 이벤트 큐에 적재되어 순서대로 처리되는 구조다.
  * 이벤트를 처리하는 스레드 풀 존재한다. 이 스레드 풀은 순차적으로 이벤트를 처리하기 때문에 이벤트 루프라고도 한다. 이벤트 루프는 이벤트 큐에서 이벤트를 뽑아서 하나씩 처리한다.
  * Spring WebFlux는 Reactor 라이브러리와 Netty를 기반으로 동작한다.
  * Netty의 경우 Tomcat과는 다르개 Thread 개수가 **Core의 개수 * 2** 개이다.
* Spring MVC처럼 I/O가 끝날때까지 스레드가 Block 되어 대기하지 않고, I/O가 시작되기 전 작업도 이벤트, I/O가 끝난 후의 작업도 이벤트로 만들어져 이벤트 큐에 들어간다.
* NIO를 이용하여 I/O작업을 처리하기 때문에 스레드 상태가 Block되지 않는다. 때문에 Thread Per Request Model 보다 스레드의 Context Switch 오버헤드가 줄어들고, 스레드 수도 작기 때문에 CPU 코어를 차지하기 위한 경합도 줄어든다.
* **높은 처리량을 가질 수 있는 이유**
  * Non-Blocking IO와 이벤트 루프를 사용하기 때문이다.
  * 이벤트 루프의 스레드를 일하는데만 집중해서 성능을 쥐어짜기 때문이다.
* **성능 저하의 원인**
  * WebFlux는 상대적으로 적은 스레드 풀을 유지하기 때문에 CPU 사용량이 높은 작업이 많거나, Blocking IO를 이용한다면 이벤트 루프가 이벤트 큐에 있는 이벤트를 빠르게 처리할 수 없다.
  * Runnable 상태의 스레드가 CPU를 점유하고 있기 때문이다.
  * 예를 들면, 동영상을 인코딩한다거나 암호화 모듈을 가지고 암호화/복호화 하는 작업에는 WebFlux가 적절하지 않을 수 있다.