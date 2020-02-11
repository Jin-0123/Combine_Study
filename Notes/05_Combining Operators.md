# Chapter 5: Combining Operators

## Getting started

## Prepending

- 오리지널 퍼블리셔에서 어떤 값이 방출되기 전에 미리 값을 추가하기를 원할 때.

### prepend(Output...)
- prepend 가변 리스트를 받을 수 있음. 즉, 오리지널 퍼블리셔와 타입만 동일하다면 어떤 개수든지 값을 추가할 수 있음.

~~~
let publisher = [3, 4].publisher
  
  publisher
    .prepend(1, 2)
    .prepend(-1, 0)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
~~~
> 순서가 중요한데, 위 코드에서 방출되는 순서는 -1 > 0 > 1 > 2 > 3 > 4

### prepend(Sequence)
- Array OR Set
- 여기서 주의해야할 점은 Set 은 순서가 없기 때문에 방출 순서도 보장하지 않음. 만약 Set(1...2)면 1, 2로 방출될 수도 2, 1 순서로 방출될 수도 있음.
- stride는 Strideable 인데, 이 프로토콜은 Sequence 를 따름. 
~~~
stride(from: 6, to: 11, by: 2)
~~~
> 6부터 10까지 2 스탭 씩, 6, 8, 10

### prepend(Publisher)
- 퍼블리셔로 부터 방출되는 값도 prepend 할 수 있음.
- prepend 아규먼트가 되는 퍼블리셔가 컴플리트 되어야 함.

## Appending

### append(Output...)
- prepend와 비슷하게 가변리스트로 받을 수 있음. 오리지널 퍼블리셔가 끝나야 그 뒤에 append 된다. 컴플리트 되지 않으면 append 안되요.

### append(Sequence)

### append(Publisher)

## Advanced combining

### switchToLatest
- 가장 최신 퍼블리셔로 스위칭
- 예) 네트워킹하는 두 개의 버튼을 번갈아 눌렀을 때 유용하게 사용가능.

### merge(with:)
- 오리지널 퍼블리셔와 아규먼트 퍼블리셔가 방출되는 값을 시간 순으로 재조합.

### combineLatest
- 오리지널 퍼블리셔의 최신 값과 아규먼트의 최신 값을 조합.
- 오리지널 퍼블리셔와 아규먼트 퍼블리셔가 모두 적어도 한 번은 값을 방출해야 조합된 값이 push.

### zip
- 오리지널 퍼블리셔와 아규먼트 퍼블리셔가 방출하는 값의 인덱스를 맞춰 짝을 지어 조합.
- 어느 하나의 퍼블리셔에서 값이 방출되면 다른 퍼블리셔의 동일한 인덱스의 값이 방출되기를 기다림.
- 짝이 안지어지면 무시되요.

