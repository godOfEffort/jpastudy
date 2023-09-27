## 리액티브 프로그래밍

---

### 애플리케이션 수준의 리액티브

애플리케이션 수준의 리액티브 프로그래밍의 주요 기능은 **비동기로 작업을 수행**할 수 있다는 것이다.
RxJava 같은 리액티브 프레임워크는 별도로 지정된 스레드 풀에서 블록 동작을 실행시켜 이 문제를 해결한다.

### 시스템 수준의 리액티브

리액티브 시스템은 여러 애플리케이션이 한 개의 일관적인, 회복할 수 있는 플랫폼을 구성할 수 있게 해주고, 
애플리케이션 중 하나가 실패해도 전체 시스템은 계속 운영될 수 있도록 도와주는 **소프트웨어 아키텍쳐**이다.
리액티브 시스템의 주요 속성으로는 **메시지 주도**가 있다.
리액티브 아키텍처에서는 컴포넌트에서 발생한 장애를 고립시킴으로 문제가 주변의 다른 컴포넌트로 전파되면서
전체 시스템 장애로 이어지는 것을 막음으로 회복성을 제공한다.
위치 투명성은 리액티브 시스템의 모든 컴포넌트가 수신자의 위치에 상관없이 다른 모든 서비스와 통신할 수 있음을 의미한다.

### 리액티브 스트림과 플로 API

리액티브 프로그래밍은 **리액티브 스트림**을 사용하는 프로그래밍이다.
**리액티브 스트림**은 잠재적으로 무한의 비동기 데이터를 순서대로 블록하지 않은 역압력을 전제해 처리하는 표준 기술이다.

역압력은 발행-구독 프로토콜에서 이벤트 스트림의 구독자가 발행자가 이벤트를 제공하는 속도보다 느린 속도로

이벤트를 소비하면서 문제가 발생하지 않도록 보장하는 장치다.

부하가 발생한 컴포넌트는 업스트림 발행자에게 이를 알릴 수 있어야 한다.

스트림 처리의 비동기적인 특성상 역압력 기능의 내장은 필수라는 사실을 알 수 있다. 

실제 비동기 작업이 실행되는 동안 시스템에는 암묵적으로 블록 API로 인해 역압력이 제공되는 것이다.

따라서 이런 상황을 방지할 수 있도록 역압력이나 제어 흐름 기법이 필요하다.

이들 기법은 데이터 수신자가 스레드를 블록하지 않고도 데이터 수신자가 처리할 수 없을만큼

많은 데이터를 받는 일을 방지하는 프로토콜을 제공한다.

### Flow 클래스 소개

자바 9에서는 리액티브 프로그래밍을 제공하는 `Flow`를 추가했다.

Flow 클래스는 중첩된 인터페이스 네 개를 포함한다.

- `Publisher`: 발행자
- `Subscriber`: 소비자
- `Subscription`: `Publisher`와 `Subscriber` 사이에 제어 흐름, 역압력을 관리
- `Processor`: `Publisher`를 구독한 다음 수신한 데이터를 가공해 다시 제공한다.

```java
public interface Publisher<T> {
        public void subscribe(Subscriber<? super T> subscriber);
    }
```
```java
    public interface Subscriber<T> {
        public void onSubscribe(Subscription subscription);
        public void onNext(T item);
        public void onError(Throwable throwable);
        public void onComplete();
    }
```
```java
public interface Subscription {
        public void request(long n);
        public void cancel();
    }
```

```java
   public interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
    }
```
`Publisher`가 이벤트를 발행할 때 호출할 콜백 메서드 네 개를 `Subscriber`에서 정의한다.

이벤트는 다음 프로토콜에서 정의한 순서로 지정된 메서드 호출을 통해 발행되어야한다.

```java
onSubscribe onNext* (onError | onComplete)?
```

`onSubscribe`가 항상 처음 호출되고 이어서 `onNext`가 여러번 호출될 수 있다.

이벤트 스트림은 영원히 지속되거나 `onComplete`콜백을 통해 더 이상 데이터가 없고 종료됨을 알린다.

`Publisher`에서 장애가 발생했을 때는 `onError`를 호출할 수 있다.


`Subscriber`가 `Publisher`에 자신을 등록할 때 `Publisher`는 처음으로 `onSubscribe`를 호출해 `Subscription`객체를 전달한다.


