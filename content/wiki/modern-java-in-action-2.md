---
title   : '모던자바인액션: 스트림' 
slug  : '/modern-java-2'
layout  : wiki 
excerpt : 
date    : 2020-07-01 11:29:27 +0900
updated : 2020-07-01 18:07:37
tags    : 
---

# 4장 스트림 소개 
  
## 4.1 스트림이란 무엇인가? 

Stream은 자바 8 API 에 새로 추가된 기능이다. 스트림을 사용하면 선언형으로 컬렉션 데이터를 처리할 수 있다. 선언형은 SQL 과 같이, 데이터를 직접 처리하는 코드를 작성하지 않고 질의로 명백히 표현하는 것을 말한다. for 문을 돌면서 매번 이 값이 100을 넘는 지 검사하는 것은 선언형이 아니지만, `100을 넘는 값만 찾아라` 라고 값을 넘기면 선언형이다. 

스트림을 이용하면 멀티스레드 코드를 구현하지 않아도 데이터를 투명하게 병렬로 처리할 수 있다. 예제를 한번 보자. Java 7 , Java 8 로 저칼로리의 음식만을 반환하는 예제를 살펴본다. 

```java
public static List<String> getLowCaloricDishesNamesInJava7(List<Dish> dishes) {
    List<Dish> lowCaloricDishes = new ArrayList<>();
    for (Dish d : dishes) {
      if (d.getCalories() < 400) {
        lowCaloricDishes.add(d);
      }
    }
    List<String> lowCaloricDishesName = new ArrayList<>();
    Collections.sort(lowCaloricDishes, new Comparator<Dish>() {
      @Override
      public int compare(Dish d1, Dish d2) {
        return Integer.compare(d1.getCalories(), d2.getCalories());
      }
    });
    for (Dish d : lowCaloricDishes) {
      lowCaloricDishesName.add(d.getName());
    }
    return lowCaloricDishesName;
  }
```

```java
public static List<String> getLowCaloricDishesNamesInJava8(List<Dish> dishes) {
    return dishes.stream()
        .filter(d -> d.getCalories() < 400)
        .sorted(comparing(Dish::getCalories))
        .map(Dish::getName)
        .collect(toList());
  }
```

일단 길이의 차이를 보시라! java 7에서는 직접 내용을 구현했지만, java8에서는 조건을 명시하고 적절한 stream API 를 사용하는 것만으로 내용을 처리했다. 더불어 java 8 코드를 멀티코어환경에서 병렬로 실행하고 싶다면, **stream()을 parallelStream()** 으로 바꿔서 처리할 수 있다. 

이런 스트림의 형태는 명시적으로 다음과 같은 이득을 준다. 

- 선언형으로 코드를 구현할 수 있다. 즉, 루프와 조건문 등 제어 블록을 사용해서 어떻게 구현할지 지정할 필요가 없이, '동작의 수행을 어떻게 할지'만 정해주면 된다.  이렇게 되면 변하는 요구사항에 쉽게 대응할 수 있다. 기존의 코드를 복사 붙여넣기 하지않고, 람다를 이용해서 저칼로리 대신 고칼로리의 요리를 필터링하도록 할 수 있다. 
  
- 위에서처럼 여러 빌딩블록 연산을 연결해서, **복잡한 데이터 처리 파이프라인을** 만들 수 있다. 여러 연산을 파이프라인으로 연결해도 여전히 가독성과 명확성이 유지된다. 이 예제에서 filter의 결과는 sorted로, 이 결과는 map으로 ... 계속 연결된다. 

즉 자바8의 스트림API 의 특징을 다음처럼 요약할 수 있다. 
- 선언형 : 더 간결하고 가독성이 좋다.
- 조립할 수 있음: 유연성이 좋아진다. (파이프)
- 병렬화: 성능이 좋아진다.

