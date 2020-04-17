# Chapter 10: Debugging

## Printing events

* `print(_:to:)`
	* 퍼블리셔가 어떻게 수행되고 있는지 확실하지 않을 때 사용하면, 어떤 일이 일어나고 있는지 많은 정보를 보여줌.
	* subscription을 받을 때 프린트, 업스트림 퍼블리셔에 대한 디스크립션 보여줌
	* 얼마나 많은 아이템이 요청되었는지 프린트
	* 방출되는 모든 값을 프린트
	* 완료 이벤트 프린트

* TextOutputStream 객체를 받을 수 있는 추가적인 파라미터가 있다. 현재 날짜, 시간 같은 추가적인 로그를 더할 수 있음.

## Acting on events — performing side effects

~~~
.handleEvents(receiveSubscription: { _ in 
	print("Network request will start")
}, receiveOutput: { _ in
	print("Network request data received")
}, receiveCancel: {
	print("Network request cancelled")
})
~~~
> 퍼블리셔의 라이프 사이클에서 모든 이벤트와 각 스텝에서의 액션을 가로채서 확인

## Using the debugger as a last resort

* `breakpointOnError()` 특정 조건에서 브레이크
