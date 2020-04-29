# Chapter 13: Resource Management

* 여러 구독자들 사이에서 하나의 리소스 결과만을 공유할 수 있도록 하지 위한 오퍼레이터: `share()`
* 리소스를 관리하기 위한 두 가지 오퍼레이터: `share()`와 `multicast(_:)`

## The share() operator
* 이 오퍼레이터의 목적은 퍼블리셔를 레퍼런스로 획득 하는 것.
* 퍼블리셔는 보통 구조체임, 우리가 함수나 저장된 프러퍼티를 넘길 때, 스위프트는 몇 번씩 복사함.(밸류 타입이기 때문)
* `share()` 오퍼레이터는 버블리셔 인스턴스를 리턴. 업스트림 퍼블리셔를 공유한다. 
* 리퀘스트가 완료된 후에 두번째 구독자가 발생한다면? 아무것도 받게 되지 못한다. 

## The multicast(_:) operator
* `multicast(_:)`는 `ConnectablePublisher` 리턴.
* `connect()` 불릴 때까지 구독하지 않는다. 
* `autoconnect()`메서드를 제공. `share()`와 비슷하게 동작 > 처음 구독을 시작하면 즉시 동작. 하나의 값만 방출하거나 구독자들이 `CurrentValueSubject`사용해서 쉐어한다면 유용

## Future
~~~
let future = Future<Int, Error> { fulfill in
	do {
		let result = try performSomeWork()
		fulfill(.success(result)) 
	} catch {
		fulfill(.failure(error)) 
	}
}
~~~
* Future는 클래스
* 생성 후, 즉시 실행
* Promiss 결과 값 저장 후, 현재와 미래의 구독자들에게 전달
