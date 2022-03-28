# 처음 만나는 코루틴

### ex 1. 간단한 코루틴

코루틴을 만드는 가장 간단한 함수는 `runBlocking` 입니다. 이렇게 코루틴을 만드는 함수를 코루틴 빌더라 합니다. `runBlocking`은 코루틴을 만들고 코드 블록이 수행이 끝날 때까지 다음의 코드를 수행하지 못하게 막습니다. 이름 그대로 블로킹(blocking)입니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println(Thread.currentThread().name)
    println("Hello")
}

//main @coroutine#1
//Hello
```

### ex 2. 코루틴 빌더의 수신 객체

`runBlocking` 안에서 `this` 를 수행하면 코루틴의 수신 객체(Receiver)인 것을 알 수 있습니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println(this)
    println(Thread.currentThread().name)
    println("Hello")
}

//"coroutine#1":BlockingCoroutine{Active}@36d64342
//main @coroutine#1
//Hello
```

`BlockingCoroutine` 은 `CoroutineScope` 의 자식입니다. 코루틴을 쓰는 모든 곳에는 코루틴 스코프(CoroutineScope)가 있습니다.

\<aside> 💡 Coroutine을 접근하는 방법은 두 가지 방법이 있습니다. 코루틴을 활용하는데 있어 `CoroutineScope`이 interface로 정의되어 있습니다. 이 interface는 정의를 통해 매번 원하는 형태의 `CoroutineContext`를 정의할 수 있고, Coroutines의 수명 주기를 관리할 수 있습니다. 애플리케이션이 동작하는 동안 별도의 수명 주기를 관리하지 않고 사용할 수 있는 `GlobalScope`가 있습니다. 이는 안드로이드 앱이 처음 시작부터 종료할 때까지 하나의 `CoroutineContext` 안에서 동작하도록 할 수 있습니다.

\</aside>

### ex 3. 코루틴 컨텍스트

코루틴 스코프는 코루틴을 제대로 처리하기 위한 정보, 즉 코루틴 컨텍스트(CoroutineContext)를 가지고 있습니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println(coroutineContext)
    println(Thread.currentThread().name)
    println("Hello")
}

//[CoroutineId(1), "coroutine#1":BlockingCoroutine{Active}@340f438e, BlockingEventLoop@30c7da1e]
//main @coroutine#1
//Hello
```

### ex 4. launch 코루틴 빌더

`launch` 는 “할 수 있다면 다른 코루틴 코드를 같이 수행”의 의미를 내포하는 코루틴 빌더입니다.

새로운 코루틴을 만들기 때문에, 새로운 코루틴 스코프를 생성합니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        println("launch: ${Thread.currentThread().name}")
        println("World!")
    }
    println("runBlocking: ${Thread.currentThread().name}")
    println("Hello")
}

/*
runBlocking: main @coroutine#1
Hello
launch: main @coroutine#2
World!
*/
```

`launch` 빌더 안에 있는 코드가 `runBlocking` 보다 늦게 수행된 것을 확인할 수 있습니다.

이는 두 개의 빌더 모두 메인 스레드를 사용하기 때문에, `runBlocking`의 코드가 메인 스레드를 모두 사용할 때까지 `launch`의 코드 블록을 대기하는 것입니다.

### ex 5. delay 함수

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        println("launch: ${Thread.currentThread().name}")
        println("World!")
    }
    println("runBlocking: ${Thread.currentThread().name}")
    delay(500L)
    println("Hello")
}

/*
runBlocking: main @coroutine#1
launch: main @coroutine#2
World!
Hello
*/
```

‘runBlocking: main @coroutine#1’ 가 호출된 이후 `delay`가 호출됩니다. 이때 `runBlocking`의 코루틴은 대기 시간동안 대기하며 `launch`의 코드 블록이 먼저 수행됩니다.

### ex 6. 코루틴 내에서 sleep

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        println("launch: ${Thread.currentThread().name}")
        println("World!")
    }
    println("runBlocking: ${Thread.currentThread().name}")
    Thread.sleep(500)
    println("Hello")
}

/*
runBlocking: main @coroutine#1
Hello
launch: main @coroutine#2
World!
*/
```

예상한 결과는 launch 블록이 실행되어야 하는데 이처럼 나오지 않을 것을 확인할 수 있습니다. Thread.sleep 은 코루틴이 아무 일을 하지 않는 동안에도 스레드를 독점하고 있습니다.

즉, 코루틴 내부의 Thread.sleep은 현재 스레드와 코루틴에 영향을 주고, delay는 코루틴에만 영향을 준다고 생각하면 됩니다.

