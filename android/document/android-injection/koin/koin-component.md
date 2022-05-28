# Koin Component

Koin은 모듈 및 선언 키워드들을 설명하는데 도움을 주는 DSL이고, 선언 키워드를 해결하기 위한 컨테이너이다. 지금 필요한 것은 컨테이너 외부에서 인스턴스를 검색하는 API이다. 그것이 Koin 컴포넌트의 목표이다.

🗨️ KoinComponent 인터페이스는 현재 기술 스택에서 제공하지 않는 인스턴스를 주입하는데 도움이 된다. `modules` 를 사용할 수 있으면 KoinComponent를 사용하지 마십시오. ⇒ Koin을 사용하는 이유가 의존성을 낮춰 코드 수정 및 유닛테스트를 용이하게 하기 위함인데, 이미 분리가 잘 되어있다면 KoinComponent를 구현할 필요는 없어보인다.



### Create a Koin Component

클래스에 Koin 기능을 사용할 수 있는 용량?(capacity)을 제공하려면 `KoinComponent` 인터페이스로 태그를 지정해야 한다.

MyService 인스턴스를 정의하는 모듈

```kotlin
class MyService

val myModule = module {
	//Define a singleton for MyService
	single { MyService() }
}
```

선언 키워드를 사용하기 전에 Koin을 시작해야 한다.

myModule과 함께 Koin 시작

```kotlin
fun main(vararg args : String) {
	//Start Koin
	startKoin {
		modules(myModule)
	}

	//Create MyComponent instance and inject from Koin container
	MyComponent()
}
```

이제 Koin 컨테이너로부터 인스턴스를 검색하는 `MyComponent` 를 쓸 수 있다.

MyService 인스턴스를 주입하기 위해 get() 및 inject() 사용

```kotlin
class MyComponent : KoinComponent {
	//lazy inject Koin instance
	val myService : MyService by inject()
	
	//or
	//eager inject Koin instance / 즉시 인스턴스를 가져오는?
	val myService : MyService = get()
}
```

### Unlock the Koin API with KoinComponents

클래스를 `KoinComponent`로 태그했다면, 다음 항목에 액세스 할 수 있다:

* `by inject()` - Koin 컨테이너로부터 구한 lazy 인스턴스(Koin에 등록된 객체를 lazy하게 받을 수 있음)
* `get()` - Koin 컨테이너로부터 가져온 eager 인스턴스
* `getProperty()` / `setProperty()` - get/set 프로퍼티

### Retrieving definitions with get & inject

Koin은 Koin 컨테이너로에서 인스턴스를 검색하는 두 가지 방법을 제공한다.

* `val t : T by inject()` - 인스턴스를 lazy하게 위임
* `val t : T = get()` - 인스턴스로부터 접근 허용

```kotlin
//is lazy evaluated
val myService : MyService **by inject()**

//retrieve directly the instance
val myService : MyService = **get()**
```

🗨️ The lazy inject form is better to define property that need lazy evaluation.



### Resolving instance from its name

필요한 경우 `get()` 또는 `by inject()`을 사용해서 다음 파라미터를 지정할 수 있다.

* `qualifier` - 선언 키워드 이름(선언 키워드에서 매개 변수 이름을 지정한 경우, 즉 동일 타입의 인스턴스가 여러 개 필요한 경우)

```kotlin
val module = module {
	single(named("A")) { ComponentA() }
	single(named("B")) { ComponentB(get()) }
}  

class ComponentA
class ComponentB(val componentA : ComponentA)
```

```kotlin
//지정된 모듈에서 검색
val a = get<ComponentA>(named("A"))
```

### No inject() or get() in your API?

API를 사용하고 있고 그 안에서 Koin을 사용하고 싶다면, `KoinComponent` 인터페이스로 원하는 클래스에 태그를 붙이면 된다.
