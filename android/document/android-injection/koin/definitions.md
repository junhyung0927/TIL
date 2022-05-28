# Definitions



#### Defining a singleton

**싱글톤 컴포넌트**를 선언한다는 것은 Koin 컨테이너가 선언된 컴포넌트의 고유한 인스턴스를 유지함을 의미한다. 모듈의 **`single` 함수를 사용해서 싱글톤을 선언**한다.

```kotlin
class MyService()

val myModule = module {
	//MySevice 클래스에서 single 인스턴스를 선언
	single { MyService() }
}
```

#### Defining your component within a lambda

single, factory & scoped 키워드를 사용하면 람다식을 통해 컴포넌트를 선언할 수 있다. 이 람다는 컴포넌트를 구성하는 방법을 설명한다. 일반적으로 컴포넌트 생성자를 통해 인스턴스화하지만, 표현식도 사용할 수 있다.

`single { Class constructor //Kotlin expression }`

람다의 결과 타입은 컴포넌트의 main 타입이다.

#### Defining a factory

**factory** 컴포넌트는 **선언할 때마다 새 인스턴스를 제공**하는 방식이다(factory 인스턴스는 다른 곳에서 선언될 때 해당 인스턴스를 주입하지 않으므로 Koin 컨테이너에 의해 유지되지 않는다).

람다와 함께 `factory` 함수를 사용해서 컴포넌트를 빌드할 수 있다.

```kotlin
class Controller()

val myModule = module {
	// Controller 클래스에서 factory 인스턴스를 작성하면 된다.
	factory { Controller() }
}
```

🗨️ Koin 컨테이너는 선언할 때마다 새로운 인스턴스를 제공하기 때문에 factory 인스턴스를 보유하지 않는다.



#### Resolving & injecting dependencies

이제 컴포넌트 요소 선언 키워드를 선언할 수 있으므로, 인스턴스를 의존성 주입과 연결할 수 있다.

**Koin 모듈의 인스턴스를 주입하려면, 요청된 컴포넌트 요소 인스턴스에 `get()` 함수**를 사용해야 한다. `get()` 함수는 일반적으로 **생성자 값을 주입**하기 위해 생성자에 사용된다.

🗨️ Koin 컨테이너로 의존성 주입을 하려면, 생성자 주입 스타일로 써야한다: 클래스 생성자 의존성 해결 이런 방법으로 사용하면 Koin에서 주입된 인스턴스로 인스턴스가 생성된다.



```kotlin
//Presenter <- Service
interface Service1() {
	@GET("/test")
	fun a()
}

interface Service2() {
	@GET("/test")
	fun b()
}

class Controller(val service: Service(), val service2 : Service2()) {

val networkModule = module {
	single { get<Retrofit>().create(Service::class.java) }
	single { get<Retrofit>().create(Service2::class.java) }
	single { Retrofit.Builder().build() }
}

val myModule = module {
	//Service 클래스의 single 인스턴스 선언
	//Controller 클래스의 single 인스턴스를 선언하고, get() 메서드와 함께 View 인스턴스를 주입한다.
	single { Controller(service = get(), service2 = get()) }
}
```

#### Definition: binding an interface

`single` 또는 `factory`는 지정된 람다 유형을 사용한다. 즉, `single { T }`와 일치하는 타입은 해당 표현식에서 일치하는 유일한 타입이다.

클래스 및 구현된 인터페이스의 예를 살펴보자.

```kotlin
//Service interface
interface Service{
	fun doSomething()
}

//Service Implementation
class ServiceImp() : Service {
	fun doSomething() { ... }
}
```

Koin 모듈에서는 다음과 같이 `as` 캐스트 Kotlin 연산자를 사용할 수 있다. 해당 스타일이 선호된다.

```kotlin
val myModule = module {
	//SeviceImp 타입만 일치
	single { ServiceImp() }

	//Sevice 타입만 일치
	single { ServiceImp() as Service }
}
```

**유추된 타입식을 사용할 수도 있다.**

```kotlin
val myModule = module {
	//ServiceImp
	single { ServiceImp() }

	//Service 타입의 객체를 받아오기 때문에 Imp에 대한 의존성이 낮아짐.
	single<Service> { ServiceImp() }
}
```

