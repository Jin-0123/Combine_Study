# Chapter 19: Testing

## Getting started

## Testing Combine operators

* Given-When-Then 
  * Given: 조건
  * When: 실행될 액션
  * Then: 기대결과 
  
## Testing collect()

~~~
func test_collect() {
	// Given
	let values = [0, 1, 2]
	let publisher = values.publisher

	// When
	publisher
		.collect()
		.sink(receiveValue: {
			// Then
			XCTAssert($0 == values, "Result was expected to be \(values) but was \($0)")
			//XCTAssert(
			//	$0 == values + [1],
			//	"Result was expected to be \(values + [1]) but was \($0)"
			//)
		})
		.store(in: &subscriptions)
}
~~~
  
## Testing flatMap(maxPublishers:)

~~~
func test_flatMapWithMax2Publishers() {
	// Given
	let intSubject1 = PassthroughSubject<Int, Never>()
	let intSubject2 = PassthroughSubject<Int, Never>()
	let intSubject3 = PassthroughSubject<Int, Never>()

	let publisher = CurrentValueSubject<PassthroughSubject<Int, Never>, Never>(intSubject1)

	let expected = [1, 2, 4]
	var results = [Int]()

	publisher
		.flatMap(maxPublishers: .max(2)) { $0 }
		.sink(receiveValue: { results.append($0) })
		.store(in: &subscriptions)

	// When

	intSubject1.send(1)

	publisher.send(intSubject2)
	intSubject2.send(2)

	publisher.send(intSubject3)
	intSubject3.send(3)
	intSubject2.send(4)

	publisher.send(completion: .finished)

	// Then
	XCTAssert(
		results == expected,
		"Results expected to be \(expected) but were \(results)"
	)
}
~~~

## Testing publish(every:on:in:)
  
~~~
func test_timerPublish() {
	// Given
	func normalized(_ ti: TimeInterval) -> TimeInterval {
		// 시간 간격 정규화
		return Double(round(ti * 10) / 10)
	}

	let now = Date().timeIntervalSinceReferenceDate

	let expectation = self.expectation(description: #function) 

	let expected = [0.5, 1, 1.5]
	var results = [TimeInterval]()

	let publisher = Timer
		.publish(every: 0.5, on: .main, in: .common) 
		.autoconnect()
		// 처음 3개의 값만 가져온다.
		.prefix(3)

	// When
	publisher
		.sink(receiveCompletion: { _ in expectation.fulfill() }, receiveValue: {
			results.append(normalized($0.timeIntervalSinceReferenceDate - now))
		})
		.store(in: &subscriptions)

	// Then
	// 최대 2초 동안 기다림
	waitForExpectations(timeout: 2, handler: nil)

	XCTAssert(
		results == expected,
		"Results expected to be \(expected) but were \(results)"
	)
}
~~~

## Testing shareReplay(capacity:)

~~~
func test_shareReplay() {
	// Given

	let subject = PassthroughSubject<Int, Never>() 
	let publisher = subject.shareReplay(capacity: 2) 

	let expected = [0, 1, 2, 1, 2, 3, 3]
	var results = [Int]()

	// When
	publisher
		.sink(receiveValue: { results.append($0) }) 
		.store(in: &subscriptions)

	subject.send(0) 
	subject.send(1) 
	subject.send(2)

	publisher
		.sink(receiveValue: { results.append($0) }) 
		.store(in: &subscriptions)

	subject.send(3)
	XCTAssert(
		results == expected,
		"Results expected to be \(expected) but were \(results)"
	)
}
~~~

## Testing production code

~~~
class ColorCalcTests: XCTestCase {
    var viewModel: CalculatorViewModel!
    var subscriptions = Set<AnyCancellable>()
  
    override func setUp() {
        viewModel = CalculatorViewModel()
    }
  
    override func tearDown() {
        subscriptions = []
    }
}
~~~

### Issue 1: Incorrect name displayed

~~~
func test_correctNameReceived() {
	// Given
	let expected = "rwGreen 66%"
	var result = ""
	
	viewModel.$name
		.sink(receiveValue: { result = $0 })
		.store(in: &subscriptions)
		
	// When
	viewModel.hexText = "006636AA"
	
	// Then
	XCTAssert(
		result == expected,
		"Name expected to be \(expected) but was \(result)"
	)
}
~~~

* 기존 코드 아래와 같이 수정하여 테스트 통과하도록 변경

~~~
.map {
	if let name = ColorName(hex: $0) {
		return "\(name) \(Color.opacityString(forHex: $0))" 
	} else {
		return "------------" 
	}
}
~~~


### Issue 2: Tapping backspace deletes two characters

~~~
func test_processBackspaceDeletesLastCharacter() { 
	// Given
	let expected = "#0080F"
	var result = ""
	
	viewModel.$hexText
		.dropFirst()
		.sink(receiveValue: { result = $0 }) 
		.store(in: &subscriptions)
		
	// When
	viewModel.process(CalculatorViewModel.Constant.backspace)
	
	// Then
	XCTAssert(
		result == expected,
		"Hex was expected to be \(expected) but was \(result)"
	)
}
~~~

> `hexText.removeLast(2)` -> `hexText.removeLast()`

### Issue 3: Incorrect background color

~~~
func test_correctColorReceived() {
	// Given
	let expected = Color(hex: ColorName.rwGreen.rawValue)! 
	var result: Color = .clear
	
	viewModel.$color
		.sink(receiveValue: { result = $0 }) 
		.store(in: &subscriptions)
		
	// When
	viewModel.hexText = ColorName.rwGreen.rawValue
	
	// Then
	XCTAssert(
		result == expected,
		"Color expected to be \(expected) but was \(result)"
	)
}
~~~

~~~
func test_processBackspaceReceivesCorrectColor() { 
	// Given
	let expected = Color.white
	var result = Color.clear
	
	viewModel.$color
		.sink(receiveValue: { result = $0 }) 
		.store(in: &subscriptions)
		
	// When
	viewModel.process(CalculatorViewModel.Constant.backspace)
	
	// Then
	XCTAssert(
		result == expected,
		"Hex was expected to be \(expected) but was \(result)"
	)
}
~~~

~~~
.map { $0 != nil ? Color(values: $0!) : .white }
~~~
> `.red` -> `.white`로 수정

### Testing for bad input

* 잘못된 hexText 값이 들어왔을 때 대응
~~~
func test_whiteColorReceivedForBadData() { 
	// Given
	let expected = Color.white
	var result = Color.clear
	
	viewModel.$color
		.sink(receiveValue: { result = $0 }) 
		.store(in: &subscriptions)
		
	// When
	viewModel.hexText = "abc"
	
	// Then
	XCTAssert(
		result == expected,
		"Color expected to be \(expected) but was \(result)"
	)
}
~~~
