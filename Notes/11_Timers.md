# Chapter 11: Timers

## Using RunLoop

~~~
let runLoop = RunLoop.main
let subscription = runLoop.schedule( 
	after: runLoop.now,
	interval: .seconds(1),
	tolerance: .milliseconds(100)
){
 	print("Timer fired")
}
~~~
> cancellable 타이머 생성 

~~~
runLoop.schedule(after: .init(Date(timeIntervalSinceNow: 3.0)))
{
	cancellable.cancel()
}
~~~
> 타이머 스탑

* RunLoop는 타이머를 생성하는 좋은 방법은 아님

## Using the Timer class
~~~
let publisher = Timer.publish(every: 1.0, on: .main, in: .common)
~~~
> repeat 타이머 생성

* 타이머 퍼블리셔는 ConnectablePublisher 리턴. 
* `connect()` 메서드 불리기 전까지 방출 안함. (`autoConnect()` 사용가능 - 첫번째 서브스크라이버가 subscibe할 때 자동적으로 시작)

## Using DispatchQueue
* 디스패치큐는 타이머 인터페이스를 제공하지는 않지만, 대안적으로 타이머를 생성할 수 있다. 

##
