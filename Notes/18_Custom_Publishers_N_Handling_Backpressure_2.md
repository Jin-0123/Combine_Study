# Chapter 18: Custom Publishers & Handling Backpressure (2)

## Coding your publisher

* 퍼블리셔는 Publishers 네임스페이스 안에 struct로 구현되지만 가끔은  `multicast()` 리턴 값인 Publishers.Multicast 같은 클래스로 구현할 때도 있음.

~~~
extension Publishers {
	// 여러 구독자가 한 인스턴스를 공유하기 떄문에 struct 대신 class
	final class ShareReplay<Upstream: Publisher>: Publisher {
		// 새로운 퍼블리셔는 업스트림의 아웃풋이나 실패 타입을 바꾸지 않는다.
		typealias Output = Upstream.Output
		typealias Failure = Upstream.Failure 
	}
}
~~~

## Adding the publisher’s required properties

~~~
// 여러 구독자가 동시에 접근 가능하므로 독점 접근을 보장하기위한 lock이 필요.
private let lock = NSRecursiveLock()
// 업스트림 퍼블리셔 참조 유지. 구독 라이프사이클에서 필요함.
private let upstream: Upstream
// 버퍼 용량
private let capacity: Int
// 기록된 값 저장
private var replay = [Output]()
// 멀티 구독자들에게 이벤트를 알려주기 위해 유지. `ShareReplaySubscription`으로 값을 받음.
private var subscriptions = [ShareReplaySubscription<Output, Failure>]()
// 완료된 이후에 값을 리플레이할 수 있음. 업스트림 퍼블리셔가 완료되었는지 기억해야함.
private var completion: Subscribers.Completion<Failure>? = nil
~~~

## Initializing and relaying values to your publisher

* 이닛 메서드 작성
~~~
init(upstream: Upstream, capacity: Int) { 
	self.upstream = upstream
	self.capacity = capacity 
}

private func relay(_ value: Output) { 
	// 멀티 구독자들과 공유해야하기 때문에 락을 통해 접근 시 값 보장.
	lock.lock()
	defer { lock.unlock() }
	// 완료되지 않은 경우에만 값 전달
  	guard completion == nil else { return }

	replay.append(value)
	if replay.count > capacity {
		replay.removeFirst() 
	}
	// 각 구독자들에게 버퍼된 값 전달.
	subscriptions.forEach {
		_ = $0.receive(value) 
	}
}
~~~

## Letting your publisher know when it’s done

~~~
private func complete(_ completion: Subscribers.Completion<Failure>) {
	lock.lock()
	defer { lock.unlock() }
	
	self.completion = completion
	subscriptions.forEach {
		_ = $0.receive(completion: completion) 
	}
}
~~~

* receive 메서드 구현, 새로운 구독을 반들고 구독자들에게 전달
~~~
// 퍼블리셔의 Output, Failure 타입과 일치하는 구독자들을 받도록 명시
func receive<S: Subscriber>(subscriber: S) where Failure == S.Failure, Output == S.Input { 
	lock.lock()
	defer { lock.unlock() } 
}
~~~

## Creating your subscription

~~~
// 새로운 구독은 구독자를 참조하고, 현재 리플레이 버퍼, 용량, 완료이벤트를 받는다.
let subscription = ShareReplaySubscription(
	subscriber: subscriber,
	replay: replay,
	capacity: capacity,
	completion: completion)

// 이벤트를 넘겨받기 위해 구독을 유지한다.
subscriptions.append(subscription)

// 구독자에게 구독을 보내면 값 요청을 시작할 수 있음.
subscriber.receive(subscription: subscription)
~~~

## Subscribing to the publisher and handling its inputs

~~~
func receive<S: Subscriber>(subscriber: S) where Failure == S.Failure, Output == S.Input { 
	lock.lock()
	defer { lock.unlock() } 
	// 업스트림 퍼블리셔는 한번만 구독한다.
	guard subscriptions.count == 1 else { return }
	let sink = AnySubscriber(
		// 다루기 쉬운 AnySubscriber 클래스 사용. 즉시 `.unlimit` 요청해서 완료되도록 한다.
		receiveSubscription: { subscription in	
			subscription.request(.unlimited) 
		},
		// 받은 값을 다운스트림 구독자들에게 전달한다.
		receiveValue: { [weak self] (value: Output) -> Subscribers.Demand in
			self?.relay(value)
			return .none
		},
		// 업스트림으로부터 받은 완료이벤트로 퍼블리셔를 완료한다.
		receiveCompletion: { [weak self] in
			self?.complete($0) 
		})
	upstream.subscribe(sink)
}
~~~
* 리테인 사이클 방지를 위해 weak 키워드 사용

