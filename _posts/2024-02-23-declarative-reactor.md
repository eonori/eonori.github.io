---
title: Reactor에 유닛 테스트가 필요한 이유
date: 2024-02-24 11:40:00 +0900
categories: [Async, Reactor]
tags: [Reactor]
---
# Reactor에 테스트가 절대적으로 필요한 이유
Reactive Streams 표준을 따르는 RXJava나 Reactor 등의 프레임워크는 백엔드 비동기 프로그래밍 구현을 위해 자주 사용됩니다.
이러한 프레임워크들의 큰 특징 중 하나는 스트림의 조합을 연산자(Operator) 단위로 제공한다는 겁니다. 
다시 말해, 이 프레임워크들은 사용자에게 "우리가 이미 유용한 연산자들을 많이 만들어 놨으니, 너희들은 단지 선언만 해서 쓰면 돼!" 라고 말합니다.
따라서 이 프레임워크들을 사용하면 자연스럽게 선언적(declarative) 프로그래밍을 하게 됩니다.

## 선언적(declartative) 프로그래밍의 단점
가독성, 직관성 면에서 명령형(imperative) 코드에 비해 월등히 나은 이 선언적 코드는 남의 코드 보기 싫어하는 개발자들에게는 분명히 큰 장점이 됩니다.
무엇을 할건지에 대한 설명만 겉으로 드러나있고, 실제 복잡한 로직은 감추어져 있으니까요. 
하지만 감추어져 있기 때문에, 선언적 프로그래밍은 같이 개발하는 동료들 간의 사전 약속이 무엇보다 중요합니다.
실제 어떻게 동작하는지에 대한 내용이 캡슐화 되어있으니, 이것을 어떻게 사용해야 되는지에 대한 공유가 이루어지지 않으면, 적절하지 않은 메소드의 호출 때 문제가 생길 겁니다.
더 큰 문제는 선언적 코드만 보았을때 겉보기엔 별 문제가 없는 것 처럼 보이니, 문제해결이 더 늦어지는 답답한 상황이 충분히 발생할 수 있죠.

자, 그렇다면 이런 선언적 코드에서 캡슐화의 레벨이 프레임워크 뒤에 숨어 있는 경우라면 어떨까요?
Reactor는 정말 많은 연산자들을 제공하지만, 이 연산자들이 실제로 어떻게 구현되어 있는지를 모두 보면서 파악하는건 불가능합니다. 이 프레임워크 개발자들도 그것을 바라진 않을거구요.
그렇다면 그냥 연산자들을 사용하면 되느냐? 그것도 아닙니다. 아래 예시를 한번 보죠.



```java
class ReactorTest {

    @Test
    public void splitAndMerge() {
        Sinks.Many<Integer> many = Sinks.many().multicast().onBackpressureBuffer();

        Flux<Integer> root = many.asFlux();

        Flux<Integer> even = root.filter(i -> i % 2 == 0);
        Flux<Integer> odd = root.filter(i -> i % 2 != 0);

        Flux<Integer> merged = Flux.merge(odd, even);

        many.tryEmitNext(1);
        many.tryEmitNext(2);
        many.tryEmitNext(3);
        many.tryEmitNext(4);
        many.tryEmitComplete();

        merged.subscribe(System.out::println,
                Throwable::printStackTrace,
                () -> System.out.println("Complete"));
    }
}
```
Reactor 에서는 서로 다른 처리 흐름의 스트림들을 하나로 모아주는 `merge`라는 연산자를 제공합니다. 코드 예시는 1에서 4까지의 정수값을 `many` 에 넣습니다.
`many` 에서 스트림은 2개로 나뉘며, 나뉜 스트림들은 짝수(`even` 스트림)와 홀수(`odd` 스트림)만 받아들이고, 다시 이 2개의 스트림들은 하나의 스트림으로 합쳐지게 됩니다. 
마지막 `subscriber`가 받게되는 값은 어떻게 될까요? 결국 두 스트림들은 하나로 합쳐지기에, 값은 1,2,3,4 모두 출력될 것이라 생각할 수 있습니다. 하지만 결과는...
```text
1
3
Complete
```
`Subscriber`는 홀수만 받았습니다. 2, 4는 어디로 간걸까요? 
분명 코드만 봤을 때는 1, 2, 3, 4를 모두 `Subscriber`가 받아야 했다고 생각할 법도 한데요.
다른 예시를 보겠습니다.

```java
class ReactorTest {

  @Test
  public void splitAndMerge2() {
    Flux<Integer> root = Flux.just(1, 2, 3, 4);

    Flux<Integer> even = root.filter(i -> i % 2 == 0);
    Flux<Integer> odd = root.filter(i -> i % 2 != 0);

    Flux<Integer> merged = Flux.merge(odd, even);

    merged.subscribe(System.out::println,
      Throwable::printStackTrace,
      () -> System.out.println("Complete"));
  }
}
```
데이터를 `Sinks.Many`를 사용해 프로그래밍 방식(programmatically)으로 넣는게 아닌 `just` 연산자를 사용해 넣도록 변경했습니다. 이러면 결과가 좀 다를까요?

