# Chapter 3: Transforming Operators

## Getting started

### Operators are publishers
* 오퍼레이터는 퍼블리셔로, 실제로 각각의 컴바인 오퍼레이터는 퍼블리셔를 반환한다.
* 퍼블리셔는 업스트림 값을 받고, 데이터르 조작한 다음 데이터를 다운스트림으로 보낸다.
* 오퍼레이터의 목적이 에러핸들링이 아니라면 업스트림 퍼블리셔로부터 에러를 전달받은 경우, 다운스트림으로 에러를 전달한다.

## Collecting values
* 퍼블리셔는 개별적인 값을 방출하거나 값의 컬렉션을 방출할 수 있다.

~~~
collect()

// 배열로 묶을 값의 개수르 정의할 수 있음.
collect(2)
~~~
> 개별적인 값의 스트림을 버퍼하다가 그 값들의 배열을 다운스트림으로 내보낸다.

## Mapping values
~~~
map(_:)
~~~

### Map key paths
~~~
map<T>(_:)
map<T0, T1>(_:_:)
map<T0, T1, T2>(_:_:_:)

// 예제) Map into the x and y properties of Coordinate using their key paths.
example(of: "map key paths") {
  // 1
  let publisher = PassthroughSubject<Coordinate, Never>()
  
  // 2
  publisher
    // 3
    .map(\.x, \.y)
    .sink(receiveValue: { x, y in
      // 4
      print(
        "The coordinate at (\(x), \(y)) is in quadrant",
        quadrantOf(x: x, y: y)
      )
    })
    .store(in: &subscriptions)
  
  // 5
  publisher.send(Coordinate(x: 10, y: -8))
  publisher.send(Coordinate(x: 0, y: 5))
}
~~~

~~~
tryMap(_:)
~~~
> map 오퍼레이터 + try 구문

## Flattening publishers
~~~
flatMap(maxPublishers:_:)
~~~
