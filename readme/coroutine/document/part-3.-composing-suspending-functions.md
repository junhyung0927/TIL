# part 3. Composing Suspending Functions

### 코루틴 Suspend 함수의 구성

어떤 로직을 수행하는 두 개의 중단 함수가 다음과 같은 이름으로 정의되어 있다고 가정한다.

* doSomethingUsefulOne
* doSomethingUsefulTwo

중요한 로직을 맡고 있는 기능이라면 원격 서비스의 호출이나 복잡한 계산을 수행하는 것들을 예로 들 수 있다. 여기서의 2개의 함수는 예제를 위해 일정시간 delay() 함수를 호출해서 지연하는 기능만 있는 함수일 뿐이다.

이 함수들을 순차적으로 호출해야 한다면 , 첫번째 함수의 결과를 이용해 두번째 함수를 호출하거나 그 호출방식을 결정해야 할 경우 아래와 같은 상황이 발생한다.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}

//The answer is 42
//Completed in 2026ms
```

위 예제를 보면 일반적인 함수처럼 순차적으로 중단 함수들을 호출했다. 이것이 가능한 이유는 코루틴에서 기본적으로 이 중단 함수들이 순차적으로 수행되기 때문이다.

결과를 보면 총 소요시간이 함수가 순차적으로 수행되었기 떄문에 거의 중단 함수 각각의 수행시간의 합 만큼 시간이 소모되었다는 것을 볼 수 있다.

### aync를 이용한 동시 수행(Concurrent using async)

위의 예로 들었던 두 중단함수가 서로 의존성이 없고 둘 중 아무것이나 빠른 쪽의 결과를 먼저 수신하고자 한다면 async{ } 빌더를 사용할 수 있다.

async{ }는 launch{ }와 동일한 컨셉이지만 다른 코루틴들과 동시에 수행되는 경량의 스레드인 별도 코루틴을 시작시킨다는 점에서 차이점이 있다. 그리고 launch{ }는 Job 오브젝트를 반환하고 어떠한 결과값도 전달해주지 않지만 async는 경량의 넌블럭킹 퓨처인 Deferred\<T>를 반환한다(결과를 나중에 제공하는 Promise).

지연값(Deferred value)에 .await() 함수를 이용해서 결과를 얻을 수 있고, Job 타입이기 떄문에 이를 이용해서 필요한 경우 취소를 요청할 수 도 있다.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}

//The answer is 42
//Completed in 1072ms
```

위 예제는 async를 사용하지 않은 경우에 비해서 2배 빠르다. 그 이유는 두 코루틴을 동시에 수행했기 떄문이다. 코루틴 프레임워크에서는 순차 실행이 기본이기 때문에 동시 수행은 항상 **명시적** 이어야 한다.

### async의 지연 실행(Lazily started async)

async는 start 파라미터에 [CoroutineStart](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/).LAZY 값을 전달해서 실행을 지연시킬 수 있다. 이 옵션이 적용된 async 코루틴은 결과 값이 필요한 시점에 await() 이나 start() 함수가 호출되는 시점에 시작된다.

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }

        //If comment out below two lines, two coroutines will be called sequentially.
        one.start()
        two.start()

        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}

