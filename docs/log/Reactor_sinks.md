---
title: Reactor Sink 트러블 슈팅
nav_order: 1
---
> **_NOTE:_** Reactor 클래스인 `Sink` 를 사용하다 경험한 이슈를 기록합니다.

## 이슈
최근에 기존 배치 작업이 정상적으로 처리되고 있지 않은 것 같다는 이슈가 제보됐는데,<br> 
확인해 보니 실제로 몇몇 데이터가 누락된(처리되지 않은) 상태였습니다.<br>

데이터 서버에서 REST API 를 통해 조회했을 때는 값이 조회됐는데,<br>
같은 데이터를 참조하는 배치 작업에서는 해당 데이터가 어째선지 처리가 되지 않았습니다.<br>

더 이상했던 점은 문제가 계속해서 발생하는 것이 아닌, 때때로 발생했다는 점이었습니다.<br>
어떤 문제가 있었는지, 배치가 처리되는 구조와 함께 이야기해 보겠습니다.
<br><br><br>

<img src="https://github.com/user-attachments/assets/60b52212-8a5d-477d-ba5f-4ad74419bca7">

배치 작업은 `spring-batch` 를 이용해 구현돼있고, 데이터 처리 방식은 위 구조와 같습니다.<br>
데이터가 있는 서버와 배치가 실행되는 app 은 gRPC bi-stream 통신을 이용해 데이터를 주고받으며,<br>
client observer 의 `onNext` 에서 `Sink` 에 데이터를 전송해 줍니다.<br>
이후 데이터 변환, kafka 메세지 발행 및 DB I/O 를 수행하고 종료됩니다.<br>

`Sink` 는 간단하게 구현돼있었는데, 어떤 설정도 따로 하지 않았습니다.
```java
//  downstream 생성
Sinks.Many<?> sinks = Sinks.many()
    .multicast()
    .onBackpressureBuffer()
```

로그를 넣어 테스트해 본 결과, downstream(sink) 에 emit 한 데이터가 이후 operator 에서 처리되지 않음을 발견합니다.<br>
`Sink.tryEmitNext` 결과인 `EmitResult` 를 받아 처리했다면 이슈를 빨리 알 수 있었겠지만, 결과에 대한 처리를 누락했기 때문에 늦게 발견하게 됐습니다.<br>

그렇다면 어째서 emit 에 실패한 것일까요?<br>
이유를 알기 위해선 **hot, cold sequence** 와 **backpressure buffer** 에 대한 이해가 필요합니다. 

---

## hot, cold sequence
먼저 hot, cold 의 개념에 대해 간단하게 이야기해 보겠습니다.

### cold sequence
cold sequence 는 구독자(subcriber)가 구독하기 전에는 데이터를 발행하지 않습니다.<br>
따라서 구독자가 발행자(publisher)의 데이터 전송을 조절할 수 있는 backpressure 형태를 갖는다고 할 수 있습니다.<br>
또한 다른 구독자가 구독하더라도 데이터를 처음부터 수신할 수 있습니다. (독립적인 sequence 를 갖습니다.)

대표적으로는 `Flux.fromIterable`, `Flux.create`, `Mono.create` 등을 사용해 만든 publisher 들이 있습니다.
```kotlin
fun main() {
    val list = listOf(1, 2, 3, 4, 5)
    val flux = Flux.fromIterable(list)
        .delayElements(Duration.ofMillis(1000L))

    flux.subscribe { i -> println("Sub-1 received: $i") }
    //  flux 에서 1, 2, 3 을 발행할 때까지 대기
    Thread.sleep(3000L)
    //  이후 구독
    flux.subscribe { i -> println("Sub-2 received: $i") }
}
```
```shell
Sub-1 received: 1
Sub-1 received: 2
Sub-1 received: 3
Sub-2 received: 1   //  sequence 의 데이터를 처음부터 수신할 수 있다.
Sub-1 received: 4
Sub-2 received: 2
Sub-1 received: 5
Sub-2 received: 3
Sub-2 received: 4
Sub-2 received: 5
```

### hot sequence
hot sequence 는 구독과 관계없이, sequence 로 데이터를 발행하는 push 구조입니다.<br>
따라서 구독자는 구독한 뒤에 발행된 데이터부터 수신할 수 있으며, 이전에 발행된 데이터는 수신할 수 없을 가능성이 있습니다.

