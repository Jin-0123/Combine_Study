# Chapter 20: In Practice: Building a Complete App_1 
 
* Combine과 함께 CoreData 사용하기

## Getting started

	1. ChuckNorrisJokes: 메인 타겟, UI 코드가 포함되어 있음.
	2. ChuckNorrisJokesModel: 모델과 서비스들을 정의, 타겟과 분리는 메인 타겟과 테스팅 타겟의 접근을 관리하는데에 좋음.
	3. ChuckNorrisJokesTests: 유닛 테스트 타겟
	
## Setting goals

	1. 조크카드 양방향 스와이프 시 인디케이터 보여주기, 호감 조크인지 비호감 조크인지 보여주기 위해
	2. 호감 조크를 저장, 나중에 읽기위해
	3. 스와이프 방향에 따라 조크카드 배경색 설정하기
	4. 현재 조크 좋아하기 혹은 싫어하기 후에 새로운 조크 패치
	5. 새로운 조크 패치 시, 인디케이터 보여주기
	6. 새로운 조크 패치가 잘못되었을 때 안내 노출
	7. 영어와 스페인어 버전의 토글
	8. 저장된 조크 리스트에 올리기
	9. 저장된 리스트 영어, 스페인어 토글
	10. 저장된 조크 삭제

## Implementing JokesViewModel

### Implementing state

~~~
@Published public var fetching: Bool = false
@Published public var joke: Joke = Joke.starter
@Published public var backgroundColor = Color("Gray")
@Published public var decisionState: DecisionState = .undecided
@Published public var showTranslation = false
~~~
* @Published 키워드 사용하면 퍼블리셔와 엮임. $ 프리픽스 사용해서 접근 가능.

### Implementing services

* `JokesService` **chucknorris.io** 로 부터 랜덤 조크 패치하는데 사용.
* 나중에 유닛테스트의 목으로 사용하기 위해 퍼블리셔의 필수 프로토콜을 정의한다.
~~~
public protocol JokeServiceDataPublisher {
	func publisher() -> AnyPublisher<Data, URLError>
}
~~~

* translation service에도 프로토콜 정의
~~~
public protocol TranslationServiceDataPublisher {
	func publisher(for joke: Joke, to languageCode: String) -> AnyPublisher<Data, URLError>
}
~~~
> 필수 메서드는 조크 인스턴스와 언어코드를 받고 조크를 번역한다.

* `TranslationService`가 `TranslationServiceDataPublisher` 따르도록 한다.
~~~
public struct TranslationService: TranslationServiceDataPublisher {

	public func publisher(for joke: Joke, to languageCode: String) -> AnyPublisher<Data, URLError> {
		URLSession.shared
			.dataTaskPublisher(for: url(for: joke, languageCode: languageCode))
			.map(\.data)
			.eraseToAnyPublisher()
	}
	
	/// ...
}
~~~
> 1. dataTaskPublisher를 조크와 언어코드로 구성된 url과 함께 생성
> 2. map 을 사용해서 퍼블리셔의 응답을 data 프러퍼티로!
> 3. 타입 지우기

* `JokesService`가 `JokeServiceDataPublisher` 따르도록 한다.
~~~
public struct JokesService: JokeServiceDataPublisher {
    public func publisher() -> AnyPublisher<Data, URLError> {
        URLSession.shared
            .dataTaskPublisher(for: url)
            .map(\.data)
            .eraseToAnyPublisher()
    }
	
	/// ...
}
~~~

* 이대로 테스트 타겟 빌드하려고 하면 2개 빌드에러 발생. 테스트 타겟도 수정 필요.
* MockJokesService.swift 파일 수정 
~~~
struct MockJokesService: JokeServiceDataPublisher {
	func publisher() -> AnyPublisher<Data, URLError> {
		// Data 방출,  실패 시 URLError 방출하는 mock 퍼블리셔 생성
		let publisher = CurrentValueSubject<Data, URLError>(data)
		// 에러 발생 시, 퍼블리셔에 에러 전달
		if let error = error { 
			publisher.send(completion: .failure(error)) 
		}
        return publisher.eraseToAnyPublisher()
    }
	
