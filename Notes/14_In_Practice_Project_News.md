# Chapter 14: In Practice: Project "News"

## Getting started with the Hacker News API

* `enum Error`: API를 주고 받을 때 커스텀 에러
* `enum EndPoint`: 연결할 API의 URL
* `property maxStories`: 패치할 최신 스토리들의 최대 수
* `struct Story`: 스토리 데이터를 디코드할 구조체

## Getting a single story
* EndPoint를 사용해서 적절한 URL을 얻어 싱글 스토리 패치하기 
~~~
private let decoder = JSONDecoder()
private let apiQueue = DispatchQueue(label: "API", qos: .default, attributes: .concurrent)
  
func story(id: Int) -> AnyPublisher<Story, Error> {
  URLSession.shared.dataTaskPublisher(for: EndPoint.story(id).url)
    .receive(on: apiQueue)
    .map(\.data)
    .decode(type: Story.self, decoder: decoder)
    .catch { _ in Empty<Story, Error>() }
    .eraseToAnyPublisher()
}
~~~

* API 호출하기 
~~~
let api = API()
var subscriptions = [AnyCancellable]()

api.story(id: 1000) 
  .sink(receiveCompletion: { print($0) },
        receiveValue: { print($0) }) 
  .store(in: &subscriptions)
~~~

## Multiple stories via merging publishers
* 동시에 다수의 스토리 패치하기 
~~~
func mergedStories(ids storyIDs: [Int]) -> AnyPublisher<Story, Error> {
  let storyIDs = Array(storyIDs.prefix(maxStories))
  precondition(!storyIDs.isEmpty)
  let initialPublisher = story(id: storyIDs[0])
  let remainder = Array(storyIDs.dropFirst())
  return remainder.reduce(initialPublisher) { combined, id in
    return combined
            .merge(with: story(id: id))
            .eraseToAnyPublisher()
    }
 }
~~~

* API 호출하기
~~~
api.mergedStories(ids: [1000, 1001, 1002])
    .sink(receiveCompletion: { print($0) },
          receiveValue: { print($0) })
    .store(in: &subscriptions)
~~~

## Getting the latest stories
* 최신 스토리 패치하기
~~~
func stories() -> AnyPublisher<[Story], Error> {
  URLSession.shared.dataTaskPublisher(for: EndPoint.stories.url)
    .map(\.data)
    .decode(type: [Int].self, decoder: decoder)
    .mapError { error -> API.Error in
      switch error {
        case is URLError:
          return Error.addressUnreachable(EndPoint.stories.url)
        default:
          return Error.invalidResponse
        }
      }
      .filter { !$0.isEmpty }
      .flatMap { storyIDs in
        return self.mergedStories(ids: storyIDs)
      }
      .scan([]) { stories, story -> [Story] in
        return stories + [story]
      }
      .map { $0.sorted() }
      .eraseToAnyPublisher()
}
~~~

* API 호출하기
~~~
api.stories()
  .sink(receiveCompletion: { print($0) },
        receiveValue: { print($0) })
  .store(in: &subscriptions)
~~~