## Adding a convenience operator

~~~
extension Publisher {
	func shareReplay(capacity: Int = .max) -> Publishers.ShareReplay<Self> {
		return Publishers.ShareReplay(upstream: self, capacity: capacity)
	}
}
~~~

## Testing your subscription

~~~
// 타임로거 정의
var logger = TimeLogger(sinceOrigin: true)
// 다른 시간에 값을 보내는 것을 시뮬레이트하기 위해서 subject를 사용
let subject = PassthroughSubject<Int,Never>()
// subject를 공유, 마지막 두 값만 전달
let publisher = subject.shareReplay(capacity: 2)
// 첫번째 값 보내기, 구독자 없어서 결과는 안찍힘.
subject.send(0)
~~~

* 구독만들고 값 보내기
~~~
let subscription1 = publisher.sink(
	receiveCompletion: {
		print("subscription2 completed: \($0)", to: &logger)
	},
	receiveValue: {
		print("subscription2 received \($0)", to: &logger)
	}
)
	
subject.send(1) 
subject.send(2) 
subject.send(3)
	
let subscription2 = publisher.sink(
	receiveCompletion: {
		print("subscription2 completed: \($0)", to: &logger)
	},
	receiveValue: {
    	print("subscription2 received \($0)", to: &logger)
	}
)

subject.send(4)
subject.send(5) 
subject.send(completion: .finished)

// 약간의 지연이 있는 구독 추가
var subscription3: Cancellable? = nil
DispatchQueue.main.asyncAfter(deadline: .now() + 1) { 
	print("Subscribing to shareReplay after upstream completed") 
	subscription3 = publisher.sink(
		receiveCompletion: {
			print("subscription3 completed: \($0)", to: &logger)
		},
		receiveValue: {
			print("subscription3 received \($0)", to: &logger)
		}
	) 
}
~~~
* 구독의 할당이 해제되면 종료되는 것을 기억할 것. 지연된 것을 유지하려면 변수를 사용해야함.
* 0은 안찍힘. 첫번째 구독자가 shared publisher를 구독하기 전이라서.
* 모든 값은 현재, 미래의 구독자들에게 전달된다.
* subscription2에는 마지막 2개 값만 전달
* subject완료된 이후에 subscription3생성됨. 마지막 2개의 값 여전히 받음.
* shared publisher가 완료된 이후에도 completion 이벤트 잘 전달됨.

## Verifying your subscription

* 프린트 찍어서 publisher를 한번만 구독했는지 확인해보자.
~~~
// let publisher = subject.shareReplay(capacity: 2)
let publisher = subject 
	.print("shareReplay")
	.shareReplay(capacity: 2)
~~~

## Handling backpressure

* publisher로 부터 오는 값을 저항하고자 하는 것
	* 센서와 값은 매우 빈번히 데이터를 처리해야하는 경우
	* 큰 파일을 전달할 때
	* 데이터 업데이트 시 복잡한 UI 렌더링
	* 유저 인풋 기다리기
	* 들어오는 속도를 구독자가 유지할 수 없는 데이터 처리할 때
	
* publisher-subsciber 매커니즘은 유연하다. push의 반대인 pull로 고안되어있음. 이것의 의미는 구독자들이 퍼블리셔가 값을 방출하는 것을 요청하고, 얼마나 받을 것인지 명시하는 것을 말한다.
* 구독자가 새 값을 받을 때마다 수요는 업데이트한다.
* 구독자가 받고자 하면 열고, 더 이상 받기를 원하지 않으면 닫음.

* 더 많은 값이 들어왔을 때 일어나는 일은 내가 어떻게 고안하느냐에 따라 달려있음.
	* 흐름을 제어한다. pubilsher가 처리할 수 있는 것 이상의 것을 보내지 않도록.
	* 처리할 수 있을 때까지 값을 버퍼한다. 메모리 리스크가 있음.
	* 처리하지 못하면 드랍.
	* 요구 사항에 따라서 위에 것을 조합한다.
	
* backpressure를 처리하는 방식
	* 정체를 처리하는 커스턴 subscription을 가진 publisher
	* 퍼블리셔 체인의 끝에서 값을 전달하는 구독자
	
## Using a pausable sink to handle backpressure

