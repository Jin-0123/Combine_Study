# Chapter 16: Error Handling

## Getting started

## Never

* `Never` 타입은 퍼블리셔가 절대 실패할 수 없다는 것을 의미함.
* 퍼블리셔에 대해 강력하게 보장.
* `Never` 타입을 가진 퍼블리셔는 퍼블리셔 값을 소비하는데에만 집중하게 함.
* `Just`의 `Failure` 타입이 `Never`

## setFailureType

~~~
.setFailureType(to: MyError.self)
~~~
* `.setFailureType(to:)`를 사용해서 퍼블리셔의 에러타입을 변경할 수 있다.

## assign(to: on:)

~~~
Just("Shai") 
	.handleEvents(
		receiveCompletion: { _ in print("2", person.name) } 
	)
	// .assign을 이용해서 이름 set
	.assign(to: \.name, on: person)
	.store(in: &subscriptions)
~~~

* 위의 코드에서 `.setFailureType(to: Error.self)`을 사용하면 다음과 같은 에러 발생.
> referencing instance method 'assign(to: on:)' on 'Publisher' requires the types 'Error' and 'Never' be equivalent

## assertNoFailure

* `assertNoFailure` 연산자는 개발할 때 사용하는 유용한 연산자. 에러발생 시, `fatalError`로 크래시를 발생시켜 수정에 도움을 줌.

## Dealing with failure

### try* operators

~~~
.tryMap { value -> Int in
	let length = value.count
	guard length >= 5 else { throw NameError.tooShort(value) }
	return value.count 
}
~~~
* `.tryMap`을 이용해서 에러 던지기

### Mapping errors

* `.tryMap`을 사용하면 컴플리션의 Error 타입을 Swift.Error타입으로 바꾼다.
* 이때 `.mapError`를 사용하면 내가 원하는 타입의 에러로 바꿀 수 있다.

~~~
Just("Hello")
	.setFailureType(to: NameError.self)
	.try Map { throw NameError.tooShort($0) }
	.mapError { $0 as? NameError ?? .unknown }
	.sink(receiveCompletion: { completion in
		switch completion {
		case .finished:
			print("Done!")
		case .failure(.tooShort(let name)):
			print("\(name) is too short!")
		case .failure(.unknown):
			  print("An unknown name error occurred")
		}
	}, receiveValue: { print("Got value \($0)") })
	.store(in: &subscriptions) }
~~~

## Designing your fallible APIs

* Error를 왜 감싸야하는지?
  * DadJokes.Error로 실패하는 것을 보장하는 것은 에러를 처리하기에 유용. 어떤 상황인지 정확히 알 수 있기때문.
  * API 상세나 유출되지 않는다. 단지 정의된 에러만 받게되기 때문.

~~~
func getJoke(id: String) -> AnyPublisher<Joke, Error> {
	let url = URL(string: "https://icanhazdadjoke.com/j/\(id)")!
	var request = URLRequest(url: url)
	request.allHTTPHeaderFields = ["Accept": "application/json"]

	return URLSession.shared
	.dataTaskPublisher(for: request)
	.map(\.data)
	.decode(type: Joke.self, decoder: JSONDecoder())
	.mapError { error -> DadJokes.Error in
		switch error {
		case is URLError:
			return .network
		case is DecodingError:
			return .parsing
		default:
			return .unknown
		}
	}
	.eraseToAnyPublisher()
}
~~~

* API 호출 전에 검증
~~~
// 숫자와 문자가 함께 있어야만 한다면? 숫자만 보냈을 때 에러 발생시키기
guard id.rangeOfCharacter(from: .letters) != nil else {
	return Fail<Joke, Error>(error: .jokeDoesntExist(id: id))
	.eraseToAnyPublisher()
}
~~~

## Catching and retrying

* `.retry` 를 사용해서 재시도 횟수를 정의하고, retry를 할 수 있음.
* 에러 발생 시, 대체 이미지 설정
~~~
.replaceError(with: UIImage(named: "na.jpg")!)
~~~

~~~
example(of: "Catching and retrying") {
	photoService
		.fetchPhoto(quality: .high)
		.handleEvents(receiveSubscription: { 
			_ in print("Trying ...")
		}, receiveCompletion: {
			guard case .failure(let error) = $0 else { return }
			print("Got error: \(error)")
		})
		// 3번 재시도
		.retry(3)
		// high 퀄리티 패치 실패 시, low로 가져올 수 있도록 
		.catch { error -> PhotoService.Publisher in
			print("Failed fetching high quality, falling back to low quality")
			return photoService.fetchPhoto(quality: .low)
		}
		// 에러나면 대체되는 이미지
		.replaceError(with: UIImage(named: "na.jpg")!)
		.sink(receiveCompletion: { print("\($0)") }, receiveValue: { image in
			image
			print("Got image: \(image)")
		})
		.store(in: &subscriptions)
}
~~~