```kotlin
CoroutineScope(Main).launch {
    start = System.currentTimeMillis()
    println("[D] start")
    val Job1 = launch {
        job1_start = System.currentTimeMillis()
        delay(1000)
        print("[D] working1 ")
        println("time : ${System.currentTimeMillis() - job1_start}")
    }

    val Job2 = launch {
        job2_start = System.currentTimeMillis()
        Thread.sleep(1000)
        print("[D] working2 ")
        println("time : ${System.currentTimeMillis() - job2_start}")
    }

    val Job3 = launch {
        job3_start = System.currentTimeMillis()
        delay(2000)
        print("[D] working3 ")
        println("time : ${System.currentTimeMillis() - job3_start}")
    }

}.invokeOnCompletion {
    print("[D] all finish ")
    println("time : ${System.currentTimeMillis() - start}")
}

/*
[2021–02–21 23:23:31.424] [D] start
[2021–02–21 23:23:32.483] [D] working2 time : 1001 -> thead 독점하고 있어 이미 시간이 흐름
[2021–02–21 23:23:33.483] [D] working1 time : 2005 -> 그 후 코루틴 실행
[2021–02–21 23:23:35.489] [D] working3 time : 3004
[2021–02–21 23:23:35.491] [D] all finish time : 4067
*/
```

### ex 7. 한번에 여러 launch

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        println("launch1: ${Thread.currentThread().name}")
        delay(1000L)
        println("3!")
    }
    launch {
        println("launch2: ${Thread.currentThread().name}")
        println("1!")
    }
    println("runBlocking: ${Thread.currentThread().name}")
    delay(500L)
    println("2!")
}

/*
runBlocking: main @coroutine#1
launch1: main @coroutine#2
launch2: main @coroutine#3
1!
2!
3!
*/
```

delay 값을 바꿔보면 `suspend` 이후 깨어나느 순서에 따라 출력 결과가 달라질 것입니다.

### ex 8. 상위 코루틴은 하위 코루틴을 끝까지 책임진다.

`runBlocking` 안에 두 개의 `launch` 빌더가 계층화되어 있어 구조적인 것을 확인할 수 있습니다.

`runBlocking` 은 그 속에 포함된 `launch` 가 다 끝나기 전까지 종료되지 않습니다.

```kotlin
import kotlinx.coroutines.*

fun main() {
    runBlocking {
        launch {
            println("launch1: ${Thread.currentThread().name}")
            delay(1000L)
            println("3!")
        }
        launch {
            println("launch2: ${Thread.currentThread().name}")
            println("1!")
        }
        println("runBlocking: ${Thread.currentThread().name}")
        delay(500L)
        println("2!")
    }
    print("4!")
}

/*
runBlocking: main @coroutine#1
launch1: main @coroutine#2
launch2: main @coroutine#3
1!
2!
3!
4!
*/
```

### ex 9. suspend 함수

앞서 살펴본 `delay`, `launch` 등과 같은 함수는 코루틴 내에서만 호출할 수 있었습니다. 그럼 이 함수들을 포함한 코드들을 어떻게 함수로 분리하는지 알아봅시다.

코드의 일부를 함수로 분리할 때는 함수의 앞에 `suspend` 키워드를 붙이면 됩니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doThree() {
    println("launch1: ${Thread.currentThread().name}")
    delay(1000L)
    println("3!")
}

suspend fun doOne() {
    println("launch1: ${Thread.currentThread().name}")
    println("1!")
}

suspend fun doTwo() {
    println("runBlocking: ${Thread.currentThread().name}")
    delay(500L)
    println("2!")
}

fun main() = runBlocking {
    launch {
        doThree()
    }
    launch {
        doOne()
    }
    doTwo()
}

/*
runBlocking: main @coroutine#1
launch1: main @coroutine#2
launch1: main @coroutine#3
1!
2!
3!
*/
```

doOne 함수는 `suspend` 함수를 호출하지 않았기 때문에 `suspend` 키워드를 붙이지 않은 일반 함수로 지정해도 됩니다.

만약 `suspend` 함수를 다른 함수에서 호출하려면, 그 함수가 `suspend` 함수이거나 코루틴 빌더를 통해 코루틴을 만들어야 가능합니다.

***

### RFC

_suspend 함수는 어떻게 작동하는가? - 내부 동작_

[https://www.notion.so/Suspend-2bc82f1b4e914a16a3005d9f78fe8e09](https://www.notion.so/Suspend-2bc82f1b4e914a16a3005d9f78fe8e09)
