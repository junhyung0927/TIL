# part 4. Coroutine Context and Dispatchers

코루틴은 항상 코틀린 표준 라이브러리에 정의된 [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 타입의 값으로 표현되는 일부 컨텍스트에서 실행된다.

코루틴 컨텍스트는 다양한 요소의 집합이다. 이번 파트에서 알아볼 주된 요소는 코루틴의 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html)과 코루틴의 dispatcher이다.

### Dispatchers and threads

코루틴 컨텍스트에는 해당 코루틴의 실행에 사용하는 스레드 또는 스레드(일반 스레드)를 결정하는 CoroutineDispather가 포함된다. 코루틴 dispatcher는 코루틴 실행을 특정 스레드로 제한하거나 스래드 풀로 dispatcher 하거나 제한되지 않은 상태로 실행할 수 있다.

[launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) 및 [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) 같은 모든 코루틴 builder는 새로운 코루틴 또는 기타 context 요소의 dispatchers를 명시적으로 지정하는데 사용할 수 있는 선택적 옵션인 CoroutineContext라는 파라미터를 허용한다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }    
}

//실행화면
Unconfined            : I'm working in thread main @coroutine#3
Default               : I'm working in thread DefaultDispatcher-worker-1 @coroutine#4
newSingleThreadContext: I'm working in thread MyOwnThread @coroutine#5
main runBlocking      : I'm working in thread main @coroutine#2
```

launch{ } 빌드는 매개 변수 없이 실행될 때, 실행중인 [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html)에서 context(dispatchers도 함께)를 상속한다. 이 경우 메인 스레드에서 실행되는 메인 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) 코루틴의 context를 받는다.

Dispatchers.Unconfined는 main 스레드에서도 실행되는 것처럼 보이는 특별한 dispatcher이지만, 사실은 다른 메커니즘이다. 이에 대해서는 나중에 설명한다.

scope에 다른 dispatcher가 명시적으로 지정되지 않은 경우 기본 dispatcher가 사용된다. 이것은 dispatcher의 기본값이며 스레드 풀의 백그라운드를 공유하여 사용한다.

newSingleThreadContext는 코루틴을 실행할 스레드를 만든다. 이렇게 만든 스레드는 매우 비싼 자원을 사용한다. 실제 애플리케이션에서는 newSingleThreadContext 스레드가 더 이상 필요하지 않은 경우, close 함수를 사용하여 릴리즈하거나 최상위 변수에 저장하여 애플리케이션 전체에 걸쳐 사용해야 한다.

### Unconfined vs confined dispatcher

[Dispatchers.Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html) 코루틴 디스패처는 호출한 스레드에서 코루틴을 시작하지만, 첫 번째 suspension 지점까지만 시작한다. suspension이 끝나면 호출된 suspension 함수에 의해 다른 스레드로 코루틴을 재개한다. unconfined dispatcher는 CPU 시간을 소비하거나 특정 스레드에 제한된 공유 데이터(UI 데이터와 같은)를 업데이트하지 않는 루틴에 적합하다.

반면에 confined dispatcher는 기본적으로 외부 CoroutineScope로부터 dispatcher를 상속받는다. 특히 runBlocking 코루틴의 기본 dispatcher는 호출한 스레드에 제한되므로 이를 상속받으면 예측 가능한 FIFO 스케줄링을 통해 이 스레드의 실행을 제한할 수 있는 효과가 있다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }    
}

//실행화면
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

runBlocking { ... }에서 context를 상속받은 코루틴은 main 스레드에서 계속 실행되는 반면에 unconfined 스레드는 delay 함수가 사용 중인 기본 executor 스레드에서 다시 실행되는 것을 확인할 수 있다.

> unconfined dispatcher는 코루틴의 일부 연산을 즉시 수행해야 하기 때문에 나중에 실행할 필요가 없거나 side-effect가 발생하는 특정 사례에서 도움이 될 수 있는 메커니즘이다. unconfined dispatcher는 일반 코드에서 사용해서는 안된다.

### RFC

***

[Coroutine context and dispatchers | Kotlin](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html#children-of-a-coroutine)
