# Dagger의 기초와 중요 개념

### Dagger란?

Dagger는 코드 생성 기반의 의존성 주입 프레임워크다. 안드로이드에서는 매우 느린 리플렉션을 사용한 기존 프레임워크보다 안드로이드에 더 적합하지만, 안드로이드 전용으로 만들어진 것은 아니다.

* Android를 위한 표준 DI 프레임워크. Android를 위해서 만들어진 것은 아니지만, 구글에서 권장하고 있는 DI 프레임워크이다.
* JSR-330 표준을 annotation processor 기반 코드 생성을 이용해서 구현했다.
* 단, annotation을 통한 코드 생성이 일어나므로 빌드 시간에 상당한 오버헤드가 발생한다.
* Android Studio 지원
  * 실시간으로 코드 생성 결과를 알 수 있고, lint 지원도 있음
  * 에러 발생 시, 정확한 에러 원인을 알려줌
* 모든 구현이 코드 생성 기반이므로 실행 속도가 빠르다.
* Android에서 사용할 수 있는 DI 툴들 중 가장 강력하고 유연하며, 방대한 양의 기능을 제공한다.

### 간단하게 시작하는 Dagger 첫 걸음

#### Gradle 설정

```kotlin
//app.gradle
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
}

...

dependencies {
    implementation "com.google.dagger:dagger:$dagger_version"
    kapt "com.google.dagger:dagger-compiler:$dagger_version"
}

//최상단 build.gradle
buildscript {
    ext {
        dagger_version = '2.43.2'
    }
}
```

#### 컴포넌트 구현

```kotlin
import dagger.Component

@Component
interface ApplicationComponent {
    fun inject(activity: MainActivity)
}
```

#### Application(#onCreate()), Activity#onCreate() 초기화

```kotlin
class MyApplication : Application() {
    val appComponent = DaggerApplicationComponent.create()
}

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        (applicationContext as MyApplication).appComponent.inject(this)
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

#### 바인딩 구현

```kotlin
class Foo @Inject constructor() {
    init {
        Log.i("Foo", "Hello, DI World")
    }
}
```

#### 주입 구현

```kotlin
class MainActivity : ComponentActivity() {
    @Inject lateinit var foo: Foo
		...

}
```

### Dagger 주요 개념

#### Components

* Dagger가 객체들의 생성을 관리하기 위해 생성하는 injector 클래스의 원형
* **interface 혹은 abstract class로 정의해야 함**
* Builder / Factory 인터페이스를 제공
* 아래 방법은 컴포넌트의 메서드 형태로 객체 인스턴스를 얻을 수 있게 해줌

```kotlin
@Component
interface AppGraph {
	fun newsApi(): NewsApi
}
// 사용 예 : val newsApi = Application.appGraph.newsApi()
```

위의 코드를 컴파일 하면

```java
@Component("dagger")
class DaggerAppGraph implements AppGraph {
	@Overrides NewsApi newsApi() {
		return ...
	}
}
```

#### Bindings

어떤 클래스(key)가 그 클래스를 실제로 생성할 수 있는 방법을 value로 만든 해쉬테이블이라고 생각하면 이해하기 좋다.

<figure><img src="../../.gitbook/assets/Untitled (30) (1).png" alt=""><figcaption></figcaption></figure>

각 객체의 생성이 실제로 정의되는 곳이다(ex: @Provides 메서드)

```kotlin
@Provides
fun provideNewsRemoteDataSource(newsApi: NewsApi): NewsRemoteDataSource = 
	NewsRemoteDataSource(newsApi)

@Provides
fun provideNewsApi(newsApi: ProdNewsApi): NewsApi = newsApi
```

**함수의 매개변수가 있는 경우에 유의**해야 한다.

* 매개 변수가 있다면 또 다른 바인딩이 있는지 검색한 후, 있다고 하면 그것을 주입 해줘야 한다. 만약 없다면, 컴파일 에러가 발생한다. 이미 바인딩이 되어 있다면, 특별히 고민할 필요 없이 알아서 설정 해준다.
* 함수의 매개 변수가 있는 경우, 또 다른 바인딩을 검색해서 주입해줌

#### Binding Keys

Foo, Bar 라는 식별자가 있을 때, 이를 key라고 부른다.

⇒ Dagger에게 어떤 바인딩이 사용되어야 하는지를 쉽게 정의할 수 있다. 하지만 상당히 엄격한 편이다.

**🔎 주의**

* List와 ArrayList는 다른 식별자로 인식한다.
* ArrayList 타입의 Binding Key를 설정한다면, 주입할 때도 반드시 ArrayList 타입을 사용해야 한다.
* List\<String>과 List\<Integer>도 다르게 인식한다.
* @Provides ArrayList\<String>으로 바인딩을 설정한 다음 List\<String>을 주입하려고 하면 에러가 발생한다.

#### @Inject

_일반적이지만, 장황한 표현_

```kotlin
@Provides
fun provideNewsRemoteDataSource(newsApi: NewsApi): NewsRemoteDataSource {
	return NewsRemoteDataSource(newsApi)
}
```

_위와 동일하지만, 간결한 표현_

```kotlin
class NewsRemoteDataSource @Inject constructor(newsApi: NewsApi) {
}
```

@Inject는 인터페이스에서 사용될 수 없다. @Inject를 통해서 어떤 타입이 나와야 되는지를 이미 코드에서 보여주고 있다. 즉, 해당 클래스 밖에 없다는 것을 알 수 있다.

#### Qualifiers(수식자)

* 하나의 타입에 대해서 다른 인스턴스를 전달하고 싶을 경우가 있음
* 타입과 qualifiers의 조합으로 바인딩 키를 만들 수 있음
  * retrofit, http interceptor가 일반적인 인스턴스와 인증에서만 사용되는 두 가지 경우가 있을 때
  * Context는 ApplicationContext의 인스턴스일 수도, Activity Context 일 수도 있음

```kotlin
@Qualifiers
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifiers
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient
```

정의된 qualifiers는 어디에서든 타입과 함께 사용될 수 있다.

```kotlin
@AuthInterceptorOkHttpClient
@Provides
fun provideAuthInterceptorOkHttpClient(
	authInterceptor: AuthInterceptor
): OkHttpClient = 
		OkHttpClient.Builder()
			.addInterceptor(authInterceptor)
			.build()

