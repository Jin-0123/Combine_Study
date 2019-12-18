Combine의 두 가지 핵심 컴포넌트 **publishers**와 **subscribers**를 배워보자.

## Gettring started
* playground를 통해 실습.
* publishers, subscribers, subscriptions에 대하여 배울 것이고, 이것은 combine의 기초를 만들고, 비동기적으로 데이터르 보내고 받는 것을 활성화 시킨다.

## Hello Publishers
* Combine의 핵심인 Publisher 프로토콜.
* 이 프로토콜은 시간에 따라 값의 시퀀스를 한명 혹은 더 많은 subscriber에게 전달하기 위한 요구사항을 정의한다. 즉, publisher는 관심이 있는 값을 포함한 이벤트를 알리고 방출한다. 
* NotificationCenter와 같은 종류로 생각하 수 있다. 실제로, NotificationCenter는 `publisher(for: object:)` 의 이름의 메소드르 가지고 있다. 이것은 Publisher 타입에 브로드캐스트 알림을 보낼 수 있도록 한다.
~~~
example(of: "Publisher") {
  // 1
  let myNotification = Notification.Name("MyNotification")

  // 2
  let publisher = NotificationCenter.default
    .publisher(for: myNotification, object: nil)
}
~~~
 * `publisher(for: object:)` 리턴 값은 Publisher이고, 이것은 default notification center에서 알림이 브로드캐스트할 때 이벤트를 방출시킨다.
 
 * publisher는 두 가지 종류의 이벤트를 방출시킨다. 
    1. 값, 참조되어진 요소
    2. completion event
 * publishers는 0개 혹은 그 이상의 값을 방출할 수 있다. completion event(정상적인 completion event 혹은 error)도 방출할 수 있다. publisher가 completion event를 한버 방출하면 그것은 끝나게 되고 더 이상의 이벤트는 방출하지 않는다. 

예제 다음 라인에 다음의 코드르 작성하고, 확인해보자.
~~~
// 3
let center = NotificationCenter.default

// 4
let observer = center.addObserver(
  forName: myNotification,
  object: nil,
  queue: nil) { notification in
    print("Notification received!")
}

// 5
center.post(name: myNotification, object: nil)

// 6
center.removeObserver(observer)
~~~
* 예제를 빌드해보면, 콘솔에 output이 출력된다. 
* 이 예제에서는 output이 실제로 publisher에서부터 나온 것은 아니다. 그렇게하려면 subscriber가 필요하다.
 
 ## Hello Subscriber
* subscriber publisher로부터 input을 받을 수 있는 요구사항을 정의하는 프로토콜이다. 
* publisher는 최소 한명의 subscriber가 있어야 이벤트를 방출한다.

### Subscribing with _sink(_:_:)_
~~~
example(of: "Subscriber") {
  let myNotification = Notification.Name("MyNotification")
  let publisher = NotificationCenter.default.publisher(for: myNotification, object: nil)
  let center = NotificationCenter.default
  
  // 1
  let subscription = publisher
    .sink { _ in
      print("Notification received from a publisher!")
    }
    
  // 2
  center.post(name: myNotification, object: nil)
  
  // 3
  subscription.cancel()
}
~~~
**_sink:_**
> it simply provides an easy way to attach a subscriber with closures to handle output from a publisher
* `sink` 메소드는 publisher로 부터 ouput을 다루기 위한 클로저를 가진 subsciber를 첨부하는 간단한 방법이다.
이 예제에서는 클로저들을 무시하고 대신에 알림을 받았을 때, 메시지를 출력하도록 했다.
* `sink` 오퍼레이터는 publisher가 방출하는 많은 값들을 계속해서 받을 것이다. 이것은 _unlimited demand_ 로 알려져있고, 곧 더 자세히 배우게 될 것이다.
* 이 예제에서는 sink 오퍼레이터 제공하는 두 가지 클로저를 사용하지는 않았지만, 그 클로저를 설명하자면 하나는 completion event를 받아 핸들링하는 것이고 다른 하나는 값을 받아 핸들링하는 것이다.


~~~
example(of: "Just") {
  // 1. Just를 사용해서 publisher를 만든다. 이것은 초기값을 넣어서 publisher를 생성한다.
  let just = Just("Hello world!")
  
  // 2. publisher에 subscription을 생성한다. 그리고 이벤트를 받았을 때 각 메시지를 출력한다.
  _ = just
    .sink(
      receiveCompletion: {
        print("Received completion", $0)
      },
      receiveValue: {
        print("Received value", $0)
    })
}
~~~

**_Just:_**
> A publisher that emits an output to each subscriber just once, and then finishes.
* publisher로써 각 subscriber에 output을 방출하고 끝난다.
* 다른 subsciber를 추가하면 새 subsciber에도 Just publisher의 output이 출력되고, 끝난다.

### Subscribing with _assign(to: on:)_
assign(to: on:) 객체의 KVO-compliant 프러퍼티에서 받은 값을 할당할 수 있도록 한다.
~~~
example(of: "assign(to:on:)") {
  // 1. 클래스와 변수를 정의하고, didSet 옵저버에 값을 출력하도록한다.
  class SomeObject {
    var value: String = "" {
      didSet {
        print(value)
      }
    }
  }
  
  // 2. 클래스의 인스턴스르 생성한다.
  let object = SomeObject()
  
  // 3. 스트링 배열을 방출하는 publisher를 생성한다.
  let publisher = ["Hello", "world!"].publisher
  
  // 4. publisher를 subscibe하고, 객체의 value에 각 값(Hello, world)을 할당한다.
  _ = publisher
    .assign(to: \.value, on: object)
}
~~~

## Hello Cancellable
