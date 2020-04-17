# Chapter 9: Networking

## URLSession extensions

* 검색을 위해 데이터 전송
* 다운로드
* 업로드
* 스트림 
* 웹소켓

~~~
guard let url = URL(string: "https://mysite.com/mydata.json") else {
	return
}

let subscription = URLSession.shared
.dataTaskPublisher(for: url) .sink(receiveCompletion: { completion in
	if case .failure(let err) = completion {
			print("Retrieving data failed with error \(err)")
		}
	}, receiveValue: { data, response in
		print("Retrieved data of size \(data.count), response = \ (response)")
})
~~~
> URLRequest, URL을 받아서 API 호출을 컴바인으로 작성하기

## Codable support
~~~
let subscription = URLSession.shared 
	.dataTaskPublisher(for: url) 
	.tryMap { data, _ in
		try JSONDecoder().decode(MyType.self, from: data) 
	}
	.sink(receiveCompletion: { completion in if case .failure(let err) = completion {
		print("Retrieving data failed with error \(err)")
    }
  }, receiveValue: { object in
    print("Retrieved object \(object)")
})
~~~

* tryMap 오퍼레이터 부분을 아래 라인처럼 변경 가능
~~~
.map(\.data)
.decode(type: MyType.self, decoder: JSONDecoder())
~~~

## Publishing network data to multiple subscribers
* subsciber가 다수라면, 매번 요청하기 때문에 `share()` 사용.
* `multicast()` 오퍼레이터 사용하면 퍼블리셔를 ConnectablePublisher로 생성.
* `connect()` 가 호출된 다음, 방출.