	/// ...
}
~~~

### Finish implementing JokesViewModel

* 2개의 프러퍼티 추가
~~~
private let jokesService: JokeServiceDataPublisher
private let translationService: TranslationServiceDataPublisher
~~~

* initalizer 업데이트
* `$joke` 퍼블리셔 subscription 추가 
~~~
public init(jokesService: JokeServiceDataPublisher = JokesService(), translationService: TranslationServiceDataPublisher = TranslationService()) {
	self.jokesService = jokesService
	self.translationService = translationService
	
	$joke
        .map { _ in false }
        .assign(to: \.fetching, on: self)
        .store(in: &subscriptions)
}
~~~

#### Fetching jokes

~~~
public func fetchJoke() {
	fetching = true
	// 이 전에 추가했던 구독을 취소하기 위해서 빈 컬렉션으로 셋팅
	jokeSubscriptions = []
	// joke service publisher 구독 시작
	jokesService.publisher()
		// 실패 시 한번 재시도
		.retry(1)
		// 퍼블리셔로 부터 받은 데이터 전달(디코드 오퍼레이터이용)
		.decode(type: Joke.self, decoder: Self.decoder)
		// 에러 메시지 노출 추 Joke 인스턴스 에러로 대체
		.replaceError(with: Joke.error)
		// 메인 큐에서 결과 받기
		.receive(on: DispatchQueue.main)
		// 퍼블리셔 통해 joke 전달, 패치된 번역 보여줄 수 있게
		.handleEvents(receiveOutput: { [unowned self] in
			self.joke = $0 
		})
		// 에러나면 보여주지 않도록 필터
		.filter { $0 != Joke.error } 
		// 번역패치 트리거 위해 플랫맵 사용, 번역 전 조크 퍼블리셔를 번역된 조크 새 퍼블리시로 변환.
		.flatMap { [unowned self] joke in
			self.fetchTranslation(for: joke, to: "es") 
		}
		// 메인큐에서 결과 받기
		.receive(on: DispatchQueue.main) 
		// 받은 조크 퍼블리셔에 assign
		.assign(to: \.joke, on: self)
		.store(in: &jokeSubscriptions)
	}
~~~

#### Fetching translations

* `fetchJoke()`와 유사
~~~
func fetchTranslation(for joke: Joke, to languageCode: String)
    -> AnyPublisher<Joke, Never> {
	// 이미 번역되어 있다면 republish
	guard joke.languageCode != languageCode else {
		return Just(joke).eraseToAnyPublisher()
	}
	// translationService 퍼블리셔 구독시작 
	return translationService.publisher(for: joke, to: languageCode)
		.retry(1)
		.decode(type: TranslationResponse.self, decoder: Self.decoder)
		.compactMap { $0.translations.first }
		// translationService 통해 리턴된 값으로 새로운 joke 생성
		.map {
			Joke(id: joke.id, value: joke.value, categories: joke.categories, languageCode: languageCode, translationLanguageCode: languageCode, translatedValue: $0)
		}
		.replaceError(with: Joke.error)
		.eraseToAnyPublisher()
  }
~~~

#### Changing the background color

* 카드 포지션에 따라 배경색을 업데이트해보자.
~~~
public func updateBackgroundColorForTranslation(_ translation: Double) {
	switch translation {
	case ...(-0.5):
		backgroundColor = Color("Red")
	case 0.5...:
		backgroundColor = Color("Green")
	default:
		backgroundColor = Color("Gray")
	}
}
~~~