> filter (혹은 sorted, map, collect) 와 같은 연산은 **하이레벨 빌딩 블록** 이므로, 특정 스레딩 모델에 제한되지 않고 자유롭게 어떤 상황이든 사용할 수 있다. 내부적으로 단일 스레드 모델에 사용할 수 있지만, 멀티코어 아키텍처를 최대한 활용하게 되어있다. 결과적으로 우리는 데이터 처리과정을 병렬화 하면서도 ... 스레드와 락을 걱정할 필요가 없다! 

> 컬렉션을 제어하는데 도움되는 다른 라이브러리들 : 구아바, 아파치, 람다제이 구아바는 구글에서 만든 라이브러리로, 멀티맵, 멀티셋등 추가적인 컨테잌너 클래스를 제공한다. 아파치 라이브러리도 비슷한 기능을 제공한다. 람다제이는 선언형으로 컬렉션을 제어하는 다양한 유틸을 제공한다. 

## 4.2 스트림 시작하기

### 컬렉션 스트림
자바8 컬렉션에는 스트림을 반환하는 stream 메서드가 추가됐다. 즉 List, Set등에서 Stream을 얻을 수 있다는 뜻이다. 추가로, 내가 임의로 정한 숫자 범위나 I/O 자원에서도 Stream을 얻을 수 있다. 

스트림은 
- 데이터처리연산을 지원하도록:스트림은 함수형에서 일반적으로 지원하는 연산, 그리고 DB와 비슷한 연산을 지원한다. filter, map, reduce, find, match, sort 등으로 데이터 조작이 가능하다.
- 소스에서 추출된: 스트림은 컬렉션, 배열, I/O 자원 등의 데이터 제공 소스로부터 데이터를 소비한다. 
- 연속된 요소: 컬렉션과 마찬가지로 특정 요소로 이루어진 연속된 값집합의 인터페이스를 제공한다. 컬렉션은 자료구조이므로 주제가 데이터이고, 스트림은 표현 계산으로 데이터 조작을 하는데 주를 두므로 스트림의 주제는 계산이라고 할 수 있다. 

를 말한다.


스트림은 아래처럼 두가지 특징이 있다. 
- 파이프라이닝 : 스트림은 연산끼리 연결해서 커다란 파이프라인을 구성할 수 있도록, **스트림 자신을 반환한다**. 그 덕분에 laziness, 쇼트서킷과 같은 최적화도 얻을 수 있다. 

- 내부 반복 : 스트림은 내부 반복을 지원한다. (콜렉션은 반복자로 명시적으로 반복 / iterator().next 등)


## 4.3 스트림 vs 컬렉션 

아까 위에서 `스트림과 컬렉션은 특정 요소로 이루어진 연속된 값 집합`이다. 차이는 주제가 데이터이냐 계산이다라고 말했다. 이를 좀더 알아보자! 🤔
**연속된** 이라는 말은 랜덤하게 아무 값에나 접근하는 것이 아니라 순차적으로 값에 접근한다는 것이다. 

스트림과 컬렉션의 가장 큰 차이는 **데이터를 언제 계산하느냐**이다. 컬렉션은 현재 자료구조가 포함하는 모든 값을 메모리에 저장하는 자료구조다. 즉, 컬렉션의 모든 요소는 컬렉션에 추가하기 전에 계산되어야한다. 

반면 스트림은 이론적으로는, 요청할 때만 요소를 계산하는 고정된 자료구조다. (스트림에 요소를 추가하거나, 제거할 수 없다.) 이런 특성은 프로그래밍에 큰 도움을 준다. 사용자가 요청하는 값만 스트림에서 추출한다는 것이 핵심이다. 결과적으로, 스트림은 생산자와 소비자 관계를 형성한다. 소비자 중심의 스트림은 요청을 받을 때만 만든다(즉석 제조)

컬렉션은 생산자 중심으로, 소비자에게 생성되기전에 전에 모든 값을 저장해두는 형태를 갖는다. 무한한 소수를 포함하는 콜렉션을 만든다고하자. 이렇게 되면 계속 루프를 돌며 추가하는 과정때문에, 소비자는 영원히 결과를 알 수 없게 된다. 