[https://woovictory.github.io/2019/07/08/DI/](https://woovictory.github.io/2019/07/08/DI/)

### Additional type binding

경우에 따라서는, 하나의 선언 키워드에서 여러 가지 타입을 일치시키려 한다.

클래스 및 인터페이스의 예를 살펴보자.

```kotlin
//Service interface
interface Service{
	fun doSomething()
}

//Service Implementation
class ServiceImp() : Service {
	fun doSomething() { ... }
}
```

바인딩 추가 타입을 만들려면, 바인딩 연산자를 클래스와 함께 사용한다.

```kotlin
val myModule = module {
	//ServiceImp, Service 클래스를 ServiceImp 타입으로 일치
	single { ServiceImp() } **bind Service::class**
}
```

여기서는 get()을 통해 Service 타입을 직접 확인할 수 있다. 그러나 Service를 바인딩하는 선언 키워드가 여러 개일 경우 bind<>() 함수를 사용해야 한다.

#### Definition: naming & default bindings

선언 키워드에 이름을 지정하여, 동일한 타입에 대한 두 가지 선언 키워드를 구분할 수 있다:

선언 키워드를 이름으로 요청하면 된다:

```kotlin
val myModule = module {
	single<Service>(named("default")) { ServiceImpl() }
	single<Service>(named("test")) { ServiceImpl() }
}

val service : Service by inject(qualifier = named("default"))
```

get() 및 inject() 함수를 사용함으로써 필요한 경우 선언 키워드 이름을 지정할 수 있다.

이 이름은 named() 함수에서 생성된 'qualifier'이다.

```kotlin
val myModule = module {
	single<Service> { ServiceImpl1() }  
	single<Service>(named("test")) { ServiceImpl2() }
}
```

* `val service : Service by inject()` 는 `ServiceImpl1` 를 트리거 할 것이다.
* `val service : Service by inject(named("test"))` 는 `ServiceImpl2` 를 트리거 할 것이다.

#### Declaring injection parameters

모든 `single`, `factory` 또는 `scoped` 선언 키워드에서 injection 파리미터, 즉 **선언 키워드에 의해서 주입되고 사용될 파라미터를 사용**할 수 있다.

```kotlin
class Presenter(val view : View)

**class TestViewModel(id: Long, private val repository: Repository) {
}

// in view
val id = 5;
private val viewmodel: TestViewModel by viewmodel { parameterOf(id) } 

val viewModelModule = module {
	viewModel { (id: Long) -> TestViewModel(id, repository=get()) }
}**

val myModule = module {
	//view 매개변수를 받아 Presenter의 파라미터로 넣어주는 과정?
	single{ (view : View) -> Presenter(view) }
}
```

해결된 의존성과는 반대로, inject 파라미터는 resolution API를 통해 전달되는 파라미터다. 즉, 이러한 파라미터는 `parametersOf` 와 함께 `get()` 및 `inject()` 파라미터를 사용해서 전달된 값이다.

```kotlin
val presenter : Presenter by inject { parametersOf(view) }
```

#### Using definition flags

Koin DSL은 몇 가지 flag들을 제안한다.

**Create instances at start**

선언 키워드 또는 모듈은 `CreatedAtStart` 로 플래그가 지정될 수 있으며, 시작될 때나 원할 때 생성된다.

첫 번째로 선언 키워드 또는 모듈에 `createOnStart` 플래그를 설정한다.

* 선언 키워드의 CreatedAtStart 플래그

```kotlin
val myModuleA = module {
	//get() 호출할 떄 생성
	single<Service> { ServiceImp() }
}

val myModuleB = module {
	//eager creation for this definition 
	//호출 없이 즉각적 생성
	single<Service>(createdAtStart=true) { TestServiceImp() }
}
```

* module의 CreatedAtStart 플래그

```kotlin
val myModuleA = module {
	single<Service> { ServiceImp() }
}

val myModuleB = module(createdAtStart=true) {
	single<Service>{ TestServiceImp() }
}
```

`startKoin` 함수는 `createdAtStart` 로 플래그가 지정된 선언 키워드 인스턴스를 자동으로 생성한다.

```kotlin
startKoin {
	modules(myModuleA, myModuleB)
}
```

🗨️ UI 대신에 백그라운드 스레드를 사용하는 것과 같은 특별한 시간에 몇 가지 선언 키워드를 로드하고 싶다면, 오직 get/inject 컴포넌트를 사용해야 한다.

#### Dealing with generics

Koin은 일반 타입 인자를 고려하지 않는다. 예를 들어, 아래의 모듈은 2가지의 list를 정의하려 한다.

```kotlin
module {
	single { ArrayList<Int> }
	single { ArrayList<String> }
}
```

Koin은 이러한 선언 키워드로 시작하지 않고, 하나의 선언 키워드를 다른 선언 키워드로 재정의하려 한다.

이를 허용하려면, 이름 또는 위치(모듈)를 통해 구별해야 하는 두 가지 선언 키워드를 사용한다.

```kotlin
module {
	single(named("Ints")) { ArrayList<Int>() }
	single(named("Strings")) { ArrayList<String>() }
}
```
