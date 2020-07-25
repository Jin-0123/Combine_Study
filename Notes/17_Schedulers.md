# Chapter 17: Schedulers

## An introduction to schedulers

- schduler: 언제 어떻게 클로저를 실행할지 정하는 프로토콜
- 스케줄러는 다음 액션이 실행되는 컨텍스트를 제공, 액션은 클로저, 프로토콜 안에 정의되어있는..
- 클로저의 텀은 퍼블리셔에 의해 전달되는 값을 숨길 수 있음. 특정 스케줄러에서 동작하는.
- 어느 쓰레드에서 동작하는지는 어떤 스케줄러를 선택했느냐에 따라 달라진다.
- 스케줄러는 쓰레드와 다르다.
- 유저 액션은 메인쓰레드 > 이거는 어떤 동작을 트리거, 백그라운드 스케줄러에서 > 전달된 데이터를 화면에 업데이트, UI 업데이트는 메인쓰레드
- 선택한 implementation에 따라 시리즈하게 동작하던지, 병렬로 동작할 수 있음.

## Operators for scheduling

- `subscribe(on:)`, `subscribe(on:options:)`: 특정 스케줄러에서 구독을 생성 **create**
- `receive(on:)`, `receive(on:options:)`: 특정 스케줄러에서 값들을 전달 **deliver**

## Introducing subscribe(on:)

- 퍼블리셔는 구독 전까지 실행 노노
- 퍼블리셔를 구독할 때 실행 단계
    1. Publicsher는 구독자를 받고, Subscription을 생성한다.
	2. Subscriber는 구독을 받고, 퍼블리셔로부터 값을 요청한다.
	3. 퍼블리셔는 구독을 통해 동작을 시작한다.
	4. 퍼블리셔는 구독을 통해 값을 방출한다.
	5. 오퍼레이터는 값을 변형한다.
	6. 구독자들은 마지막 값을 받는다.
- `subscribe(on:)`: 오퍼레이터를 사용하면 특정한 스케줄러에서 오퍼레이터를 동작시킬 수 있다.

## Introducing receive(on:)

~~~
.receive(on: DispatchQueue.main)
~~~
> 백그라운드로부터 결과가 방출되었더라고 메인쓰레드에서 값을 받는 것이 보장

## Scheduler implementations

- ImmediateScheduler: 
현재의 쓰레드에서 즉시 코드를 실행하는 단순한 스케줄러, `subscribe(on:)`, `receive(on:)`을 사용해서 변경하지 않았더라면 디폴트 컨텍스트이다.
- RunLoop: Foundation’s Thread object에서 시도
- DispatchQueue: 시리얼 혹은 동시에 동작 가능
- OperationQueue: 동작할 아이템의 실행을 조절하는 큐

## ImmediateScheduler

~~~
.receive(on: ImmediateScheduler.shared)
~~~
> 현재 쓰레드에서 즉시 스케줄 > 메인 쓰레드에서 동작

~~~
.receive(on: DispatchQueue.global())
~~~

### ImmediateScheduler options
- SchedulerOptions 값을 옵션의 아규먼트로 넣을 수 있음.

### ImmediateScheduler pitfalls

- ImmediateScheduler 특징은 즉각적인 것.
- `schedule(after:)` 사용할 수 없음.
- 지연을 명시하는 SchedulerTimeType 이니셜라이저를 가지지 않음.

## RunLoop scheduler

- Timer > RunLoop에서 동작
- RunLoop.current린? 호출할 당시에 쓰레드와 연관되어 RunLoop

### A little comprehension challenge

- `subscribe(on: DispatchQueue.global())`을 통과하게 한다면? Thread 1, 왜냐? Timer를 사용하기에 사용자가 선택한 스케줄러와 관계없이 RunLoop

### Scheduling code execution with RunLoop

- 스케줄러를 사용하면 가능한 빨리 혹은 가까운 미래의 코드를 스케줄할 수 있음. 가까운 미래에 스케줄 하는 것은 ImmediateScheduler는 불가하지만 RunLoop는 가능.
- RunLoop의 SchedulerTimeType value 값은 Date.

~~~
RunLoop.current.schedule(after: .init(Date(timeIntervalSinceNow: 4.5)), 
						tolerance: .milliseconds(500)) {
							threadRecorder?.subscription?.cancel() 
						}
~~~
> threadRecorder 구독 취소하기, 4.5초 이후 실행 및 허용 오차 0.5

### RunLoop options

- ImmediateScheduler와 값이 SchedulerOptions을 받지 않음

### RunLoop pitfalls

- DispatchQueue에서 실행되는 코드에서 RunLoop.current를 사용하지 말 것.
- DispatchQueue 스레드는 일시적 일 수 있으므로 RunLoop을 사용하는 것이 거의 불가능합니다.

## DispatchQueue scheduler

- 직렬수행 동시수행 가능
- 직렬 > 작업을 순차로 수행
- 여러 작업을 병렬로 수행 > CPU 최대치로 사용
- serial queue: 공유된 자원을 겹치지 않고 사용가능함으로 락킹이 없음
- oncurrent queue: 동시에 오퍼레이터 수행하기 때문에 pure computation에 적합

### Queues and threads

- 모든 큐, 직렬, 동시 쓰레드에서의 코드 실행은 시스템에서 관리되어진다.
- RunLoop.current을 사용해서 스케줄을 해서는 안된다. DispatchQueue가 쓰레드를 관리하는 방식이기 떄문에
- 모든 디스패치 큐는 값은 쓰레드 풀을 공유한다.
- 시리얼 큐에서는 사용가능한 쓰레드를 사용.
- `subscribe(on:)`, `receive(on:)`을 사용할 때, 매번 동일할 거라고 간주하지 말 것.

### Using DispatchQueue as a scheduler

~~~
let serialQueue = DispatchQueue(label: "Serial queue")
// let sourceQueue = DispatchQueue.main
let sourceQueue = serialQueue

let source = PassthroughSubject<Void, Never>()
let subscription = sourceQueue.schedule(after: sourceQueue.now,
                                        interval: .seconds(1)) {
                                        source.send()
}

let setupPublisher = { recorder in
	source
	.recordThread(using: recorder) 
	.receive(on: serialQueue) 
	.recordThread(using: recorder) 
	.eraseToAnyPublisher()
}
~~~

### DispatchQueue options

- `.userInteractive.`: 사용자 인터페이스를 가능한 빨리 업데이트 하기를 원하는 경우
-  `.background`: 상대적으로 업데이트 압박이 적은 경우

## OperationQueue

~~~
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 1
let subscription = (1...10).publisher
	.receive(on: queue)
	.sink { value in
		print("Received \(value) on thread \(Thread.current.number)")
		}
~~~

### OperationQueue options

- 사용할 수 있는 옵션 없음.

### OperationQueue pitfalls

- OperationQueue 기본적으로 동시실행. 동시의 DispatchQueue처럼 동작.


