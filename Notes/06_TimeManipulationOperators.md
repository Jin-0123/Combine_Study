# Chapter 6: Time Manipulation Operators

## Shifting time
`delay(for:tolerance:scheduler:options)`
* 스케줄러에게 요청한 시간동안 지연된 후, 값이 방출
* Timer: 퍼블리셔 클래스의 한 부분이고, Connectable임. 즉, 값을 방출하기 전에 connected 되어야 함. `autoconnect`를 사용해서 연결가능

## Collecting values
`collect()`
* 버퍼처럼 그룹별로 방출하고 싶을 때 
* 일정시간동안 값의 평균을 알아볼 때 유용.
~~~
let collectedPublisher = sourcePublisher
	.collect(.byTime(DispatchQueue.main, .seconds(collectTimeStride)))
	// 방출되는 값을 배열처럼 보고싶다면 flatMap을 사용.
	.flatMap { dates in dates.publisher }
~~~

## Collecting values (part 2)
`collect(_:options:)`
* 일정 시간 뿐만아니라 최대 그룹지을 값의 수를 명시할 수 있음.

~~~
let collectedPublisher2 = sourcePublisher
	.collect(.byTimeOrCount(DispatchQueue.main, .seconds(collectTimeStride), collectMaxCount))
	.flatMap { dates in dates.publisher }
~~~
> collectTimeStride 시간동안의 그룹지어지든지 아니면 collectMaxCount만큼 모이면 그룹지어지든지.

## Holding off on events
* 텍스트 필드에서 url 자동완성을 처리할 때 도움을 줄 두 오퍼레이터 : debounce and throttle

### Debounce
`debounce(for:scheduler:)`
* 일정 시간을 주기로 가장 마지막 값을 방출 

~~~
let debounced = subject
	// 1초마다 마지막 값을 방출.
	.debounce(for: .seconds(1.0), scheduler: DispatchQueue.main)
	// 일관성을 보장해주기 위해 share() 사용, 모든 구독자에게 같은 값을 방출.
	.share()
~~~

* feed(with:): ex. `subject.feed(with: data)` The feed(with:) method takes a data set and sends data to the given subject at pre-defined time intervals.
* **Summary:** A function that can feed delayed values to a subject for testing and simulation purposes
* 서브젝트에 데이터셋을 전달해서 테스트 할 수 있게 함. 

* **퍼블리셔 컴플리트를 위해 주의해야할 사항은 설정된 debounce 타임이 경과하기 전에, 마지막 값이 방출되어 종료된다면 debounce의 마지막값은 절대 볼 수 없음.**

### Throttle
`throttle(for:scheduler:latest:)`

~~~
let throttled = subject
	// throttleDelay을 주기로 첫번째의 값(latest가 false이기때문)을 방출
	.throttle(for: .seconds(throttleDelay), scheduler: DispatchQueue.main, latest: false)
	.share()
~~~

* Debounce VS Throttle
* debounce: 일정 주기를 기다려서 값을 받고, 특정 인터벌 이 후, 가장 최신 값을 방출. 그래서 기다린 시간만큼 지연이 있음.
* throttle: 특정 인터벌을 기다리고, 가장 첫번째 혹은 가장 최신의 값을 방출.  

## Timing out
`timeout`
* 완료된 퍼블리셔를 방출하거나 에러를 방출하거나.

~~~
// 5초 후, 완료된 퍼블리셔 방출
let timedOutSubject = subject.timeout(.seconds(5), scheduler: DispatchQueue.main)

// 5초 후, 커스텀 에러를 방출
let timedOutSubject2 = subject.timeout(.seconds(5), scheduler: DispatchQueue.main, customError: { .timedOut })
~~~

## Measuring time
`measureInterval(using:)`
* 연속된 두 값 사이의 경과된 시간을 알고 싶을 때.
~~~
let measureSubject = subject.measureInterval(using: DispatchQueue.main)
~~~
* 나노세컨즈로 방출, 일반적인 초로 보고싶으면 환산해야함.
* 다른 스케줄러를 사용할 수도 있음. `DispatchQueue.main` 대신 `RunLoop.main` 선택은 개인의 취향차이. 일반적으로는 `DispatchQueue`를 사용.
