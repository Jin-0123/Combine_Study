# Chapter 15: In Practice: Combine & SwiftUI

### Declarative syntax

### Cross-platform

### New memory model

## Hello, SwiftUI!
* 데이터 모델이 변할 때마다 SwiftUI는 현재 뷰를 바꿀 것을 요청한다. 최신 데이터에 따라 변화할 수 있도록 함.

## Memory management

#### No data duplication
* 기존 방식은 조건에 따라 데이터를 화면에 노출할 때, 디팬던시를 추가해야하고 데이터를 중복으로 저장해야한다.
* SwiftUI는 데이터 중복을 제거하고, 한 곳에 효율적으로 데이터를 관리한다.

### Less need to "control" your views
* SwiftUI의 선언적 UI 문법 기초 다루기
* UI 인풋의 다양한 타입 어떻게 선언하는지 알아보기
* 파이프데이터와 데이터 모델을 세울 때 컴바인 사용하기

## Experience with SwiftUI

## Getting started with "News"

## A first taste of managing view state
* presentingSettingsSheet 변수를 사용해 Settings 버튼 클릭 시, 시트 노출하기
* @State
  * view 의 바깥 공간으로 프러퍼티 저장이 이동한다. 변수가 변경되는 것이 self를 변경하지 않는다.
  * 로컬 저장소로서 표시된다. 즉, 뷰의 자신의 데이터 일부.
  * $로 구독하거나 바인드 할 수 있음.
  
## Fetching the latest stories

## Using ObservableObject for model types
* ObservableObject 구현해서 적절한 메모리 관리와 함께 SwiftUI 뷰와 데이터 바인딩하기
* @ObservedObject
  * 뷰로부터 프러퍼티 저장에서 제거된다. 대신 그리고 오리지널 모델을 바인딩하는데 사용된다. 즉, 데이터 복제안함.
  * 바깥 공간 프러퍼티로 마크된다. 즉, 뷰의 자신의 데이터가 아님.
  * @Published, @State 연산자와 같이 바인딩하고, 구독하는데 사용함.
  
## Displaying errors
* API 에러 시, 에러 얼럿 노출하기
~~~
.alert(item: self.$model.error) { error in 
  Alert(
    title: Text("Network error"),
    message: Text(error.localizedDescription), dismissButton: .cancel()
  ) 
}
~~~

## Subscribing to an external publisher
* 다른 퍼블리셔 주입하기
* 현재 시간에 맞춰서 포스트 시간 텍스트 변경해보기
  * 타이머 퍼블리셔 생성, onReceive 호출될 때 현재 시간으로 변경.
  * 현재 시간 변수에 @State 붙여주기
  
## Initializing the app's settings
* 메인 리터뷰에서 스토리 필터 적용하기

## Editing the keywords list

### System environment
* 시스템 환경 변수 사용해보기
* 시스템 테마에 따라 링크 색 변경하기
~~~
// 컬러 스킴
@Environment(\.colorScheme) var colorScheme: ColorScheme

// 색 선언한 곳에서 사용
.foregroundColor(self.colorScheme == .light ? .blue : .orange)
~~~

## Custom environment objects
* @EnvironmentObject 사용해서 필터 키워드 추가 / 삭제 / 이동 실습하기
