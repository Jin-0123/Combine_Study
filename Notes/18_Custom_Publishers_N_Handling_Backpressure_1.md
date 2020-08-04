# Chapter 18: Custom Publishers & Handling Backpressure (1)

## Creating your own publishers

* publishers 만드는 방법
	* 퍼블리셔 네임스페이스의 간단한 익스텐션 메서드 사용.
	* 값을 생성하는 Subscription로 Publishers 네임스페이스 안에 타입 구현
	* 위와 동일하지만 업스트림 값을 변환한 Subscription로 구현
* 철커스텀 서브스크립션 없이 커스텀 퍼블리셔를 생성할 수는 있지만 추천하지는 않음. 취소당할 수 있음.

## Publishers as extension methods

* 기존 연산자를 사용해서 간단한 연산자 만들기
* `unwrap()` 연산자 만들기: 옵셔널 값을 언랩핑하고 nil 값은 무시

~~~
extension Publisher {
	func unwrap<T>() -> Publishers.CompactMap<Self, T> where Output == Optional<T> {
		compactMap { $0 }
	}
}
~~~
> * `compactMap(_:)` 사용해서 구현.
> * 편리하게 사용할 수 있도록 연산자를 Optional 타입으로 제한 (where 절)

* 연산자를 체인하는 경우처럼 복잡한 연산자를 구현할 때는 sinature가 조금 복잡해질 수 있음, 그 때 좋은 방법은 `AnyPublisher<OutputType, FailureType>` 리턴하는 것. 끝에 `eraseToAnyPublisher()`를 사용해서 타입지우고 리턴.

## Testing your custom operator

~~~
let values: [Int?] = [1, 2, nil, 3, nil, 4]

values.publisher 
	.unwrap() 
	.sink {
		print("Received value: \($0)")
	}
~~~

* 퍼블리셔 두 가지 타입으로 그룹핑
	* 생산자처럼 행동, 직접 값을 생산하는 퍼블리셔
	* 트랜스포머처럼 행동, 업스트림 퍼블리셔로부터 생산된 값을 변형하는 퍼블리셔
	
## The subscription mechanism
1. 구독자는 퍼블리셔를 구독한다.
2. 퍼블리셔는 구독을 생성하고 구독자에게 넘겨준다. `receive(subscription:)`를 콜한다.
3. 구독자는 구독으로부터 값을 요청한다. 원하는 값의 수를 보내면서. 구독의 `request(_:)`를 콜한다.
4. 구독은 작업을 시작하고, 값을 방출한다. 구독자에게 하나씩 보낸다. 구독자의 `receive(_:)`를 콜한다.
5. 값을 받으면 구독자는 새 Subscribers.Demand 를 리턴한다. 이전의 토탈 디맨드를 더하는.
6. 구독은 값을 계속 보낸다. 요청한 수에 도달할 때까지.
7. 마지막으로 오류가 있거나 완료되면 구독자의 `receive(completion:)` 메서드가 불림.


## Publishers emitting values

* 타이머 만들어보기
~~~
struct DispatchTimerConfiguration {
 	// 특정 큐에서 타이머를 시작하기를 원한다면, 옵셔널이라 신경쓰고 싶지 않으면 안넣어도 됨
	let queue: DispatchQueue?
	// 타이머가 시작할 때 인터벌, 구독 타임으로부터 시작됨.
	let interval: DispatchTimeInterval
	// 타이머 이벤트가 지연됬을 때의 데드라인 이후 최대 시간의 양
	let leeway: DispatchTimeInterval
	// 방고 싶은 타이머 이벤트의 수
	let times: Subscribers.Demand
}
~~~

### Adding the DispatchTimer publisher

~~~
extension Publishers {
	struct DispatchTimer: Publisher {
		typealias Output = DispatchTime
		typealias Failure = Never

		let configuration: DispatchTimerConfiguration
		
		init(configuration: DispatchTimerConfiguration) {
			self.configuration = configuration        
    	}
		
