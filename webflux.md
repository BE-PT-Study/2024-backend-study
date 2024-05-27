## 서론
Restemplate은 Spring 5.0 이후 더이상 사용되지 않고(deprecated) restemplate 대신 Webclient를 사용하도록 권고되고 있다.

* 참고 : https://docs.spring.io/spring-framework/docs/5.2.1.RELEASE/javadoc-api/index.html?org/springframework/web/client/RestTemplate.html 

Spring Webflux는 Reactive streams를 기반으로 데이터를 처리하므로 Reative streams, Reactor, Webflux순으로 기술하려고 한다.

## Reactive streams
reative streams는 non-blocking, back pressure를 갖는 비동기 스트림 애플리케이션을 구현할 수 있게 하는 패러다임이다.
Reactive streams 실행 흐름
1. Subsciber가 subscribe함수를 사용하여 Publisher에게 구독 요청
2. Publisher가 onSubscribe함수를 사용하여 Subscriber에게 Subscription 전달
3. Subscription의 request함수를 통해 데이터 요청
4. Puslisher는 Subscription을 통해 Subscriber의 onNext에 데이터 전달
5. 에러 발생시 onError에 시그널 전달 , 작업 완료 시 onComplete에 시그널 전달

reactive streams는 Processor, Publisher, Subscriber, Subsription으로 구성되고 back-pressure를 지원하며 Publisher와 Subscriber의 상호작용으로 이루어진다.
- back-pressure: publisher가 subscriber에게 receiver가 처리할 수 없을 정도의 속도로 데이터를 전송하여 systemic failure가 나는 것을 방지하기 위해 TCP 흐름 제어를 사용하여 배압을 바이트 단위로 조절한다.
- 블로킹: 제어권을 호출한 함수에 넘겨주어 제어권이 돌아올 때까지 기다린다
- 논블로킹: 함수를 호출해도 제어권을 그대로 가지고 있어 자신의 코드를 계속 실행한다.
- 동기: 함수를 호출 한 후 호출한 함수의 리턴값을 계속 확인한다.
- 비동기: 함수를 호출 한 후로 호출한 함수의 작업 완료 여부를 신경 쓰지 않는다.

Publisher
``` java
public interface Publisher<T> {
   public void subscribe(Subscriber<? super T> s);
}
```

Subscriber
``` java
package org.reactivestreams;
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}
```
onSubscribe: Publisher는 onSubscribe의 실행을 통해 Subscriber와 연동된 Subscription을 받는다. Subscription을 통해서 Subscriber는 Publisher와 직접적으로 통신할 필요가 없어진다. Subscription은 Subscriber로부터 전달받는 피드백을 통해서 Publisher로부터 아이템을 가져오고 그것을 Subscriber에게 전달한다. 

onNext: Publisher는 Subscription을 통해 Subscriber의 onNext에 데이터를 전달
onError: 데이터를 전달하며 에러가 발생하면 OnError 시그널을 전달
onComplete: 작업이 완료됐음을 전달



Subscription
``` java
public interface Subscription {
   public void request(long n);
   public void cancel();
}
```

Reactive streams 실행 흐름
1. Subsciber가 subscribe함수를 사용하여 Publisher에게 구독 요청
2. Publisher가 onSubscribe함수를 사용하여 Subscriber에게 Subscription 전달
3. Subscription의 request함수를 통해 데이터 요청
4. Puslisher는 Subscription을 통해 Subscriber의 onNext에 데이터 전달
5. 에러 발생시 onError에 시그널 전달 , 작업 완료 시 onComplete에 시그널 전달


## Reactor
Project Reactor는 JVM에서 비동기 및 반응형 애플리케이션 구축을 위해 위의 Reactive Streams을 활용하는 라이브러리이다.


Publisher
reator는 두 가지 publisher 타입이 존재하는데 
Mono는 0~1개의 데이터 스트림
Flux는   0~N건의 데이터 스트림이다.
*Mono 출처: https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html
*Flux 출처: https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html
 

## 대용량 리스트를 병렬로 처리: Flux.parallel()

parallel()는 downStream에 대한 데이터 처리를 병렬로 사용할 수 있게하는 메소드이다. runOn()은 어떤 스펙을 가지고 진행할 것인지 선언한다. Scheduler의 실행 전략은 다음과 같다.

elastic: thread의 숫자를 알아서 조정한다. 블로킹 IO를 리액터로 처리할 때 적합하다.
parallel: 고정 크기를 만드는 scheduler로 CPU를 많이 사용하지만 생명주기가 짧은 태스크를 위한 scheduler이다. 
single: 한 Thread를 공유한다.
immediate:  현재 Thread에서 실행한다. newParallel을 통해 원하는 쓰레드 풀을 만들 수 있다.
``` java
public void subscribeFlux(List<String> msgs){
	Flux.fromIterable(msgs)
		.parallel()
		.runOn(Schedulers.parallel())
		.sequential()
		.flatMap(msg -> sendMessage(msg))
		.subscribe(response -> {
			log.info("response: " + rsponse);
			//api 전송으로 받은 response값 처리
		})
		.doOnNext(msg-> log.info("msg is sended, msg: " + msg))
		.doOnComplete(() -> {
			log.info("messages have been sent completely.");
			//db에 로그 적재 및 상태 정보 변경 등등의 작업
		});
 }
 ``` 
