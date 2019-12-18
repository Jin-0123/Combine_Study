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
  
 예제를 빌드해보면, 콘솔에 output이 출력된다. 
 이 예제에서는 output이 실제로 publisher에서부터 나온 것은 아니다. 그렇게하려면 subscriber가 필요하다.
 
 ##
 예제를 빌드해보면, 콘솔에 output이 출력된다. 