* 좋은지 싫은지에 결정하기 위해 joke 카드뷰의 포지션을 사용해보자.
~~~
public func updateDecisionStateForTranslation(_ translation: Double, andPredictedEndLocationX x: CGFloat, inBounds bounds: CGRect) {
	switch (translation, x) {
	case (...(-0.6), ..<0):
		decisionState = .disliked
	case (0.6..., bounds.width...):
		decisionState = .liked
	default:
		decisionState = .undecided
	} 
}
~~~

#### Preparing for the next joke

~~~
public func reset() {
	backgroundColor = Color("Gray")
}
~~~

#### Making the view model observable

* `JokesViewModel` `ObservableObject` 따르도록 변경
* 모델 변경되는 것 자동으로 변경되도록!
~~~
public final class JokesViewModel: ObservableObject {
~~~

## Wiring JokesViewModel up to the UI

* JokeCardView.swift 파일 수정
	* 모델 가져오기
	~~~
	@ObservedObject var viewModel: JokesViewModel
	~~~

	* Previews 수정으로 빌드에러 제거
	~~~
	struct JokeCardView_Previews: PreviewProvider { 
		static var previews: some View {
			JokeCardView(viewModel: JokesViewModel()) 
				.previewLayout(.sizeThatFits)
		} 
	}
	~~~
	
* JokeView.swift 파일 수정
	* 모델 가져오기
	~~~
	@ObservedObject private var viewModel = JokesViewModel()
	~~~
	
	* `JokeCardView()` 이닛 변경 
	~~~
	JokeCardView(viewModel: viewModel)
	~~~
	
* JokeCardView.swift 파일 수정 
~~~
Text(viewModel.showTranslation ? viewModel.joke.translatedValue : viewModel.joke.value)
~~~

## Displaying a required link

* 번역을 표시할 때만 버튼이 보이도록, 버튼 투명도 변경
~~~
.opacity(viewModel.showTranslation ? 1 : 0)
~~~

## Toggling the original and translated joke

* `showTranslation` 토글
~~~
action: { self.viewModel.showTranslation.toggle() }
~~~

## Setting the joke card’s background color

* 배경색 변경 > JokeView.swift 파일 변경
~~~
.background(viewModel.backgroundColor)
~~~

## Indicating if a joke was liked or disliked

* HUDView.swift에 정의 되어있는 이미지에 따라 투명도 변경 > JokeView.swift 파일 변경
~~~
HUDView(imageType: .thumbDown)
	.opacity(viewModel.decisionState == .disliked ? hudOpacity : 0)
	.animation(.easeInOut)
      
HUDView(imageType: .rofl)
	.opacity(viewModel.decisionState == .liked ? hudOpacity : 0)        
	.animation(.easeInOut)
~~~

## Handling decision state changes

~~~
private func updateDecisionStateForChange(_ change: DragGesture.Value) {
	viewModel.updateDecisionStateForTranslation(translation, andPredictedEndLocationX: change.predictedEndLocation.x, inBounds: bounds) 
}
~~~

~~~
private func updateBackgroundColor() {
	viewModel.updateBackgroundColorForTranslation(translation)
}
~~~

## Handling when the user lifts their finger

~~~
private func handle(_ change: DragGesture.Value) { 
	// 현재 decisionState 값 로컬 카피 
	let decisionState = viewModel.decisionState
	switch decisionState { 
	// 결정되지 않은 경우
	case .undecided:
		cardTranslation = .zero
		self.viewModel.reset() 
	default:
		// 새로운 오프셋 적용, 변경된 번역 적용. 일시적으로 카드 숨기기.
		let translation = change.translation
		let offset = (decisionState == .liked ? 2 : -2) * bounds.width
		cardTranslation = CGSize(width: translation.width + offset,
	    showJokeView = false
		// 숨기고, 원래 포지션으로 카드 위치 변경 그리고 새로운 조크 패치 후 카드 보이기
		reset()
	}
}
~~~

* reset() 메서드에 추가
~~~
self.viewModel.reset() 
self.viewModel.fetchJoke()
~~~
