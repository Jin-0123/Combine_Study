# Chapter 7: SequenceOperators

## Getting started

## Finding values

* `min()`
	* 최솟값
	* `.finished` 이벤트를 받을 때까지 기다렸다가 방출
	* **Comparable** 프로토콜을 따르지 않는 경우는 `min(by:)`를 사용해서 비교 클로저를 제공할 수 있음
	
* `max()`
	* 최댓값
	* `min()`과 마찬가지고 퍼블리셔가 끝날 때까지 기다림(greedy)
	
* `first()`
	* 첫번째 값을 찾으면 끝나는 것을 기다리지 않고 구독 취소(lazy)
	* `first(where:)`을 사용해서 조건을 걸 수 있음
	
* `last()`
	* 가장 마지막 값
	* 퍼블리셔 끝날 때 까지 기다림(greedy)

* `output(at:)`
	* 특정한 인덱스만 방출
	
* `output(in:)`
	* 특정한 인덱스인 여러 개의 값을 방출하고 싶을 때 사용
	* `output(in: 1...3)` range로 지정해서 방출
	* 여기서 방출된 값은 컬렉션이 아닌 개별적인 값
	
## Querying the publisher

* `count()`
	* 얼마나 많은 값을 방출했는지?
	* 퍼블리셔가 끝나야 방출(greedy)
	
* `contains()`
	* 해당 값이 포함되어있는지 여부
	* **true** 가 반환되면 구독 취소(lazy)
	* **Comparable** 프로토콜을 따르지 않는 경우 혹은 특정 조건에 매칭시키고 싶은 경우 `contains(where:)` 사용
	
* `allSatisfy()`
	* 해당 조건에 모두 만족하는지 여부
	* 퍼블리셔가 끝날 때까지 기다림
	* 조건에 만족하지 않으면 (false가 방출되면) 구독 취소
	
* `reduce()`
	* 더해진 값과 현재의 값을 클로저에서 제공, 하나의 결과를 리턴
	
	
	
	
	