대표적으로는 `Sink`, `ConnectableFlux`, `Flux.share`, `Mono.just` 등을 사용해 만든 publisher 들이 있습니다.<br>
```kotlin
fun main() {
    val list = listOf(1, 2, 3, 4, 5)
    val flux = Flux.fromIterable(list)
        .delayElements(Duration.ofMillis(1000L))
        .publish()

    flux.subscribe { i -> println("Sub-1 received: $i") }
    //  flux 에서 1, 2, 3 을 발행할 때까지 대기
    Thread.sleep(3000L)
    //  이후 구독
    flux.subscribe { i -> println("Sub-2 received: $i") }
}
```
```shell
Sub-1 received: 1
Sub-1 received: 2
Sub-1 received: 3
Sub-1 received: 4
Sub-2 received: 4   //  구독 이후 발행된 데이터부터 수신할 수 있습니다.
Sub-1 received: 5
Sub-2 received: 5
```

---

## sink 와 backpressure
앞서 이야기한 대로, `Sink` 는 hot sequence 의 일종입니다.<br>
따라서 구독 여부와 관계없이 데이터를 발행하게 됩니다. (push)

그렇다면 구독 이전에 발행된 데이터는 유실되는 걸까요?<br>
`Sink` 에서는 위 같은 상황을 위해 backpressure buffer 를 관리합니다.<br>
```
.onBackpressureBuffer()
//  아무것도 설정하지 않은 경우, buffer capacity 는 아래 설정을 따릅니다.
//  Queues.SMALL_BUFFER_SIZE=Math.max(16, Integer.parseInt(System.getProperty("reactor.bufferSize.small", "256")))
```
위 함수를 이용해 backpressure buffer 와 관련된 설정을 할 수 있습니다.<br>
첫 번째 인자로는 buffer 의 크기를 설정할 수 있고,<br>
두 번째 인자는 `autoCancel` flag 이며, 구독자가 여럿 존재하는 상황에서 `dispose` 가 발생했을 때 처리하는 방식에 대해 조절할 수 있는 값입니다. 

`Flux.create` 을 이용해 sequence 를 생성해 보셨다면 아시겠지만,<br>sequence 생성 시점에 overflow strategy 를 지정해 줄 수 있습니다.<br>
`Sink` 는 기본적으로 `OverflowStrategy.BUFFER` 가 설정됩니다.

따라서 `Sink` 에 발행한 데이터는 backpressure buffer 에 먼저 저장되고,<br>
buffer count 가 감소하는 시점은 구독자가 데이터를 요청(subscription.request)하는 순간입니다.<br>

따라서 backpressure buffer 가 bounded capacity 를 갖는다면<br>
publisher 가 데이터를 발행하는 속도가 subscriber 에서 데이터를 요청하는 속도보다 빠를 때 유실(drop) 되는 데이터가 발생할 수 있습니다.<br>

저의 경우에는 위 속도 차이가 더 심했었는데,<br>
DB I/O 를 줄이기 위해 subscriber 에서 데이터를 batch 형태로 처리하고 있었기 때문입니다(`window` 사용).<br>

---

## 해결안
그렇다면 위 상황을 어떻게 해결할 수 있을까요?<br>
제가 생각한 방법은 2가지 정도입니다.<br>

### 1. backpressure buffer 와 subscriber request 사이즈 조절
drop 이 되는 기준이 buffer 에 존재하는 element 의 수라면,<br>
element 를 저장하는 배열의 크기를 늘리고 구독자가 좀 더 빠르게 element 를 release 해주면 됩니다.<br>

저는 이 방법을 채택했는데, DB I/O 를 늘리기가 더 부담스러웠기 때문에 batch 처리를 포기할 수는 없었기 때문입니다.<br>
따라서 구독자 쪽에서 요청하는 batch 크기를 줄이고, buffer 의 크기를 늘려 처리할 수 있었습니다.

다만 buffer 의 크기가 늘어난 만큼 메모리를 점유하기 때문에, 이 점에 유의해야 합니다.

### 2. sink 이후 처리하는 operation 을 다른 sequence 로 분리한다.
operation 을 여럿 처리하는 경우 subscriber 와 publisher 의 처리 속도가 벌어질 텐데요,<br>
이 때 발생할 수 있는 데이터 유실을 막기 위해 hot sequence 에 어떤 operation 도 없이 block 만 하도록 합니다.<br>

reactive context 에서는 사용할 수 없는 방법이지만, blocking 이 허용되는 환경에서는 위와 같이 처리해 데이터 유실을 막아볼 수는 있을 것 같습니다.<br>

### 3. buffer 가 available 할 때까지 대기한다(spinlock 처럼)
대기하는 시간이 정말 짧을 것이라 보장할 수만 있다면 이 방법을 쓸 수도 있겠지만,<br>
reactive context 내에서 처리하는 방식과는 전혀 어울리지 않는 방법인 것 같습니다.