~~~
protocol Pausable {
	var paused: Bool { get }
	func resume()
}
~~~
* `pause()` 메서드는 필요없음. 각 값을 받을 때 일시정지를 결정하기때문.
* 더 정교하게 정지해야하는 구독자는 `pause()`를 부를 수도 있음.

~~~
//  pausable subscriber는  Pausable, Cancellable 둘다 따름. 이 객체는 pausableSink가 반환할 오브젝트임, 구조체가 아닌 클래스로 구현한 이유는 값이 복사되기를 원하지 않고, 라이프사이클에서 특정 지점에 변경 가능성이 있음.
final class PausableSubscriber<Input, Failure: Error>: Subscriber, Pausable, Cancellable {
	// publisher 스크림을 관리하고 최적화하기위해서 유니크한 identifier 제공.
	let combineIdentifier = CombineIdentifier()
	// true는 더 많은 값을 받을 수 있음을 나타냄, false는 구독 일시중지를 가리킴
	let receiveValue: (Input) -> Bool
	// publisher로부터 완료 이벤트를 받으면 completion 클로저가 불리게 됨.
	let receiveCompletion: (Subscribers.Completion<Failure>) -> Void
	// 일시중지 후 더 많은 값을 요청하기 위해서 구독을 유지함, 리테인 사이클을 피하기 위해 더 이상 필요하지 않으면 nil로 설정
	private var subscription: Subscription? = nil
	// Pausable 프로토콜 속성에 따라 paused 프러퍼티 노출
	var paused = false
	
	// 두 개의 클로저는 새 값을 받을 때, 완료이벤트를 받을 때 호출됨. sink와 유사, 그러나 더 많은 값을 받을 수 있는지 보류할 것인지를 나타내는 Bool값이 있는 것이 다름.
	init(receiveValue: @escaping (Input) -> Bool, receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void) {
		self.receiveValue = receiveValue
		self.receiveCompletion = receiveCompletion 
	}

	// 구독 취소할 때 리테인사이클을 피하기 위해 nil을 설정하는 것을 잊지 말 것.
	func cancel() { subscription?.cancel()
	  subscription = nil
	}
	
	func receive(subscription: Subscription) {
		// 나중에 일시정지 할 수 있도록 저장
		self.subscription = subscription 
		// 즉시 값 하나를 요청. 구독자는 일시정지를 할 수 있는데, 일시정지가 언제 필요한지 예측할 수 없다. 한번에 하나씩만 요청
		subscription.request(.max(1))
	}
	
	func receive(_ input: Input) -> Subscribers.Demand { 
		// 새 값을 받으면 receiveValue 호출하고 paused 상태 업데이트
		paused = receiveValue(input) == false 
		// 일시 정지된 경우 `.none` 이라면 더 이상 값을 원하지 않음을 나타냄, 그렇지 않으면 사이클을 유지하기 위해 1개 더 요청
		return paused ? .none : .max(1)
	}
	
	func receive(completion: Subscribers.Completion<Failure>) { 
		// 완료이벤트를 받으면 receiveCompletion 호출 더 이상 필요하지 않으므로 subscription은 nil
		receiveCompletion(completion)
		subscription = nil
	}
	
	func resume() {
		guard paused else { return }
		paused = false
		// 일시정지된 publishr는 다시 시작할 때 값 하나를 요청한다.
		subscription?.request(.max(1))
	}
}
~~~

~~~
extension Publisher {
	func pausableSink(receiveCompletion: @escaping ((Subscribers.Completion<Failure>) -> Void), receiveValue: @escaping ((Output) -> Bool)) -> Pausable & Cancellable {
		// PausableSubscriber를 인스턴스화하고 self를 구독한다.
		let pausable = PausableSubscriber(receiveValue: receiveValue, receiveCompletion: receiveCompletion) 
		self.subscribe(pausable)
		// 구독자는 구독을 재개하고 취소하는데에 사용할 객체
		return pausable
	} 				
}
~~~

## Testing your new sink

* 홀수마다 pause, 타이머로 재개
~~~
let subscription = [1, 2, 3, 4, 5, 6]
	.publisher
	.pausableSink(receiveCompletion: { completion in
		print("Pausable subscription completed: \(completion)") 
	}) { value -> Bool in
		print("Receive value: \(value)")
	    if value % 2 == 1 {
			print("Pausing")
			return false
		}
		return true
	}
	
let timer = Timer.publish(every: 1, on: .main, in: .common) 
				.autoconnect()
				.sink { _ in
					guard subscription.paused else { return } 
					print("Subscription is paused, resuming") 
					subscription.resume()
				}
~~~
