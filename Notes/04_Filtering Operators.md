# Chapter 4: Filtering Operators

## Getting started
ex) filter vs. tryFilter. The only difference between them is that the latter provides a throwing closure.
try 키워드의 차이점은 throwing 클로저는 공급한다는 것.

## Filtering basics
- filter 
- removeDuplicates: 중복제거 - 스트링을 포함한 Equatable을 따르는 모든 밸류에 대해 중복을 제거. Equatable을 따르고 있지않으면 removeDuplicates 오버라이드해서 동일한지 여부 판단해서 리턴.

## Compacting and ignoring
- compactMap: nil 거르기.
- ignoreOutput: 값의 방출이 끝났는데, 아무 값도 받지 못한 상황을 테스트할 때 사용.

## Finding values
- first(where:) : 
	- where 조건에 해당하는 첫번째 값. 
	- **lazy** 자기가 필요한 만큼만 받고 컴플리트.
	- 매칭 이 후에 값을 계속 방출할까? 놉. 서브스크립션 취소.
	- 즉, 값을 만나면 서브스크립션 취소되고, 컴플리트. 
- last(where:) : 
	- where 조건에 해당하는 가장 마지막 값. 
	- **greedy** 가장 마지막 값까지 기다림.
	- 그렇기 때문에 업스트림은 어느 지점에는 꼭 컴플리트되는 퍼블리셔여야함.
	
## Dropping values
- dropFirst : 처음 count 만큼 무시
- drop(while:) : while 문의 특정 조건에 해당하는 동안은 무시.
	- filter : 조건이 true 일 때 방출. 조건을 계속 체크.
	- drop(while:) : true 인 동안은 스킵. 조건에 맞아 한번 방출된 이 후에는 조건이 실행되지 않음.
- drop(untilOutputFrom: isReady) : 아규먼트 isReady(퍼블리셔) 값이 방출되면, 그 때부터의 값. isReady 퍼블리셔가 값을 방출하기 전까지 무시.

## Limiting values
- 값을 필터하고 스킵하는 조건이 스태틱 값일 수도, 클로저 조건일 수도, 또 다른 퍼블리셔일 수도 있음.
- prefix 패밀리, 전 챕터에서 배운 drop 패밀리들과 비슷. 하지만 조건에 도달하면 값을 받는 drop 패밀리와 다르게 조건에 도달할 때까지 값을 받음.
- prefix(_:) : dropFirst 반대, count 개수 까지만 받음. **lazy**
- prefix(while:) : drop(while:) 반대, true 인 동안만 받아요. false 면 바로 컴플리트. **greedy**
- prefix(untilOutputFrom: isReady) : isReady 값이 방출되면, 컴플리트. isReady 퍼블리셔가 값을 방출하면 종료.




	
	