`request`는 주어진 개수의 이벤트를 처리할 준비가 되었음을 알리고

`cancel`을 호출해 구독을 취소할 수 있다.

자바 9의 플로 명세서에서는 인터페이스들이 어떻게 협력해야 하는지 설명하는 규칙 집합을 정의한다.

- `Publisher`는 반드시 `Subscription`의 `request` 메서드에 정의된 개수 이하의 요소만 `Subscriber`에게 전달해야 한다.
  - 동작이 성공적으로 끝났으면 `onComplete`를 호출한다.
  - 문제가 발생하면 `onError`를 호출해 `Subscription`을 종료할 수 있다.
- `Subscriber`는 요소를 받아 처리할 수 있음을 `Publisher`에게 알려야한다.
  - `onComplete`, `onError` 신호를 처리하는 상황에서 `Subscriber`는 `Subscription`이나 `Publisher`의 어떤 메서드도 호출할 수 없다.
  - `request` 호출 없이도 언제든 종료 시그널을 받을 준비가 되어있어야 한다.
  - `cancel`이 호출되더라도 한 개 이상의 `onNext`를 받을 준비가 되어있어야 한다.
- `Publisher`와 `Subscriber`는 `Subscription`을 공유해야 하며 각각이 고유한 역할을 수행해야 한다.
  - `onSubscribe`와 `onNext` 메서드에서 `request` 메서드를 동기적으로 호출할 수 있어야한다.
  - `cancel`를 몇 번을 호출해도 한 번 호출한 것과 같은 효과를 가져야한다. 여러 번 호출해도 스레드에 안전해야 한다.
  - 같은 `Subscriber` 객체에 다시 가입하는 것은 권장하지 않지만 강제하지는 않는다.
  
### 예제코드
```java
public class TempInfo {
    public static final Random random = new Random();

    private final String town;
    private final int temp;

    public TempInfo(String town, int temp) {
        this.town = town;
        this.temp = temp;
    }

    public static TempInfo fetch(String town) {
        if(random.nextInt(10) == 0) {
            throw new RuntimeException("Error!");
        }
        return new TempInfo(town, random.nextInt(100));
    }

    @Override
    public String toString() {
        return "TempInfo{" +
                "town='" + town + '\'' +
                ", temp=" + temp +
                '}';
    }

    public String getTown() {
        return town;
    }

    public int getTemp() {
        return temp;
    }
}
```

```java
public class TempSubscriber implements Flow.Subscriber<TempInfo> {

    private Flow.Subscription subscription;

    //이벤트를 발행할 때 onSubscribe가 항상 처음 호출된다.
    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    //onNext는 여러번 호출
    @Override
    public void onNext(TempInfo tempInfo) {
        System.out.println( tempInfo );
        subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.println(throwable.getMessage());
    }

    @Override
    public void onComplete() {
        System.out.println("Done!");
    }
}

```

```java
public class TempSubscription implements Flow.Subscription {

    private final Flow.Subscriber<? super TempInfo> subscriber;
    private final String town;

    public TempSubscription(Flow.Subscriber<? super TempInfo> subscriber, String town) {
        this.subscriber = subscriber;
        this.town = town;
    }

    @Override
    public void request(long n) {
        for(long i = 0L; i < n; i++) {
            try {
                subscriber.onNext(TempInfo.fetch(town));
            } catch (Exception e) {
                subscriber.onError(e);
                break;
            }
        }
    }

    @Override
    public void cancel() {
        subscriber.onComplete(); //구독이 취소되면 완료신호를 Subscriber로 전달
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        //Publisher가 이벤트를 발행할 때 호출할 콜백 메서드 Subscriber에 정의
        getTemperatures().subscribe(new TempSubscriber());
    }

    private static Flow.Publisher<TempInfo> getTemperatures() {
        return new Flow.Publisher<TempInfo>() {
            //`Subscriber`가 `Publisher`에 자신을 등록할 때 `Publisher`는 처음으로 `onSubscribe`를 호출해 `Subscription`객체를 전달한다.
            @Override
            public void subscribe(Flow.Subscriber<? super TempInfo> subscriber) {
                subscriber.onSubscribe(new TempSubscription(subscriber, "New York"));
            }
        };
//
//        return subscriber -> subscriber.onSubscribe(
//                new TempSubscription(subscriber, "New York") );
    }
}
```
