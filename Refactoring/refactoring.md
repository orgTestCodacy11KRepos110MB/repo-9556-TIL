# Refactoring with Java

> 백기선

## Smell 1. 이해하기 힘든 이름

- 깔끔한 코드에서 가장 중요한 것 중 하나가 바로 "**좋은 이름**" 이다.
- 함수, 변수, 클래스, 모듈 이름이 모두 어떤 역할을 하는지 어떻게 쓰이는지 직관적이어야 한다.
- 사용할 수 있는 리팩토링 기술
  - 함수 선언 변경하기(Change Function Declaration)
  - 변수 이름 바꾸기(Rename Variable)
  - 필드이름 바꾸기(Rename Field)

## Refactoring 1. 함수 선언 변경하기

- 좋은 이름을 찾아내는 방법?
- 함수에 주석을 작성한 다음, 주석을 함수 이름으로 만들어 본다.
- 함수의 매개변수는
  - 함수 내부의 문맥을 결정한다.
    - ex, 전화번호 포매팅 함수
  - 의존성을 결정한다.
    - ex, Payment 만기일 계산 함수에서 Payment 객체를 넘길 것이냐 or DueDate만 넘길 것이냐. 의존성을 결정
- intellij change signature 기능(cmd + f6)
  - 파라미터, 메서드이름, 리턴타입, 접근제한자 수정 가능

## Refactoring 2. 변수 이름 바꾸기

- 더 많이 사용되는 변수일수록 그 이름이 더 중요하다

  - 람다식의 변수 vs 함수의 매개변수

- 다이나믹 타입을 지원하는 언어에서는 타입을 이름에 넣기도 한다.

- 처음부터 핏한 네이밍을 하긴 어렵다.

- 시간이 지나면서 핏한 변수 이름으로 리팩토링 하는 습관을 들이자

- 

  