```text
1
3
2
4
Complete
```
네, 다릅니다. 첫번째 예시와 달리 모든 정수값을 `Subscriber`가 잘 받았네요. 대체 스트림 안에서 무슨 일이 일어났기에 이런 차이를 보이는 걸까요?
그리고 하나 더, 모든 값을 받기는 했지만 1, 2, 3, 4 순이 아닌 1, 3, 2, 4 순으로 받게 된 이유는 뭘까요? 
이것을 알려면, Reactive Stream 기반 프레임워크들이 내부적으로 어떻게 동작하는 지를 알아야 합니다.

## 조립, 구독, 런타임 ...
우리가 Reactor의 연산자를 이용해 스트림을 만들면, 겉으로 보이는 코드는 꽤나 깔끔하고 단순해 보입니다. 하지만 이 단순하고 깔끔해 보이는 겉모습과 달리,
내부적으로는 스트림으로 들어오는 데이터들을 처리하기 위해 무려 3단계의 작업이 필요합니다. 
스트림의 수명 주기라고 부르는 이 단계들은 조립(assembling), 구독(subscribe), 런타임(runtime) 3단계로 구성됩니다.
실제로 데이터가 각 연산자를 거치는 동작은 런타임 단계에서만 일어납니다. 그럼 그 전의 조립과 구독은 뭘까요?

### 조립
조립은 연산자들을 모두 연결시키는 것을 말합니다. 두번째 예시에서 선언한 연산자들을 차례로 보면
`just`, `filter`, `merge`, `subscribe` 순으로 되어있습니다. 연산자들은 모두 `Publisher`를 상속한 클래스들입니다.
데이터 방출을 담당하는 역할의 이 `Publisher` 들을 연결시키는 것이 조립 단계에서 할 일 입니다.
연결은 어떻게 이루어질까요? `just`를 호출하면, `FluxJust` 클래스의 인스턴스를 리턴합니다.
그리고 난 후에 호출하는 `filter`는 `FluxFilter` 인스턴스를 리턴하게 되죠. 이 `FluxFilter`는 전에 만든 `FluxJust`를 멤버 변수로 가지게 됩니다.
실제 프레임워크 코드 예시를 볼까요?
```java
public abstract class FluxOperator<I, O> extends Flux<O> implements Scannable {

    protected final Flux<? extends I> source;
    
    protected FluxOperator(Flux<? extends I> source) {
        this.source = Objects.requireNonNull(source);
    }
  // ...
}
```
사실 `Publisher` 와 연산자 클래스 사이에는 더 많은 중간 클래스들이 상속 관계로 이어져 있습니다. 물론 이 중간 클래스들이 어떤 역할인지 일일이 알 필요는 전혀 없고,
다만 `FluxFilter`의 부모 클래스를 쫒아가다 보면 `source` 라는 멤버 변수가 있고, 여기에 이전 연산자를 넣는다는 사실을 확인할 수 있습니다.
그렇다면 다음 연산자들도 당연히 자신의 `source` 필드에 이전 연산자를 계속해서 넣게 되겠죠. 이것이 조립 단계입니다.


#### 그래서, 조립은 왜 하는건데?
어떻게 조립이 이루어지는지 대강 이해했습니다. 그런데, 조립을 왜 하는걸까요? 조립의 역할은 사실 여러가지입니다.
조립의 과정에서 같은 연산자 여러개가 연결되어 있는 경우, 성능 향상을 위해 하나의 연산자로 바뀌기도 하며, 연산자 중간마다 훅(hook)을 배치해,
런타임 단계 때 연산자 마다 중복 없이 공통 로직을 적용하고 싶을 때 도움이 될 수 있습니다. 데이터를 본격적으로 넣기 전 이런 사전 단계가 있으면
더욱 유연한 코드 개발이 가능합니다. 특히 훅은 컨텍스트(Context)와 관련해서 나중에 설명할 기회가 더 있을 것 같네요.

### 구독
스트림에 데이터가 흐르게 하려면, 마지막 연산자 인스턴스의 `subscribe` 메소드를 호출해야 합니다.
구독 순서는 마지막 연산자가 호출한 `subscriber` 에서 출발하여 맨 처음 연산자의 `subscriber` 까지 체인을 만들게 되는게 일반적입니다.
조립 순서와는 반대가 되죠. 구독 단계 역시 런타임 단계 전 최적화를 미리 수행할 수 있고, 특히 `subscribeOn` 과 같은 메소드를 이용해 구독 단계에서 스레드를 변경할 수 있습니다.
   
#### `merge` 안에서 일어나는 일
구독 체인은 특이할게 없으면, 맨 위 연산자가 만드는 `subscriber` 까지 연결될 겁니다. 그렇다면 특이한 경우는 어떤게 있을까요?
바로 위 예시에서 보여준 `merge` 입니다. 스트림이 2개에서 하나로 합쳐지는 부분이니, 구독 단계에서는 반대로 하나의 구독 체인이 여기서 2개로 나뉘게 됩니다.
여기서 구독이 어떻게 일어나는지를 살펴보면 예시의 결과가 왜 그렇게 나왔는지를 알 수 있습니다. 어디 한번 살펴봅시다.
