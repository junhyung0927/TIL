# 스코프 빌더, 잡

### ex 10. suspend 함수에서 코루틴 빌더 호출

코루틴 빌더는 항상 코루틴 스코프 내에서만 호출해야 합니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() {
    launch {
        println("launch1: ${Thread.currentThread().name}")
        delay(1000L)
        println("3!")
    }

    launch {
        println("launch2: ${Thread.currentThread().name}")
        println("1!")
    }

    launch {
        println("launch3: ${Thread.currentThread().name}")
        delay(500L)
        println("2!")  
    }
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}
```

### ex 11. 코루틴 스코프

`coroutineScope` 를 사용해서 코루틴 스코프를 만들 수 있습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() = coroutineScope {
    launch {
        println("launch1: ${Thread.currentThread().name}")
        delay(1000L)
        println("3!")
    }

    launch {
        println("launch2: ${Thread.currentThread().name}")
        println("1!")
    }

    launch {
        println("launch3: ${Thread.currentThread().name}")
        delay(500L)
        println("2!")  
    }
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}

/*
4!
launch1: main @coroutine#2
launch2: main @coroutine#3
1!
launch3: main @coroutine#4
2!
3!
runBlocking: main @coroutine#1
5!
*/
```

코루틴 스코프는 runBlocking과 차이점이 있습니다. `runBlocking` 는 현재 스레드를 멈추게 한 후 대기하지만, `coroutineScope` 는 현재 스레드를 멈추게 하지 않습니다.

호출한 쪽은 `suspend` 상태가 되고, 시간이 지나면 다시 활동하게 됩니다.

### ex 12. Job을 이용한 제어

코루틴 빌더 `launch` 는 Job 객체를 반환하며, 이를 통해 종료될 때까지 대기할 수 있습니다.

`launch` 빌더는 결과가 없는 코루틴을 생성하는 빌더입니다. 여기서 결과는 반환 인스턴스가 아닌 결과값(value)을 의미합니다.

반환하는 Job 인스턴스는 생성된 해당 코루틴을 제어하는데 사용합니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() = coroutineScope {
    val job = launch {
        println("launch1: ${Thread.currentThread().name}")
        delay(1000L)
        println("3!")
    }
    job.join()

    launch {
        println("launch2: ${Thread.currentThread().name}")
        println("1!")
    }

    launch {
        println("launch3: ${Thread.currentThread().name}")
        delay(500L)
        println("2!")  
    }
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}

/*
launch1: main @coroutine#2
3!
4!
launch2: main @coroutine#3
1!
launch3: main @coroutine#4
2!
runBlocking: main @coroutine#1
5!
*/
```

### ex 13. 가벼운 코루틴

코루틴은 협력적으로 동작하기 때문에 여러 개의 코루틴을 만드는 것에 큰 비용이 들지 않습니다.

10만 개 정도의 코루틴을 만드는 것도 큰 부담이 되지 않습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() = coroutineScope {
    val job = launch {
        println("launch1: ${Thread.currentThread().name}")
        delay(1000L)
        println("3!")
    }
    job.join()

    launch {
        println("launch2: ${Thread.currentThread().name}")
        println("1!")
    }

    repeat(1000) {
        launch {
            println("launch3: ${Thread.currentThread().name}")
            delay(500L)
            println("2!")  
        }
    }
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}
```

***

### RFC

[\[Kotlin\] 코루틴 #4 - CoroutineBuilder와 ScopeBuilder](https://jaejong.tistory.com/64)
