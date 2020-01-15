 ## Hello Future
* subscriber에게 하나의 값을 방출하고 complete하는 publisher를 만들기 위해 **Just**를 사용하듯이 **Future**는 비동기적으로 하나의 결과값을 내고 complete 되도록 사용할 수 있다. 
* subscriptions을 저장하지 않으면, 현재 코드 스코프가 끝나면 취소된다. 
* 하나의 값을 방출하고 끝내거나 실패하거나.
* **Promise**는 Result<Output, Failure>를 파라미터로 가지 클로저를 뜻하는 alias

* 두번째 subscription을 수행하면, future는 다시 promiss를 수행하는 것이 아니라 결과를 공유하여 같은 값을 받게된다.
* Future는 만들자마자 실행되기 때문에 일반 publisher처럼 subsciber가 필요하진 않는다.

## Hello Subject
* subject는 non-Combine 코드와 Combine 코드의 subscribers 에게 값을 보낼 수 있는 중간자 역할을 한다.
* **Passthrough** subject는 요구에 따라 새 값을 publish 하도록 한다. 다른 publisher 와 동일하게 방출할 value와 type을 명시해야하며, subsciber 또한 그 인풋타입과 실패 타입을 매칭시켜야한다.
* **CurrentValueSubject** imperative 코드에서 publisher의 현재 값을 알고싶을 때 사용.
* subscription 이 값으로서 저장하는 대신 여러 개의 AnyCancellable의 컬렉션 안에 subscription으로 저장할 수 있다. 
* 새로운 값을 셋팅할 때, send() / 새 값을 할당하려고 할 때, value 프러퍼티
* `print()` 오퍼레이터를 사용해서 모든 publishing event 로그르 콘솔에서 볼 수 있음

## Dynamically adjusting demand
* `max([num])` subsciber 의 수요를 조절할 수 있다.

## Type erasure
* `.eraseToAnyPublisher()` Publisher 의 타입을 지우고, AnyPublisher 타입으로 만들어줌.
* AnyPublisher는 send()가 없어 새 값을 넣을 수 없음.