		// 퍼블리셔 프로토콜 필수 메서드인 receive 작성.
		func receive<S: Subscriber>(subscriber: S) where Failure == S.Failure, Output == S.Input {
			let subscription = DispatchTimerSubscription(subscriber: subscriber, configuration: configuration)
			// 구독자들에게 요청 값을 받으려면 구독자에게 구독을 받아라! 
			subscriber.receive(subscription: subscription)
        }
    } 
}
~~~

### Building your subscription

* 구독의 역할은?
	* 구독자들로부터 초기 디맨드를 받을 것.
	* 수요에 따라 타이머 이벤트를 생성할 것.
	* 구독자에게 값을 받고 반환할 때마다 수요 카운트에 추가할 것.
	* 요청한 것보다 더 많은 값을 전달하지 않도록 하라.
	
~~~
private final class DispatchTimerSubscription <S: Subscriber>: Subscription where S.Input == DispatchTime { }
~~~
> * 이 구독은 구독 프로토콜을 통해서만 볼 수 있으므로 private.
> * 참조를 전달하기 원하기에 클래스. 구독자들은 Cancellable 컬렉션에 추가할 뿐만아니라 유지하고 독립적으로 `cancel()`을 호출한다.
> * 인풋 타입이 DispatchTime 인 구독자들에게 값을 방출

### Adding required properties to your subscription

struct DispatchTimerConfiguration {
 	// ...
	// 가입자가 패스되는 컨피규레이션 
	let configuration: DispatchTimerConfiguration 
	// 컨피규레이션으로부터 복사된 타이머의 최대 수. 값을 보낼때마다 감소하는 카운터
	var times: Subscribers.Demand
	// 구독자들로부터의 현재 수요. 값을 보낼 때마다 감소.
	var requested: Subscribers.Demand = .none 
	// 타이머 이벤트를 생성할 내부 DispatchSourceTimer
	var source: DispatchSourceTimer? = nil
	// 가입자. 구독이 완료, 실패, 취소되지 않는 한 구독자를 유지해야함.
	var subscriber: S?	
}

### Initializing and canceling your subscription
### Letting your subscription request values
### Activating your timer

~~~
private final class DispatchTimerSubscription<S: Subscriber>: Subscription where S.Input == DispatchTime {
	// 가입자가 패스되는 컨피규레이션 
    let configuration: DispatchTimerConfiguration
	// 컨피규레이션으로부터 복사된 타이머의 최대 수. 값을 보낼때마다 감소하는 카운터
    var times: Subscribers.Demand
	// 구독자들로부터의 현재 수요. 값을 보낼 때마다 감소.
    var requested: Subscribers.Demand = .none
	// 타이머 이벤트를 생성할 내부 DispatchSourceTimer
	var source: DispatchSourceTimer? = nil
	// 가입자. 구독이 완료, 실패, 취소되지 않는 한 구독자를 유지해야함.
    var subscriber: S?
    
    init(subscriber: S, configuration: DispatchTimerConfiguration) {
	    self.configuration = configuration
		self.subscriber = subscriber
		self.times = configuration.times
	}
    
    func cancel() {
		source = nil
		subscriber = nil
    }
    
    func request(_ demand: Subscribers.Demand) {
		guard times > .none else {
	    	subscriber?.receive(completion: .finished)
	    	return
    	}
		// 새로운 수요가 들어오면 현재 수요 값에 더함
		requested += demand
		
		if source == nil, requested > .none { 
			// 옵셔널 큐에 타이머 생성
			let source = DispatchSource.makeTimerSource(queue: configuration.queue)
			// 컨피규레이션 인터벌 초 이후 타이머 
			source.schedule(deadline: .now() + configuration.interval, repeating: configuration.interval, leeway: configuration.leeway)
			// 타이머의 이벤트 핸들러. 약한 참조하지 않으면 메모리 해제 안됨.
			source.setEventHandler { [weak self] in
				// 현재 요청된 값 있는지 확인.
				guard let self = self,
				self.requested > .none else { return }
				// 방출된 두개의 카운터 감소시키기
				self.requested -= .max(1) 
				self.times -= .max(1)
				// 구독자에게 보냄.
				_ = self.subscriber?.receive(.now())
				// 타이머가 더 이상 없다면 끝난 것으로 하고 종료!
				if self.times == .none {
					self.subscriber?.receive(completion: .finished) 
				}
			}
			self.source = source 
			source.activate()
		}
	}
}
~~~

