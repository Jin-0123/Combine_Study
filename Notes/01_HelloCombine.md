## 시작

_In Apple's own words:_
> "The Combine framework provides a declarative approach for how your app processes events. Rather than potentially implementing multiple delegate callbacks or completion handler closures, you can create a single processing chain for a given event source. Each part of the chain is a Combine operator that performs a distinct action on the elements received from the previous step."

Combine 프레임워크는 어떻게 앱이 이벤트를 실행하는지에 대한 서술문의 접근을 제공한다. 여러 개의 델리케이트 콜백이나 컴플리션 핸들러 클로저의 잠재적인 구현보다도 이벤트르 위한 하나의 이벤트 프로세스 체인을 만들게 된다. 체인의 각 파트는 Combine 연산자이고, 그것은 이전 스텝으로부터 받은 요소에서 별개의 액션으로 실행된다.

## Asynchronous programming

간단한 예로, 싱글 쓰레드 언어의 경우 한 프로그램은 각 라인이 순차적으로 실행된다. 

_in pseudocode:_
~~~
begin
  var name = "Tom"
  print(name)
  name += " Harding"
  print(name)
end
~~~

동기코드는 이해하기 쉽고, 데이터 상태에 대한 설득을 쉽게 하도록 만든다. 싱글 쓰레드로 실행될 때, 데이터의 현재 상태가 무엇인지 항상 확신할 수 있다.

멀티 쓰레드의 경우, 한 코드가 다른 코어에서 동시에 실행될 때 코드의 어떤 위치에서 공유되어진 첫번째 상태가 변경되어질지 예측하는 것은 어렵다.
~~~
--- Thread 1 ---
begin
  var name = "Tom"
  print(name)

--- Thread 2 ---
name = "Billy Bob"

--- Thread 1 ---
  name += " Harding"
  print(name)
end
~~~
* 오리지널 코드에서 다른 CPU 코어에 정확히 같은 시간에 실행될 때
* `name += " Harding"` 실행되기 직전에 오리지널 코드 값인 Tom 아닌 Billy Bob 실행될 때

이 코드가 실행될 때, 정확히 어떤 일이 일어날지는 시스템 로드에 달려있다. 그리고 프로그램을 실행시킬 때마다 매번 다른 결과를 보게될 것이다.

## Foundation and UIKit/AppKit
애플은 수 년에 걸쳐 비동기 프로그램을 개선해왔다. 애플은 각기 다른 시스템 레벨에서 비동기 코드를 생성할 수 있도록 몇몇의 매커니즘을 제공했다. 

* **NotificationCenter**: 관심있는 이벤트가 발생했을 때, 언제든지 코드를 실행시킨다. 예를 들면, 디바이스 방향이 바뀐다든지 화면에서 키보드가 보이거나 숨겨질 때.
* **The delegate pattern**: 한 오브젝트가 다른 오브젝트를 대신하여 일을 하거나 조정을 위해 액션을 취해야할 때 정의한다. 예를 들면, 리모트 알림이 도착했을 때, 어떤 일이 일어나야하는지 정의한다. 그러나 이 코드 실행되는지, 얼마나 많이 이 코드가 실행되는지 알 수 없다.
* **Grand Central Dispatch** and **Operations**: 추상적인 작업의 일부 실행을 돕는다. 연속된 큐에서 순차적으로 코드가 실행되도록 스케줄 할 수 있고, 다른 우선순위로 다른 큐에서 동시에 멀티 테스크를 실행시키는데 사용할 수 있다. 
* Closures: 코드 안에서 나눠줄 수 있는 독립된 코드 조각을 생성한다. 그래서 다른 객체들이 그것을 얼마나 많이, 어떤 문맥에서 실행 여부를 결정할 수 있다.

