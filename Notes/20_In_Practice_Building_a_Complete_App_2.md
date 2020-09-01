# Chapter 20: In Practice: Building a Complete App_2

## Trying out your app

## Your progress so far

* 남은 작업
   2. 호감 조크를 저장, 나중에 읽기위해
   8. 저장된 조크 리스트에 올리기
   9. 저장된 리스트 영어, 스페인어 토글
   10. 저장된 조크 삭제
    
## Implementing Core Data with Combine

## Review the data model

    * Core Data는 JokeManagedObject에 대한 클래스 정의를 자동 생성. JokeManagedObject의 익스텐션 메서드와 JokeManagedObject 컬렉션에 메서드를 생성해서 조크를 저장하고 삭제.

## Extending JokeManagedObject to save jokes

~~~
// CoreData, SwiftUI 등 필요 라이브러리 import
import Foundation
import SwiftUI
import CoreData
import ChuckNorrisJokesModel

// JokeManagedObject 클래스 확장
extension JokeManagedObject {

    static func save(joke: Joke, inViewContext viewContext: NSManagedObjectContext) {
        // 에러일 경우는 저장하지 않음
        guard joke.id != "error" else { return }
        // JokeManagedObject 엔티티 이름으로 fetch request 생성
        let fetchRequest = NSFetchRequest<NSFetchRequestResult>(entityName: String(describing: JokeManagedObject.self)) 
        // id와 동일한 조크 가져오기
        fetchRequest.predicate = NSPredicate(format: "id = %@", joke.id)
        
        // 패치 리퀘스트 실행, 성공하면 원래 있던 값을 전달된 값으로 업데이트
        if let results = try? viewContext.fetch(fetchRequest), let existing = results.first as? JokeManagedObject {
            existing.value = joke.value
            existing.categories = joke.categories as NSArray 
            existing.languageCode = joke.languageCode 
            existing.translationLanguageCode = joke.translationLanguageCode
            existing.translatedValue = joke.translatedValue
        } else { 
            // 존재하지 않으면 새 값으로 저장
            let newJoke = self.init(context: viewContext) 
            newJoke.id = joke.id
            newJoke.value = joke.value
            newJoke.categories = joke.categories as NSArray 
            newJoke.languageCode = joke.languageCode newJoke.translationLanguageCode =
            joke.translationLanguageCode
            newJoke.translatedValue = joke.translatedValue 
        }
        
        // 저장 시도!
        do {
            try viewContext.save()
        } catch {
            fatalError("\(#file), \(#function), \(error.localizedDescription)")
        }
    }
}
~~~

## Extending collections of JokeManagedObject to delete jokes

~~~
extension Collection where Element == JokeManagedObject, Index == Int {
    // 삭제하기 위해서 인덱스 셋 전달
    func delete(at indices: IndexSet, inViewContext viewContext: NSManagedObjectContext) {
        // 인덱스만큼 delete 반복 호출
        indices.forEach { index in
            viewContext.delete(self[index]) 
        }
        
        // 저장 시도!
        do {
            try viewContext.save()
        } catch {
            fatalError("\(#file), \(#function), \(error.localizedDescription)") 
        }
    }
}
~~~

## Create the Core Data stack

* Core Data 스택을 셋업할 수 있는 여러가지 방법이 있지만 접근제어의 이점을 얻기 위해 SceneDelegate로만 접근할 수 있도록 스택을 생성한다.

* App/SceneDelegate.swift 위쪽에 import문 추가

~~~
import Combine
import CoreData
~~~

* CoreDataStack 정의 추가

~~~
// CoreDataStack 네임스페이스 역할만 하기 때문에 enum이 적절. 인스턴스화 할 필요없음.
private enum CoreDataStack {
    // 관리된 오브젝트 모델, 저장 코디네이터, 관리된 오브젝트 컨텍스트를 캡슐화하는 실제 코어 데이터 스택
    // SwiftUI’s Environment API 사용하므로써 앱 전체에 공유되어질 것.
    static var viewContext: NSManagedObjectContext = {
        let container = NSPersistentContainer(name:"ChuckNorrisJokes")
        container.loadPersistentStores { _, error in 
            guard error == nil else { fatalError("\(#file), \(#function), \ (error!.localizedDescription)") } 
        }
        return container.viewContext
    }()
    
    // 컨텍스트를 저장하는 스테틱 메서드
    // 저장 전에 이미 저장되었는지 확인할 것.
    static func save() {
        guard viewContext.hasChanges else { return }
        do {
            try viewContext.save() 
        } catch {
            fatalError("\(#file), \(#function), \(error.localizedDescription)")
        } 
    }
}
~~~

* `scene(_:willConnectTo:options:)` 메서드 변경

~~~
 let contentView = JokeView() 
    .environment(\.managedObjectContext, CoreDataStack.viewContext)
~~~

* `sceneDidEnterBackground(_:)` 메서드에 추가 

~~~
CoreDataStack.save()
~~~

## Fetching jokes

* Views/JokeView.swift 추가

~~~
@Environment(\.managedObjectContext) private var viewContext
~~~

* `handle(_:)`메서드  `let translation = change.translation` 전에 아래 코드 추가 

~~~
if decisionState == .liked {
    JokeManagedObject.save(joke: viewModel.joke, inViewContext: viewContext)
}
~~~

## Showing saved jokes

* `LargeInlineButton` 변경

~~~
LargeInlineButton(title: "Show Saved") {
    self.presentSavedJokes = true 
}
.padding(20)
~~~

* `NavigationView` 블럭 끝에 추가

~~~
.sheet(isPresented: $presentSavedJokes) { 
    SavedJokesView()
        .environment(\.managedObjectContext, self.viewContext) 
}
~~~
> `$presentSavedJokes` 새 값을 보낼 때마다 트리거
> 값이 `true`면 인스턴스화하고 저장된 조크의 뷰를 보여주고, `viewContext`에 전달

## Finishing the saved jokes view

* SavedJokesView.swift 프러퍼티 추가

~~~
@Environment(\.managedObjectContext) private var viewContext
~~~

* `private var jokes = [String]()` 대체

~~~
@FetchRequest(sortDescriptors: [NSSortDescriptor(keyPath: \JokeManagedObject.value, ascending: true)], animation: .default)
private var jokes: FetchedResults<JokeManagedObject>
~~~
> 컴파일 오류 발생하지만 delete 기능 활성화하면 해결

* SwiftUI wrapper 프러퍼티
    * 패치되어 정렬된 sortDescriptors 배열을 가져로고 주어진 애니메이션에 따라 보여질 리스트를 업데이트
    * 저장소 변경될 때마다 자동으로 패치, 업데이트된 데이터로 리렌더링하기 위해 뷰를 트리거
    
## Deleting jokes

* `ForEach(jokes, id: \.self)`, `.onDelete` 블럭에 아래 코드 작성

~~~
ForEach(jokes, id: \.self) { joke in 
    // 번역될 값을 노출할지 안할지 결정.
    Text(self.showTranslation ? joke.translatedValue ?? "N/A" : joke.value ?? "N/A")
        .lineLimit(nil) 
}
.onDelete { indices in 
    스와이프하여 조크 삭제, delete 메서드 콜 호출
    self.jokes.delete(at: indices, inViewContext: self.viewContext)
}
~~~
