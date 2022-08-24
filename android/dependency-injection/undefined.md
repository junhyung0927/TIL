# 의존성 주입이란 무엇인가?



### 의존성 주입이란?

**Dependency Injection**의 정의는 객체 인스턴스의 생성을 직접하지 않고 (다른 곳에서 만들어져서) 필요한 곳에 **전달**되도록 코드를 **조직화**하는 기법이다. 즉, 일관된 방법으로 정의되고 외부에서 필요한 경우 사용할 수 있는 기법이다.

* 객체 인스턴스를 사용하는 곳을 클라이언트라고 부름
* 의존성( a dependency): 사용되는 객체 인스턴스
* 클라이언트가 아닌, 잘 조직화 된 곳에서 일관성 있는 형태로 객체 생성 방법을 정의함

```kotlin
//클라이언트 내에서 직접 생성하는 경우
class NewsRemoteDataSource {
	private val newsApi = ProdNewsApi()
}

class NewsRemoteDataSource(
	private val newsApi: NewsApi
) { ... }
```

클라이언트 내에서 직접 생성하는 경우는 newsApi의 정의가 코드 안에 고정되는 문제가 발생한다. 개발용 백엔드 API와 프로덕션 백엔드 API를 스위칭하면서 사용하고 싶을 경우, 각각을 위한 데이터소스 클래스를 따로 만들어주는 경우가 발생할 수 있다. 이런 경우, 테스트가 어려워지고 외부 의존성에 깊게 관여될 수 있다.

### 왜 의존성 주입을 사용해야 하는가?

#### **코드를 재사용 가능하게 한다**

* 클라이언트 코드의 변경이 없어도 손쉽게 다른 형태의 DataSource 인스턴스를 만들 수 있다.
* 객체의 생성 과정이 바뀌더라도 클라이언트 코드에 영향을 주지 않는다(ex: 파라미터의 변경).

#### 클라이언트는 interface로만 객체를 알고 있으면 되므로, 보다 나은 설계를 가능하게 한다

* 클라이언트는 객체를 어떤 식으로 생성되는지 알 필요가 없기 때문에, 보다 추상화를 잘 할 수 있게 해준다.
* 멀티모듈에서 **모듈별 의존성**을 떼어내는 데에 매우 유용하다.

#### 코드를 테스트 가능하게 한다. 객체의 생성 방법을 설정에 따라 다르게 할 수 있다.

* 프로덕션을 위한 구현을 테스트용으로 간단하게 전환 가능하게 한다.

```kotlin
//의존성 주입을 사용하지 않으면 분기문에 의해 코드의 양이 많아진다.
class NewsRemoteDataSource {
	private val newsApi: NewsApi = when (BuildConfig.type()) {
		PROD -> ProdNewsApi()
		TEST -> FakeNewsApi()
	}
}

//의존성 주입을 사용하면 사용하고 싶은 객체를 NewsRemoteDataSource의 파라미터로 가져오면 된다. 
val fakeNewsRemoteDataSource = NewsRemoteDataSource(FakeNewsApi())
```

### Injector 클래스

Injector (혹은 Container) 클래스는 주입되는 모든 타입들을 위한 생성 방법을 정의한다. 즉, 모든 의존성 주입을 위한 인스턴스화가 일어나는 곳이다.

```kotlin
class ProdInjector {
	fun getNewsApi(): NewsApi = ProdNewsApi()
	fun getNewsRemoteDataSource(): NewsRemoteDataSource =
		NewsRemoteDataSource(getNewsApi())
}
```

### 왜 의존성 주입 프레임워크를 써야 하는가?

* 의존성 주입 과정에서 많은 양의 보일러 플레이트 코드가 필요하다.
* Injector 클래스를 유지보수 하는 것은 프로젝트가 커지면 커질 수록 힘들다.
* 앱 빌드의 설정에 따라 다른 종류의 Injector를 구현하는 것은 상당히 많은 양의 코드 중복을 만든다.
* DI 프레임워크가 없으면, Injector 클래스가 역설적으로 확장을 어렵게 만들 수 있다. ⇒ 분리가 어렵기 때문
* DI 프레임워크들은 멀티모듈을 쉽게 해준다.
* DI 프레임워크는 보통 의존성 정의를 위한 여러 단계의 계층을 제공한다. ⇒ 의존성 정의를 쉽게 유지보수하고 더 작은 단위로 쪼갤 수도 있다. 의존성 주입을 위한 여러 가지 단계들 중 바꾸고 싶은 부분만 수정할 수 있다.

### 어떤 DI 프레임워크를 선택할 것인가?

#### ServiceLocator 패턴

전역적으로 관리해야 할 클래스들이 아주 복합하지 않다면 ServiceLocator 패턴을 사용하는 것이 좋다. 아래의 기준만 충족해도 훌륭한 추상화가 가능하다.

* 일관된 바인딩 정의를 제공할 수 있는 저장소(== ServiceLocator)가 잘 구현되어 있어야 함
* 바인딩 정의가 분산 가능해야 함(멀티모듈을 위해) good example: ServiceLocator가 자식 SL을 갖는 트리 구조
* 현재 상태에서 가장 유효한 Context를 정확히 저장하고 있어야 함
* 굳이 전역적으로 정의할 필요가 없는 생성 로직들을 위해 표준화된 Factory 구조를 별도로 정의

또는, 사용자 정의로 구현할 수도 있고 Koin DI 라이브러리를 사용할 수도 있다.

#### ServiceLocator의 예

```kotlin
val container = Container(context)
container
	.bind(Foo1::class, FooImpl1())
	.bind(Foo2::class, FooImpl2())
	.bind(Foo3::class, FooImpl3())

//in client
val foo = container.get(Foo1::class)

//in another place
val childContainer = Container(context, parent = container)
childContainer.bind(Bar::class, BarImpl())

val foo2 = childContainer.get(Foo2::class)
val bar = childContainer.get(Bar::class)
```
