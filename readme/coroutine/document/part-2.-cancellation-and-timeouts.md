# part 2. Cancellation and Timeouts



### 취소(Cancellation)

코루틴에서 실행되는 모든 중단 함수(suspending function)들은 취소 요청에 응답 가능하도록 구현되어야 한다. 즉, 중단 함수는 실행 중 취소 가능한 구간마다 취소 요청이 있었는지 확인한 후 실행을 즉시 취소하도록 구현되어야한다. kotlinx.coroutines 라이브러리의 모든 중단함수는 이러한 취소 요청에 대응하도록 구현되어있다.

취소를 지원하는 중단 함수들은 실행하는 동안 취소가 가능한 지점마다 현재 코루틴이 취소 되었는지 확인하며, 만약 취소 되었다면, CancellationException을 발생시키며 종료한다.

> _(ReactiveX Observable을 구현해 본 경험이 있다면, Observable의 코드 실행 중 취소 가능한 구간마다 isDisposed() 같은 취소 상태 확인 함수를 이용하여 현재 Observable의 구독 취소 여부를 확인하고 Observable의 데이터 방출을 정지하고 종료해야 하는지 판단했던 기억이 있을 것이다._

### Cancelling coroutine execution

장기간 실행되는 애플리케이션에서는 백그라운드 코루틴을 세밀하게 제어해야 한다. 예를 들어, 사용자는 코루틴을 시작한 페이지를 닫았을 수도 있고, 이제 그 결과는 더 이상 필요하지 않고 그 동작은 취소될 수 있어야한다. 이런 경우 launch 함수는 실행 중인 코루틴을 취소할 수 있는 job을 반환한다.

```kotlin
fun main() = runBlocking{
    val job = launch{
        repeat(1000) {i ->
            println("job: I'm sleeping $i")
            delay(500L)
        }
    }
    delay(1300L) //잠시 대기
    println("main: I'm tired of waiting!")
    job.cancel() //job 취소
    job.join() // job의 모든 일이 끝나기를 대기
    println("main: Now I can quit")
}

job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

`main()` 이 `job.cancel` 을 호출하는 즉시, 다른 코루틴이 취소되었기 때문에, 그 이후 어떤 출력도 볼 수 없다. `cancel` 과 `join` 호출을 결합한 Job 익스텐션인 `[cancelAndJoin](<https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/cancel-and-join.html>)` 함수도 있다.

### Cancellation is cooperative

코루틴의 취소는 굉장히 협조적이다. kotlinx.coroutine의 모든 suspend 함수는 취소할 수 있다. 이들은 코루틴의 취소를 확인한 후 CancellationException을 던진다. 단, 코루틴이 연산 작업 중이고 취소 여부를 확인하지 않는다면, 다음 코드처럼 취소할 수 없다.

```kotlin
fun main() = runBlocking{
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default){
        var nextPrintTime = startTime
        var i = 0
        while(i < 5){ //연산 루프, CPU를 소비한다.
            //초당 2번 씩 메시지를 출력한다.
            if(System.currentTimeMillis() >= nextPrintTime){
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) //잠시 대기
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() //job을 취소하고 완료되기를 기다림.
    println("main: Now I can quit")
}

job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm sleeping 3 ...
job: I'm sleeping 4 ...
main: Now I can quit
```

### Making computation code cancellable

연산 코드를 취소할 수 있게 만드는 방법에는 2가지가 있다.

* 취소를 확인하는 정지 기능을 주기적으로 발동 시키는 것. 이를 위한 좋은 선택지인 yield() 함수가 있다.
  * yield() 함수는 코루틴을 취소 요청에 친화적인 코드로 만들기 위해서 취소가 가능한 시점마다 다른 Continuation에 실행시간을 양보해주는 기능을 한다.
* 취소 상태를 명시적으로 확인하는 것.
  * CoroutineScope에 정의된 isActive 속성을 참조해서 코루틴이 비활성화(Inactive) 상태인 경우 작업을 중단하는 것.

```kotlin
//yield 함수 이용

fun main(args: Array<String>) = runBlocking{
	val job = launch(Dispatchers.Default) {
		for(i in 1..10){
			yield()
			println("Im sleeping $i ...")
			Thread.sleep(500L)
		}
	}

	delay(1300L)
	println("main: Im tired of waiting!")
	job.cancelAndJoin()
	println("main: Now I can quit.")
}

Im sleeping 1 ...
Im sleeping 2 ...
Im sleeping 3 ...
main: Im tired of waiting!
main: Now I can quit.
```

```kotlin
//CoroutineScope에 정의된 isActive 속성을 참조하는 방법

fun main() = runBlocking{
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default){
        var nextPrintTime = startTime
        var i = 0
        while(isActive){ //취소 가능한 연산 루프
            if(System.currentTimeMillis() >= nextPrintTime){
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) //잠시 대기
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() //job을 취소하고 완료되기를 기다림.
    println("main: Now I can quit")
}

job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit
```

`isActive`는 `CoroutineScope` 객체를 통해 코루틴 내부에서 사용할 수 있는 익스텐션 프로퍼티이다.

* isActive함수는 현재 job이 실행중 일때 true를 반환한다.(아직 완료가 안됬거나, 취소를 하기 전)

### Closing resources with finally

Cancellable한 suspend 함수는 취소 시 `CancellationException` 을 던지므로 통상적인 방법으로 처리할 수 있다. 예를 들어 `try {...} finally {...}` 이나 `use` 함수는 코루틴이 취소될 때 정상적으로 최종 작업을 진행한다. 또는 Kotlin use() 함수를 사용하는 방식도 있다.

```kotlin
fun main() = runBlocking{
    val job = launch {
        try {
            repeat(1000){ i ->
                println("job: I'm sleeping $i")
                delay(500L)
            }
        } finally {
            println("job: I'm running finally")
        }
    }
    delay(1300L)
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}

job: I'm sleeping 0
job: I'm sleeping 1
job: I'm sleeping 2
main: I'm tired of waiting!
job: I'm running finally
main: Now I can quit.
```

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = launch {
        SleepingBed().use {
            it.sleep(1000)
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}

class SleepingBed : Closeable {
    suspend fun sleep(times: Int) {
        repeat(times) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    override fun close() {
        println("main : I'm running close() in SleepingBed!")
    }

}
```

### Run non-cancellable block

위 예제 코드에서 실행하는 코루틴이 취소되기 때문에 `finally` 블록에서 `suspend` 함수를 사용하려고 하면 `CancellationException` 이 발생한다. 즉, 이미 `CancellableException`이 발생한 코루틴의 finally 블록 안에서 중단 함수를 호출하면 현재 코루틴은 이미 취소된 상태이기 때문에 `CancellationException` 이 발생한다.

보통 리소스를 정리하는 함수들은(파일 닫기, 작업 취소, 통신 채널의 닫기) 넌-블럭킹(Non-Blocking)으로 동작하기 때문에 이러한 제약이 큰 문제가 되지 않는다. 하지만 이미 취소된 코루틴 안에서 동기적으로 어떤 중단 함수(suspend)를 호출해야 하는 상황이라면 `withContext{ }` 코루틴 빌더에 `NonCancellable` 컨텍스트를 전달하여 이를 처리할 수 있다.

```kotlin
fun main() = runBlocking{
    val job = launch {
        try {
            repeat(1000){ i ->
                println("job: I'm sleeping $i")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable){
                println("job: I'm running finally")
                delay(1000L)
                println("job: And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L)
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}

job: I'm sleeping 0
job: I'm sleeping 1
job: I'm sleeping 2
main: I'm tired of waiting!
job: I'm running finally
job: And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
```

### Timeout

코루틴의 실행을 취소하는 일반적인 이유는 코루틴 실행 시간이 일정 시간 제한을 초과했기 때문이다. 이 경우 타임아웃(Timeout)을 지정하고 이 시간을 넘어설 경우 해당 작업을 취소하도록 구현할 수 있다.

```kotlin
fun main() = runBlocking {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}

I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms
	at kotlinx.coroutines.TimeoutKt.TimeoutCancellationException(Timeout.kt:122)
	at kotlinx.coroutines.TimeoutCoroutine.run(Timeout.kt:88)
	at kotlinx.coroutines.EventLoopBase$DelayedRunnableTask.run(EventLoop.kt:316)
	at kotlinx.coroutines.EventLoopBase.processNextEvent(EventLoop.kt:123)
	at kotlinx.coroutines.DefaultExecutor.run(DefaultExecutor.kt:61)
	at java.base/java.lang.Thread.run(Thread.java:830)
```

`TimeoutCancellationException` 은 `withTimeout` 에서 던져지는 `CancellationException` 의 하위 클래스다. 실행결과에서 보면 해당 예외가 콘솔에 출력된 것을 볼 수 없을 것이다. 왜냐하면 취소된 코루틴의 `CancellationException` 내부에 존재하기 때문이다. 이 예외는 코루틴이 완료되었다는 정상적인 신호로 여겨진다. 하지만 위의 코드에서는 main 함수 안에서 `withTimeout` 이 사용되었다.

취소는 단순한 예외에 불과하기 때문에 모든 리소스는 일반적인 방법으로 닫히게 된다. 우리는 코드를 timeout과 함께 `try {...} catch (ex. TimeoutCancellationException {...})` 블록으로 감싸서 특정 타입의 시간 초과에 대해 특별한 추가적인 작업을 처리할 수도 있고, `withTimeout` 과 유사하지만 timeout시 null을 반환하는 `withTimeoutOrNull` 을 사용하여 처리할 수도 있다.

```kotlin
fun main() = runBlocking {
    val result = withTimeoutOrNull(1300L){
        repeat(1000) {i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" //이 라인이 실행되기 전에 취소될 것이다.
    }
    println("Result is $result")
}

I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```