### 4.3.1 한번만 탐색할 수 있다. 

반복자 (iterator)와 마찬가지고 스트림도 한번만 탐색할 수 있따. 다시 탐색하려면 초기의 데이터 소스에서 새로운 스트림을 만들어야한다. (이때 소스가 재사용이 가능한, 컬렉션 등 이어야한다. I/O 채널이라면 소스는 이미 지나갔으므로 스트림을 만들 수 없다.)

### 4.3.2 외부반복, 내부반복

컬렉션 인터페이스를 사용하려면 for-each 등을 사용하여 직접 사용자가 요소를 반복해야한다. 이를 외부 반복이라고한다. 반면 스트림 라이브러리는 반복을 알아서 처리하고, 결과 값을 어딘가 저장해주는 내부 반복을 사용한다. 

스트림 라이브러리의 내부 반복은 데이터표현과 하드웨어에 따라서 병렬성 구현을 자동으로 선택해준다는 이점도 있다. 반면 외부반복에서는 병렬성을 스스로 관리해야한다. (synchronized로 시작하는 병렬성 구현) 

## 4.4 스트림 연산 

스트림에서는 파이프라이닝이 가능하다고 했다. 그런데, 이렇게 파이프라이닝이 가능하도록 Stream을 반환하는 연산을 중간연산, 그리고 스트림을 닫는 연산을 최종 연산이라고 한다. 

### 4.4.1 중간 연산
filter와 sorted와 같은 연산은 다른 스트림을 반환한다. 중요한 특징은 단말 연산을 스트림 파이프라인에 실행하기 전까지는 아무 연산도 수행하지 않는 다는 것, 즉 lazy 하다는 것이다. 

```java
List<String> names = menu.stream()
        .filter(dish -> {
          System.out.println("filtering " + dish.getName());
          return dish.getCalories() > 300;
        })
        .map(dish -> {
          System.out.println("mapping " + dish.getName());
          return dish.getName();
        })
        .limit(3)
        .collect(toList());
    System.out.println(names);
```

이 프로그램의 실행결과는 다음과 같다. 

```
filtering::pork
mapping::pork
filtering::beef
mapping::beef
filtering::chicken
mapping::chicken
[pork, beef, chicken]
```

menus에는 300칼로리가 넘는 음식이 여러개가 있다. 그러나 limit(3)의 연산까지 고려되어, 모든 요리를 다 고려하는 것이 아니라 처음 3개만 선택되었다. 이는 쇼트서킷이라고 불리는 기법덕분이다. 또한 filer와 map은 서로 다른 연산이지만 한 과정으로 병합되었다. 이를 루프 퓨전이라고 한다. 

### 4.4.2 최종 연산

최종 연산은 스트림에서 결과를 도출한다. 최종 연산에 의해 보통 List, Integer, void 등 Stream이 아닌 결과를 반환한다. collect, count 등이 있다.

### 4.4.3 스트림 이용하기 

즉 스트림은 다음과 같이 이용할 수 있다. 

- 질의를 수행할 데이터 소스  (에서 스트림 만들기)
- 스트림 파이프라인을 구성할 중간 연산 연결
- 스트림 파이프라인을 실행하고 결과를 만들 최종 연산 


# 5장 스트림 활용 

## 5.1 필터링

필터링의 두가지 방법을 배워보자.

- Predicate 필터링
- 고유 요소 필터링 

### 5.1.1. Predicate 필터링 

스트림 인터페이스는 filter 메서드를 지원한다. filter에서는 Predicate(boolean을 반환하는 함수, T -> boolean!) 을 인수로 받아서 이와 일치하는 모든 요소를 반환하는 스트림을 반환한다. 

### 5.2.2 고유 요소 필터링, distinct 
스트림은 고유 요소로 이루어진 스트림을 반환하는 distinct 메서드를 지원한다! 고유의 여부는 객체의 hashCode와 equals 로 결정된다. 중복을 필터링할 때 유용하다.