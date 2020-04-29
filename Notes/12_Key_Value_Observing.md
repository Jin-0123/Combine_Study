# Chapter 12: Key-Value Observing

## Introducing publisher(for:options:)

## Preparing and subscribing to your own KVO-compliant properties
* 객체가 구조체가 아닌 클래스이고, NSObject를 상속받고 있는 경우
* `@objc dynamic` 속성으로 마크되어 있는 프로퍼티

~~~
// 1. NSObject 프로토콜을 상속 받고 있는 클래스 생성(KVO를 위한 필수조건)
class TestObject: NSObject {
	// 2. @objc dynamic 붙여서 프러퍼티 생성
 	@objc dynamic var integerProperty: Int = 0
}
let obj = TestObject()
// 3. integerProperty 옵저빙하는 퍼블리셔 생성하고 구독
let subscription = obj.publisher(for: \.integerProperty) .sink {
    print("integerProperty changes to \($0)")
  }
// 4. 프러퍼티 업데이트
obj.integerProperty = 100 
obj.integerProperty = 200
~~~

* Objc가 브릿지된 스위프트의 타입, 즉 스위프트의 배열, 딕셔너리에서는 KBO가 잘 동작함. 
* 그러나 순수 스위프트 타입의 경우에는 사용할 수 없다.

## Observation options
* `.inital` 초기 값 방출
* `.prior` 변화가 일어났을 때, 이전의 값과 새 값 모두 방출 
* `.old`와 `.new` 퍼블리셔에서는 잘 사용되지 않는다. 둘 다 아무것도 하지 않음

## ObservableObject
* NSObject 프로토콜을 따르지 않더라도 ObservableObject를 따르고, `@Published`로 프러퍼티를 감싸준다면 `objectWillChange` 퍼블리셔가 자동으로 생성된다.
* `objectWillChange`는 어떤 프러퍼티로 변경되었는지 알 수는 없음. 