* 퍼블리셔 익스텐션에 DispatchTimerSubscription 정의 추가. 더 쉽게 체인할 수 있도록.
~~~
extension Publishers {
	static func timer(queue: DispatchQueue? = nil, interval: DispatchTimeInterval, leeway: DispatchTimeInterval = .nanoseconds(0), times: Subscribers.Demand = .unlimited) -> Publishers.DispatchTimer {
		return Publishers.DispatchTimer(configuration: .init(queue: queue, interval: interval, leeway: leeway, times: times))
	}
}
~~~

## Testing your timer

~~~
var logger = TimeLogger(sinceOrigin: true)
let publisher = Publishers.timer(interval: .seconds(1), times: .max(6))
let subscription = publisher.sink { time in
	print("Timer emits: \(time)", to: &logger)
}

DispatchQueue.main.asyncAfter(deadline: .now() + 3.5) {
	subscription.cancel() 
}
~~~


## Publishers transforming values

* 다음을 따르는 오퍼레이터 만들기
	* 첫번째 구독자에게 업스트림 퍼블리셔로부터 구독한다.
	* 각각 새 구독자에게 마지막 N 값을 리플레이
	* 사전에 방출된 완료 이벤트를 릴레이한다.

### Implementing a ShareReplay operator
### Initializing your subscription
### Sending completion events and outstanding values to the subscriber

~~~
fileprivate final class ShareReplaySubscription<Output, Failure: Error>: Subscription {
	let capacity: Int
	var subscriber: AnySubscriber<Output,Failure>? = nil 
	var demand: Subscribers.Demand = .none
	var buffer: [Output]
	var completion: Subscribers.Completion<Failure>? = nil
	
	init<S>(subscriber: S, replay: [Output], capacity: Int, completion: Subscribers.Completion<Failure>?) where S: Subscriber, Failure == S.Failure, Output == S.Input {
		self.subscriber = AnySubscriber(subscriber)
		self.buffer = replay
		self.capacity = capacity
		self.completion = completion 
	}
	
	private func complete(with completion: Subscribers.Completion<Failure>) {
		guard let subscriber = subscriber else { return } 
		self.subscriber = nil
		self.completion = nil
		self.buffer.removeAll()
		subscriber.receive(completion: completion)
	}
	
	 private func emitAsNeeded() {
		guard let subscriber = subscriber else { return }
		// 버퍼가 있고, 처리되지 않은 수요가 있을 때 방출.
		while self.demand > .none && !buffer.isEmpty { 
			self.demand -= .max(1)
			// 구독자에게 처리되지 않은 버퍼 보내고, 다음 수요 받음.
			let nextDemand = subscriber.receive(buffer.removeFirst())
			if nextDemand != .none {
				// 새로운 수요 있으면 총 수요에 포함.
				self.demand += nextDemand 
			}
		}
		// 완료이벤트 있으면 보내기.
		if let completion = completion {
			complete(with: completion)
		}
	}
	
	func request(_ demand: Subscribers.Demand) { 
		if demand != .none {
			self.demand += demand 
		}
  		emitAsNeeded()
	}
	
	func cancel() {
		complete(with: .finished) 
	}
	
	func receive(_ input: Output) {
		guard subscriber != nil else { return }
		// 처리되지 않은 버퍼 추가. 무한 수요같은 경우에 대한 처리가 필요하지만 현재는 괜춘.
		buffer.append(input)
		// 요청된 것보다 많은 값을 허용하지 말 것. 선입선출.
		if buffer.count > capacity {
			buffer.removeFirst() 
		}
		// 결과 구독자에게 전달.
		emitAsNeeded()
	}
	
	// 구독자 제거, 버퍼 비우기. 메모리 관리로 굿.
	func receive(completion: Subscribers.Completion<Failure>) { 
		guard let subscriber = subscriber else { return } 
		self.subscriber = nil
		self.buffer.removeAll()
		subscriber.receive(completion: completion) 
	}
}
~~~
