# MVP, 그리고 non-MVC에서 공통적으로 고려해야할 것들



## Model - View - Presenter?

1990년대 초 IBM에서 최초로 구현되어 2006년 마틴 파울러의 소개로 널리 알려진 패턴이다.

사실 안드로이드 초기부터 구전으로 전해져 왔던 아키텍처로써, 뷰는 비즈니스 로직에 관련된 부분을 관여하지 못하도록 분리하고 Presenter로 넘겨 처리한다.

### 핵심 아이디어

<figure><img src="../../../.gitbook/assets/Untitled (37).png" alt=""><figcaption></figcaption></figure>

* 모든 UI 관련 비즈니스 로직을 프리젠터에서 처리함
* 뷰는 프리젠터 요청에 따라 수동적으로 UI를 처리함
* 개방 - 폐쇄 원칙 : 뷰보다 더 높은 수준의 요소인 프리젠터를 뷰의 변경으로부터 보호함
  * how? : 뷰는 반드시 인터페이스로만 프리젠터를 참조(Contract)
  * 의존성 역전의 원칙(DIP)을 적용하면 뷰와 프리젠터 서로가 인터페이스로만 각자를 참조하도록 가능함

### MVP 예시

```kotlin
interface TasksContract {
	interface View : BaseView<Presenter> {
		fun setLoading(active: Boolean)
		fun setAddTasks(tasks: List<Task?>)
		...
	}

	interface Presenter : BasePresenter {
		val filtering: TasksFilterType
		fun loadTasks(forceUpdate: Boolean)
		...
		fun openTaskDetails(requestedTask: Task)
		fun completeTask(completedTask: Task)
	}
}
```

* Todo 앱의 각 작업(task)를 보여주고 추가하는 화면을 위한 인터페이스 정의
* Contract 패턴 : 뷰와 프리젠터 간의 직접 연결을 끊음

#### 좋지 않은 사례

위의 코드에서 Presenter의 메서드들을 자세히 살펴보면 데이터를 넘겨주는 메서드가 있다는 것을 알 수 있다. openTaskDetails 메서드에서는 이름에서부터 좋지 않은 예감이 든다(행위에 관련된).

<figure><img src="../../../.gitbook/assets/Untitled (38).png" alt=""><figcaption></figcaption></figure>

위의 코드는 View에 구현을 보여준다. 구조를 살펴보면 Adpater가 itemListener를 사용해서 각각의 리스트 아이템을 관리하고 있다. 가장 안좋은 케이스는 TasksAdapter가 프리젠터를 직접 관여하고 있는 것이다. 그렇게 되면 각각의 리스트 아이템은 프리젠터를 전부 다 알게 되므로 필요 이상 인터페이스를 노출하게 된다.

**문제점**

* openTasksDetails() : 프리젠터 로직이 너무 구체적이다. 이렇게 되면 나중에 프리젠터가 변경되면 뷰까지 영향을 받게 되므로 개방 - 폐쇄 원칙을 위반하게 된다.
* 리스트 아이템의 각 상태에 대해 어떤 비즈니스 로직이 실행되어야 하는지 뷰의 아주 작은 요소까지 이해하고 있다. 즉, 뷰는 그리 수동적이지 못하게 된다.
* Fragment는 뷰로 보기 힘든데, 뷰의 구현체가 Fragment가 된 것도 좋은 예는 아니다.

### 해법

#### 수동적인 뷰(Passive View)

**뷰는 완전한 험블 객체(Humble Object)여야 한다.**

* 이러한 문제 때문에 마틴 파울러는 프리젠터라는 이름을 폐기하고 Supervising Controller라고 부를 것을 제안하기도 함
* **수동적인 뷰는 (플랫폼 의존적인 : context) 그리기 이외에는 아무것도 알 수 없도록 해야 한다**

#### 수동적인 뷰는 이벤트 처리를 어떻게 이해해야 할까?

* “\[완료] 버튼이 눌러졌다” 이상의 의미는 감춰져야 한다
* 좋은 대안 : 이벤트 핸들러를 프리젠터에서 구현, 그 후에 UI State에 데이터와 함께 이를 람다 함수 형태로 전달

#### 좋은 예시

<figure><img src="../../../.gitbook/assets/Untitled (39).png" alt=""><figcaption></figcaption></figure>

UiState 클래스의 정의는 Contract 인터페이스 안에 함께 공유한다.

TaskUiState 안에 이벤트 핸들러가 구현되어 있으므로, 뷰는 각 람다 함수를 연결만 해주면 된다.

⇒ 향후 이벤트 발생 시 비즈니스 로직이 변경된다고 해도 뷰는 전혀 영향을 받지 않는다.

이렇게 됨으로써, 인터페이스 분리의 원칙을 지킬 수 있다. 뷰의 리스트 어댑터는 자신이 받은 UiState 이외의 것을 알 필요도, 알 수도 없다.

### MVP의 장점

#### Fat Activity / Fragment 방지

#### 수동적인 뷰는 이벤트 처리를 어떻게 이해해야 할까?

* 뷰는 화면이 액터, 프리젠터는 사용자가 액터를 담당하므로 단일 책임 원칙(SRP)에 명확해짐

#### 테스트 가능성 증대

* 플랫폼 의존적인 UI 처리는 뷰에 분리되었기 때문에, 프리젠터는 JVM 내에서 테스트가 가능해짐

#### non-MVC 구조 중 비교적 완만한 학습 곡선

* 타 구조에 비해 직관적
* 비동기 관련 처리 구조에서 UI의 결과를 뷰를 관리하는 인터페이스를 프리젠터가 호출하는 방식이기 때문에, 코루틴이나 RxJava 등의 리액티브 라이브러리가 반드시 필요하지 않음

### MVP의 단점

**프리젠터는 뷰의 기능을 기본적으로 메서드 호출을 통해서 호출한다.**

#### 뷰의 기능이 명령형(imperative)일 때 잘 동작함

* Jetpack Compose와 같은 선언형(declarative) 뷰에는 적용하기 까다로움(추가 UI State 처리를 위한 중재자가 필요함)

#### 기본적으로 뷰가 프리젠터를 참조하고 있음

* 개념적 상호 참조
* 잘못된 프리젠터 인터페이스 사용을 완전히 막기는 어려움

#### 완전히 수동적인 뷰를 구현하려면 많은 고민이 필요함

* 수동적인 부분이 부족한 뷰 인터페이스를 설계할 경우, 리스코프 치환 원칙을 위반하지 않으려면 뷰 구현이 필요 이상으로 많은 것을 생각해야 함(테스트도 의외로 쉽게 만들기 어려울 수 있음)

### MVP(그리고 non-MVC)에서의 주의사항

#### 안티패턴 : 비동기 처리의 결과를 동기적으로(보통 함수 리턴 값으로) 받는다

* Jank 혹은 ANR을 유발할 수 있음
* 실행 중에 라이프사이클이 변경되면 앱이 크래시 될 수 있음

#### 설정의 변경(Configuration Change)

* 설정 변경은 화면 회전 때만 일어나지 않음
  * 다크모드 변경, 화면 사이즈 변경, 언어 설정 변경 등등
* 만약 서버에서 로딩 이벤트를 요청했는데, 도중에 설정 변경이 일어난다면?
* 프로세스 종료
  * SavedInstanceState 등을 이용해 해결 가능