// 

@AuthInterceptorOkHttpClient
@Inject lateinit var okHttpClient: OkHttpClient
```

#### Module

* 단순히 바인딩들의 모음이다.
* 멀티모듈의 모듈(gradle module)과는 전혀 다른 개념이다.
* 모듈은 컴포넌트 안에서 설치된다(혹은 다른 모듈들 안에 포함됨).
* 주의
  * 하나의 모듈이 여러 개의 컴포넌트에 설치될 수 없음
  * 모듈이 (개발자 실수로) 아무데도 속하지 않아도 컴파일 에러가 나지 않음

```kotlin
@Component(modules = [NetworkModule::class])
interface ApplicationComponent {
	...
}
```

### 필드 주입(Field injection)

#### 의존성 주입을 위한 다른 방법

* 필드 주입은 생성자를 통하지 않고 주입하는 방법이다. (반대: 생성자 주입)
* 내부적으로 같은 패키지 내에 값만 대입하기 위한 클래스를 생성하는 형태로 구현된다.
  * private으로 지정 불가능 (최소 protected 이상)
  * final로 지정 불가능 ⇒ 상속을 통해 주입되기 때문에
  * 추가 클래스 생성의 오버헤드가 있음
* 생성자 주입과는 달리 깔끔한 binding graph가 만들어지기 어려움
* Dagger가 직접 생성할 수 없는 클래스(ex: Activity or Fragment)에서 사용됨

```kotlin
class NewsRemoteDataSource {
	@Inject lateinit var newsApi: NewsApi
}
```

### Binding Graph

의존성들의 모든 경로는 그래프 형태(Directed Acyclic Graphs)로 표현될 수 있음

<figure><img src="../../.gitbook/assets/Untitled (32) (3).png" alt=""><figcaption></figcaption></figure>

### Dagger는 그래프를 순회한다

인사이트: Dagger가 코드를 처리할 때, component 메서드에서 시작해서 그래프를 완성하게 된다.

<figure><img src="../../.gitbook/assets/Untitled (32) (1).png" alt=""><figcaption></figcaption></figure>

### 멀티 컴포넌트

* 특정 범위(ex: Activity or Fragment) 안에서만 이뤄지는 바인딩이 필요한 경우, Subcomponent를 사용하면 된다.

```kotlin
@Subcomponent(modules = [ClickActivityModule::class])
interface ActivityComponent {
	fun clickManager(): ClickManager
}
```

Subcomponent를 사용할 때 제약 사항이 존재한다. Subcomponent를 컴포넌트에 설치할 때는 어노테이션 안에 모듈이 포함되어 있는데, 이를 제외한 다른 컴포넌트를 직접 넣을 수 있는 방법은 없다. 반드시 모듈을 하나 더 만들어서 넣어야 한다.

### Dagger 실제 예

```kotlin
@Module
class ProdNewsModule {
	@Provides
	fun provideProdNewsApi(newsApi: ProdNewsApi): NewsApi = newsApi
}

@Module
class FakeNewsModule {
	@Provides
	fun provideFakeNewsApi(newsApi: FakeNewsApi): NewsApi = newsApi
}
```

```kotlin
class NewsRemoteDataSource @Inject constructor(newsApi: NewsApi)
{
}

interface NewsApi {}

class ProdNewsApi @Inject constructor(): NewsApi {
}

class FakeNewsApi @Inject constructor(): NewsApi {
}
```

```kotlin
@Component(modules = [ProdNewsModule::class])
interface ApplicationComponent {
	fun newsRemoteDataSource(): NewsRemoteDataSource
}

class MyApplication: Application {
	val appComponent = DaggerApplicationComponent.create()
	override fun onCreate()
	val newsRemoteDataSource = appComponent.newsRemoteDataSource()
	...
}
```

### 서브 컴포넌트는 다른 컴포넌트들의 자식

서브 컴포너넌트는 부모의 모든 바인딩을 상속 받는다. Application은 최상위 부모로써, 그 상위에 있는 컴포넌트들은 전부 다 상속을 받아서 사용이 가능하다. 당연히, 자신만 가지고 있는 것도 주입 받아 사용할 수 있다.

<figure><img src="../../.gitbook/assets/Untitled (33).png" alt=""><figcaption></figcaption></figure>

### 멤버 주입

대부분의 경우, Component에 일일히 getter 메서드를 만드는 것은 괴로운 일이다.

따라서 이런 형태는 생성의 세부 내용을 컨트롤 해야하는 경우에만 사용하고, @Inject를 통해서 주입되는 것이 효율적이다.

Application에서 갖고 있어야 하는 인스턴스들도 @Inject 필드 형태로 만드는 것이 편하다.

그리고 멤버 주입을 사용하면 에러에 취약하다. 주입 전까지는 값이 null이고, private이 아니기 때문에 외부에서 변경이 가능하고, final이 아니기 때문에 주입된 값이 변경될 수도 있다.



