# Chapter 8: In Practice: Project "Collage"

* UIKit 뷰 컨트롤러에서 컨바인 퍼블리셔 사용하기
* 컨바인으로 이벤트 다루기
* 뷰 컨트롤러 네비게이팅과 퍼블리셔를 통해 데이터 교환하기
* 오퍼레이터 사용해서 서브스크립션 만들어 로직 구현하기
* 컨바인 코드 안에서 Cocoa API 감싸서 편리하게 사용하기

## Getting started with "Collage"

* `private var subscription = Set<AnyCancellable>()`
* 뷰 컨트롤러의 라이프사이클과 묶여있는 변수, UI 관련된 subsctiption을 저장하는 collection

* UI 컨트롤을 바인딩할 때 가장 알맞는 타입은 `CurrentValueSubject`
* 적어도 하나의 값을 방출하는 것을 보장한다. 한번도 초기값을 선언하지 않았더라도. 이런 방식은 잘못된 상태가 되었을 때, 초기값을 계속 기다리지 않아도 된다.

~~~
images
	.handleEvents(receiveOutput: { [weak self] (photos) in
		self?.updateUI(photos: photos)
	})
~~~
* 이벤트에 따라 뷰 업데이트, 이벤트 방출할 때 사이드 이펙트 실행 (> chap 10에서 자세히)

## Talking to other view controllers

## Wrapping a callback function as a future

* `PHPhotoLibrary.performChangesAndWait(_)` 비동기적으로 포토라이브러리에 접근, 메인 쓰레드가 블락될 일은 없음

### A note on memory management

* 만약 메모리에서 해제될 수 있는 객체를 캡쳐링했다면 self를 사용하기 보다 [weak self]를 사용하거나 다른 variable을 사용해야한다. 
* 만약 메모리에서 해제될 수 없는 객체를 캡쳐링했다면 안전하게 [unowned self] 사용.

## Presenting a view controller as a future
* Future로 얼럿 뷰컨트롤러 띄우기
~~~
func alert(title: String, text: String?) -> AnyPublisher<Void, Never> {
	let alertVC = UIAlertController(title: title,
									message: text,
									preferredStyle: .alert)
	return Future { resolve in
		alertVC.addAction(UIAlertAction(title: "Close", style: .default) { _ in
			resolve(.success(()))
		})
		self.present(alertVC, animated: true, completion: nil) }
		.handleEvents(receiveCancel: {
			self.dismiss(animated: true)
		})
		.eraseToAnyPublisher()
}
~~~

* alert 호출
~~~
alert(title: title, text: description)
	.sink(receiveValue: { _ in })
	.store(in: &subscriptions)
~~~

## Sharing subscriptions

* 여러 개의 subscription을 만든다면?
* `share()` 오퍼레이터를 통해 오리지널 퍼블리셔를 공유할 수 있음. 하지만 새 구독이 들어왔을 때 이전 값을 다시 방출해주지는 않음. 그럴 경우를 대비해서 `shareReplay()` 오퍼레이터가 있음.

## Publishing properties with **@Published**
* ` @Published` 프러퍼티 랩퍼는 자동적으로 퍼블리셔에게 추가되는 것을 허가한다. 오리지널 프로퍼티의 값이 바뀔 때마다 새로운 출력값을 방출하는 프러퍼티를 지원하기위해.

~~~
struct Person {
 	@Published var age: Int = 0
}
~~~
* $age로 방출되는 새 값을 받을 수 있음.
* 초기값이 필수.

## Operators in practice

### Updating the UI after the publisher completes
~~~
newPhotos
	.ignoreOutput()
	.delay(for: 2.0, scheduler: DispatchQueue.main)
	.sink(receiveCompletion: { [unowned self] _ in
		self.updateUI(photos: self.images.value)
	}, receiveValue: { _ in })
	.store(in: &subscriptions)
~~~
* `ignoreOutput()` 방출된 값은 무시하고, 오로지 completion 이벤트만 전달한다.
* `delay(for:scheduler:)` 주어진 초만큼 기다린다. 

### Accepting values while a condition is met
~~~
let newPhotos = photos.selectedPhotos
  .prefix(while: { [unowned self] _ in
  	// 사진 6개로 제한
    return self.images.value.count < 6
  })
  .share()
~~~

* `share()` 부르기 전에 `prefix(while:)`가 불려야 모든 subcriptions에서 적용된다.