//The answer is 42
//Completed in 1017 ms
```

위 예제를 보면 두 개의 코루틴을 선언하면서 실행 옵션은 CoroutineStart.LAZY를 전달해서 실행 시점을 값이 필요한 시점으로 변경하고 있다. 개발자는 원하는 시점에 start() 함수를 호출해서 해당 코루틴을 실행 시킬 수 있다. 위의 예에서는 두 개의 코루틴을 시작시키고 그 결과를 출력하기 위해서 출력문에서 대기하게 된다.

만약 one.start(), two.start()의 라인을 실행하지 않는다면, 출력문의 one.await()이 해당 코루틴을 시작시키고 그 결과를 받고 나서 다음 two.await()이 호출되기 때문에 결국 순차적으로 수행 했을 때와 같은 2초정도의 시간이 걸리게 된다.

### 비동기 스타일 함수(Async-style functions)

위의 예제에서와 같이 doSomethingUsefulOne(), doSomethingUsefulTwo() 함수들을 호출하는 비동기 스타일의 함수를 정의할 수 있다(async [wrapper](http://www.tcpschool.com/java/java\_api\_wrapper)). 이것은 명시적으로 GlobalScope의 async{ } 코루틴 빌더를 통해 구현이 가능하다. 그리고 이 함수들이 비동기 연산을 시작시킬 것이고 그 결과를 얻기 위해서는 지연 값(Deferred\<T>)을 사용해야 함을 명시적으로 나타내기 위해서 관례적으로 함수명 뒤에 Async를 붙인다.

```kotlin
// The result type of somethingUsefulOneAsync is Deferred<Int>
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// The result type of somethingUsefulTwoAsync is Deferred<Int>
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

xxxAsync 함수들은 중단함수들이 아니다. 그러므로 어디서든 호출 될 수 있다. 하지만 이 함수들의 사용은 항상 코드를 실행함으로써 비동기로 실행(동시 실행)됨을 내포하고 있다.

다음 예제는 코루틴이 아닌 곳에서 이 함수들의 사용을 보여주고 있다.

```kotlin
fun main(args: Array<String>) {
    val time = measureTimeMillis {
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()

        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}

fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```

> 위와 같은 비동기 스타일 함수의 사용을 예제로 제공한 이유는 다른 프로그래밍 언어들에서 보편적으로 사용되는 스타일이기 때문이다. 이러한 방식은 코루틴을 이용한 비동기 프로그래밍에서는 권장되지 않는다. 그 이유는 에러 처리와 관련이 있는데, 만약 xxxAsync 함수 호출과 await() 사이에 오류가 발생한다면 xxxAsync 함수는 오류와 관계없이 계속 수행되는 문제가 있을 것이기 때문이다.

### async를 이용한 구조화된 동시성(Structured Concurrency with async)

위의 예의 동시에 수행되는 함수인 doSomethingUsefulOne()와 doSomethingUsefulTwo()를 이용해서 그 합을 결과로 반환하는 예제를 살펴보자.

async{ } 코루틴 빌더는 CoroutineScope의 확장 함수로 정의되어 있기 때문에, 이 빌더를 코루틴 스코프 안에서 호출할 수 있고 이것은 coroutineScope() 함수가 제공한다.

```kotlin
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        println("The answer is ${concurrentSum()}")
    }
    println("Completed in $time ms")
}

suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}

fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

//The answer is 42
//Completed in 1016 ms
```

위 예제에서와 같이 구조화된 동시성 모델을 유지하면 일관된 에러 처리를 할 수 있으며, 에러의 전파도 정확하게 동작한다.

취소(Cancellation)도 코루틴 계층을 따라 전파된다.

```kotlin
fun main(args: Array<String>) = runBlocking {
    try {
        val time = measureTimeMillis {
            println("The answer is ${failedConcurrentSum()}")
        }
        println("Completed in $time ms")
    } catch (throwable: Throwable) {
        println("Computation failed with $throwable")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async {

        try {
            delay(Long.MAX_VALUE) // emulates very long computation
            doSomethingUsefulOne()
        } finally {
            println("First child was canceled.")
        }
    }
    val two = async<Int> {
        println("Second child throw an exception.")
        doSomethingUsefulTwo()
        throw ArithmeticException("Exception on purpose.")
    }
    one.await() + two.await()
}

fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}

//Second child throw an exception.
//First child was canceled.
//Computation failed with java.lang.ArithmeticException: Exception on purpose.
```

결과를 보면 알 수 있듯이, two{ } 비동기 코루틴이 실패함에 따라서 긴 작업을 수행하고 있던 one{ } 비동기 코루틴이 취소되고, 이를 호출한 메인 함수로 예외가 던져진다.
