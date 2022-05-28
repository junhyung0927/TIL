# Koin DSL

Kotlin의 파워 덕분에, Koin은 앱에 애노테이션을 추가하거나, 코드를 생성하는 대신에 앱의 설명을 도와주는 DSL을 제공한다. Kotlin DSL을 사용해서, Koin은 의존성 주입을 준비할 수 있는 스마트 기능 API를 제공한다.

#### Application & Module DSL

Koin은 Koin 응용 프로그램의 요소를 설명할 수 있는 몇 가지 키워드를 제공한다.

* 응용 프로그램 DSL, Koin 컨테이너 구성 설명
* 모듈 DSL, 주입해야 하는 컴포넌트 설명

#### Application DSL

`KoinApplication` 인스턴스는 Koin 컨테이너 인스턴스 구성이다. 이렇게 한다면 로깅, 프로퍼티 로딩, 모듈을 구성할 수 있다.

새로운 `KoinApplication` 을 빌드한다면, 다음의 함수들을 사용해라.

* `koinApplication { }` - `KoinApplication` 컨테이너 구성 생성
* `startKoin { }` - `KoinApplication` 컨테이너 구성을 생성하고 GlobalContext API를 사용할 수 있도록 `GlobalContext` 에 등록

`KoinApplication` 인스턴스를 구성하기 위해서, 다음과 같은 함수 중 하나를 사용해라.

* `logger( )` - 사용할 레벨 및 Logger 구현을 하기 위한 설명 (default로 EmptyLogger를 사용)
* `modules( )` - 컨테이너에 로드할 Koin 모듈의 목록을 설정(list or vararg list)
* `properties()` - HashMap 속성을 Koin 컨테이너에 로드
* `fileProperties( )` - 지정된 파일에서 Koin 컨테이너로 프로퍼티 로드
* `environmentProperties( )` - OS 환경에서 Koin 컨테이너로 프로퍼티 로드

#### KoinApplication instance: Global vs Local

위에서 보시다 싶이, 2가지 방법으로 Koin 컨테이너 구성을 설명할 수 있다

⇒ `koinApplication` or `startKoin`

* `koinApplication` 은 Koin 컨테이너 인스턴스를 설명한다.
* `startKoin` 은 Koin 컨테이너 인스턴스를 설명하고 Koin `GlobalContext` 에 등록한다.

컨테이너 구성을 `GlobalContext`에 등록함으로써, global API는 컨테이너 구성을 직접 사용할 수 있다.

모든 `KoinComponent` 는 `Koin` 인스턴스를 나타낸다. 기본적으로 GlobalContext 중 하나를 사용한다.

#### Starting Koin

Koin을 시작하면 `KoinApplication` 인스턴스를 `GlobalContext`로 실행할 수 있다.

모듈을 사용해서 Koin 컨테이너를 시작하려면, 다음과 같이 StartKoin 함수를 사용할 수 있다.

```kotlin
//Global context의 KoinApplication 시작
startKoin {
	//logger 사용 정의
	logger()
	//모듈 사용 정의
	modules(coffeeAppModule)
}
```

#### Module DSL

Koin 모듈은 애플리케이션을 위해 inject/combine 할 선언 키워드를 수집한다. 새로운 모듈을 생성하고, 다음과 같은 함수를 사용할 수 있다.

* `module { // module content}` - Koin 모듈 생성

모듈의 내용을 설명하기 위해, 다음과 같은 함수를 사용할 수 있다.

* `factory { //definition }` - factory bean 선언 키워드 제공
* `single { //definition }` - 싱글톤 bean 선언 키워드 제공 (또한 bean이라고도 불린다)
* `get ()` - 컴포넌트 의존성 해결 (또한 scope or parameters라고 불린다)
* `bind()` - 주어진 bean 선언 키워드를 위해 bind에 유형 추가 / 지정된 컴포넌트의 타입을 추가적으로 바인딩해준다.
* `binds()` - 주어진 bean 선언 키워드를 위해 array에 유형 추가
* `scope { //scope group }` - scoped 선언 키워드에 대한 logical 그룹 정의
* `scoped { //definition }` - 오직 scope 내에서만 존재할 경우 bean 선언 키워드 제공.

#### Writing a module

Koin 모듈은 모든 컴포넌트를 선언할 수 있는 공간이다. 모듈 기능을 사용해서 Koin 모듈을 선언한다.

```kotlin
val myModule = module {
	//의존성 주입을 위한 컴포넌트 생성
}
```

해당 모듈에서는 아래 설명된 대로 컴포넌트를 선언할 수 있다.
