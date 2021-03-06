---
layout: post
title: "Javascript는 어떻게 작동하는가"
subtitle: "V8 엔진 들여다보기"
date: 2018-03-24
author: ttuyon
category: study
tags: javascript v8-engine
finished: false
---

원문: <i class="fa fa-medium"></i> [How JavaScript works: inside the V8 engine + 5 tips on how to write optimized code](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)

# Overview

자바스크립트 엔진은 자파스크립트 코드를 실행하는 프로그램 또는 인터프리터다. 자바스크립트 엔진은 기본 인터프리터로써 구현 될 수도 있고, 자바스크립트 코드를 어떤 형색의 바이트코드로 컴파일 하는 just-in-time 컴파일러로도 구현 될 수 있다.

다음은 자바스크립트 엔진을 구현한 유명한 프로젝트들의 목록이다.

- V8: 오픈소스, 구글에서 개발, C++로 작성됨
- 라이노(Rhino): 모질라 재단에서 관리되며, 오픈소스, 완전히 Java로 개발됨
- 스파이더몽키(SpiderMonkey): 첫번째 자바스크립트 엔진, 옛날엔 넷스케이프 네비게이터에서 제공했고, 지금은 파이어폭스에서 제공
- 자바스크립트 코어(JavascriptCore): 오픈소스, '니트로'라고 알려져있으며, Safari를 위해 Apple에서 개발
- 차크라(Chakra): IE/Edge

# 왜 V8 엔진이 만들어졌을까?

V8 엔진은 구글에 의해 만들어졌으며, C++로 작성되었다. 이 엔진은 Google Chrome의 내부에서 사용된다. 다른 엔진과는 다르게 V8은 Node.js의 런타임에도 사용된다.

V8은 처음엔 웹브라우저에서 자바스크립트 실행 성능을 증가시키기 위해 고안되었다. 속도를 얻기위해 V8은 인터프리터를 사용하는 대신에 자바스크립트 코드를 더욱 효율적인 기계어로 변환한다. V8은 많은 현대의 브라우저들처럼 **JIT (Just-In-Time) 컴파일러**를 구현함으로써, 실행시에 자바스크립트 코드를 기계어로 컴파일한다. 주요한 차이는 V8은 어떠한 바이트코드나 중간코드를 생성하지 않는다는 것이다.

# V8은 두가지 컴파일러를 갖는다.

(올초에 출시된) V8 버전 5.9 전까지는 두가지 컴파일러를 사용하였다: 

- full-codegen: 간단하고 상대적으로 느린 기계어를 만드는 간단하고 매우 빠른 컴파일러
- Crankshaft: 매우 최적화된 코드를 만드는 조금더 복잡한 최적화 컴파일러

또한 V8 엔진은 내부적으로 몇 개의 쓰래드를 가지고 있다:

- 메인 쓰래드: 코드를 가지고오고, 컴파일 한 다음 그것을 실행
- 컴파일을 위한 다른 쓰래드: 이 쓰래드가 있어서 코드를 최적화 하는동안 메인 쓰래드가 실행을 계속 할 수 있음
- 프로파일러 쓰래드: Crankshaft가 많은 시간을 쓰는 메서드를 최적화 할 수 있도록 알려줌
- 가비지 컬렉터 쓰래드

처음 자바스크립트 코드를 실행할 때, V8은 파싱된 자바스크립트를 어떠한 변화도 없는 기계어로 변환하는 **full-codegen**을 실행한다. 이로서 매우 빨리 기계어의 실행을 시작할 수 있다. V8은 인터프리터의 필요성을 제거하면서 중간 바이트코드 표현을 사용하지 않는다.

언젠가 당신의 코드가 실행 될 때, 프로파일러 쓰래드는 어떤 메서드가 최적화 되야하는지를 나타내는 충분한 데이터를 모은다.

다음, **Crankshaft**가 다른 쓰래드에서 최적화를 시작한다. Crankshaft는 자바스크립트 추상화 문법 트리를 **하이드로젠(Hydrogen)**이라고 불리는 고차원의 정적 단일 할당(static single-assignment, SSA) 표현으로 변경한다. 그리고 그 하이드로젠 그래프를 최적화 한다. 대부분의 최적화는 이 레벨에서 일어난다.

# 인라인화

첫번째 최적화는 가능한 많은 코드를 미리 인라인화 하는것이다. 인라인화는 호출 위치를 호출된 함수로 바꾸는 작업니다. 이 간단한 단계는 이후의 더 의미있는 최적화를 할 수 있게 해준다.

# 숨은 클래스

자바스크립트는 프로토타입 기반의 언어다: 복제 과정을 이용하여 생성된 클래스나 객체가 존재하지 않는다. 또한 자바스크립트는 객체가 초기화 된 후, 프로퍼티를 쉽게 추가하거나 제거할 수 있는 동적 프로그래밍 언어이다.

대부분의 자바스크립트 인터프리터는 메모리에 객체의 프로퍼티 값의 위치를 저장하기 위해 딕셔너리같은 구조(해시 함수에 기반하는)를 사용한다. 이러한 구조는 자바스크립트에서 프로퍼티 값을 가져오는것을 Java나 C#같은 비 동적 프로그래밍 언어보다 계산 상으로 비싸게 만든다. 자바에서 모든 객체프로퍼티들은 컴파일 전에 고정된 객체 레이아웃에 의해 결정되고, 런타임에 동적으로 추가하거나 제거할 수 없다. 결과적으로, 프로퍼티 값들은 사이에 고정 오프셋과 함께 메모리의 지속적인 버퍼로서 저장될 수 있다. 이 오프셋의 길이는 프로퍼티 타입에 기반하여 쉽게 결정 될 수 있다. 반면에 런타임동안 프로퍼티 타입을 바꿀 수 있는 자바스크립트는 불가능하다.

딕셔너리로 메모리의 객체 프로퍼티의 위치를 찾는것은 매우 비효율적이기 때문에, V8은 다른 메서드를 대신 사용한다: **숨은 클래스**. 숨은 클래스는 런타임에 실행된다는 것을 빼면 Java같은 언어에서 쓰이는 고정된 객체 레이아웃과 비슷한 동작을 한다. 이제 실제로 그들이 어떻게 동작하는지 살펴보자:

```javascript
function Point(x, y) {
  this.x = x;
  this.y = y;
}

var p1 = new Point(1, 2);
```

한번 `new Point(1, 2)`의 실행이 일어나면 V8은 "C0"이라고 불리는 숨은 클래스를 생성할것이다.

이때는 아직 어떠한 프로퍼티도 `Point`에 정의되지 않는다. 따라서 "C0"은 비어있다.

한번 `this.x = x`가 실행되면(`Point` 함수 안에서), V8은 "C0"에 기반하여 두번째 숨은 클래스인 "C1"을 생성한다. "C1"은 프로퍼티 x를 메모리 위치상 어디서 찾을 수 있는지에 대해 설명한다. 이 경우 "x"는 offset 0에 저장된다. 