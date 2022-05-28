# Start Koin

Koin은 DSL, 경량 컨테이너, 실용적인 API이다. Koin 모듈에 선언 키워드 및 정의를 선언하면, Koin 컨테이너를 시작할 준비가 된 것이다.

### The startKoin function

`startKoin` 함수는 Koin 컨테이너를 시작할 때 main 진입점을 제공한다. 실행할 Koin 모듈의 list가 필요하다. 모듈들이 로드되고 선언 키워드들을 Koin 컨테이너로 확인할 수 있다.

```kotlin
//start a KoinApplication in Global context
startKoin {
	//declare used modules
	modules(coffeeAppModule)
}
```

startKoin이 실행되면, Koin은 모듈 및 선언 키워드들을 읽을 것이다. 그러면 Koin은 필요한 인스턴스를 검색하기 위해 get() 또는 injection() 호출을 통해 준비된다.

Koin 컨테이너의 몇 가지 옵션들:

* `logger` - logging을 허용
* enviroment, [koin.properties](http://koin.properties) file, extra properties에서 프로퍼티들을 로드하는 `properties()` , `fileProperties()` , `enviromentProperties()` 옵션들

🗨️ `startKoin`은 두 번 이상 부를 수 없다. 모듈을 로드하기 위해서 여러 지점이 필요한 경우, `loadKoinModules` 함수를 사용해야 한다.



### Behind the start - Koin instance under the hood

Koin을 시작할 때, Koin 컨테이너 구성 인스턴스를 나타내는 **KoinApplication 인스턴스를 생성**한다. 일단 출시되면, KoinApplication은 모듈들과 옵션들로 인한 **Koin 인스턴스를 생산**할 것이다. 그렇게 되면, Koin 인스턴스는 **GlobalContext**에 의해 유지되며, 모든 **KoinComponent** 클래스에서 사용된다.

GlobalContext는 Koin의 기본 JVM 컨텍스트 전략이다. 이것을 startKoin이라 부르고, GlobalContext에 등록한다. 이를 통해 Koin Multipatform의 관점에서 다른 종류의 컨텍스트를 등록할 수 있다.

### Loading modules after startKoin

`startKoin` 메서드는 두 번이상 호출될 수 없다. 하지만 `loadKoinModules()` 함수를 직접 사용할 수 있다.

이 함수는 Koin을 사용하는 SDK를 만드는 사람들에게 많이 사용된다. 왜냐하면 그들은 `startKoin()` 함수를 사용할 필요가 없고 라이브러리 시작 시 loadModules만 사용할 수 있기 때문이다.

```kotlin
loadKoinModules(module1, module2 ...)
```

### Unloading modules

여러 선언 키워드들을 unload한 다음, 지정된 함수를 사용해서 인스턴스를 release 할 수 있다.

```kotlin
unloadKoinModules(module1, module2 ...)
```

### Koin context isolation

SDK Makers의 경우 글로벌하지 않은 방법으로 Koin과 함께 작업할 수 있다. 라이브러리의 DI에 Koin을 사용하고, 사용자의 컨텍스트를 분리하여 라이브러리 및 Koin을 사용하는 사용자들에 대한 충돌을 피해야 한다.

표준적인 방법으로, 다음과 같이 koin을 시작해라.

```kotlin
//start a KoinApplication and register it in Global context
startKoin {
	//declare used modules
	modules(coffeeAppModule)
}
```

이를 통해, `KoinComponent` 를 사용할 수 있다: `GlobalContext` Koin 인스턴스를 사용할 수 있을 것이다.

하지만 분리된 Koin 인스턴스를 사용하고 싶다면, 다음과 같이 사용하면 된다.

```kotlin
//create a KoinApplication
val myApp = koinApplication {
	//declare used modules
	modules(coffeeAppModule)
}
```

`myApp` 인스턴스를 라이브러리에서 사용할 수 있도록 유지한 후, 사용자 지정 KoinComponent 클래스에 전달해야 한다.

```kotlin
//Get a Context for your Koin instance
object MyKoinContext {
	var koinApp : KoinApplication? = null
}

//Register the Koin context
MyKoinContext.koinApp = KoinApp
```

```kotlin
abstract class CustomKoinComponent : KoinComponent {
	//Override defalut Koin instance, intially target on GlobalContext to yours
	override fun getKoin(): Koin = MyKoinContext?.koinApp.koin
}
```

이제 컨텍스트를 등록하고 분리된 Koin 컴포넌트 요소를 실행한다.

```kotlin
//Register the Koin context
MyKoinContext.koinApp = myApp

class ACustomKoinComponent : CustomKoinComponent(){
	//inject & get will target MyKoinContext
}
```

### Stop Koin - closing all resources

모든 Koin 리소스를 종료하고 인스턴스 및 정의들을 삭제할 수 있다. 이를 위해 어디에서나 stopKoin() 함수를 사용해서 Koin GlobalContext를 중지 할 수 있다. 그렇지 않으면 KoinApplication 인스턴스에서 close()를 호출하면 된다.
