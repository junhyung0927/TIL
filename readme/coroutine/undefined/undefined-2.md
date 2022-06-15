# 취소와 타임아웃

### ex 14. Job에 대해 취소

`Job` 을 사용해서 `cancel` 메서드를 호출하여 코루틴을 취소할 수 있다.

`Job` 은 N 개의 코루틴의 동작을 제어할 수 있으며, 하나의 코루틴 동작을 제어할 수도 있다.

코루틴의 `Job` 은 아래 표와 같이 여섯 가지 코루틴 상태를 포함하며, actvie/completed/canceled 상태에 따라 값이 변동한다.

![](<../../../.gitbook/assets/Untitled (28).png>)

이러한 `Job`을 바탕으로 코루틴의 상태를 확인할 수 있고, 제어할 수 있다. job.cancel()을 호출하게 되면 즉시 취소 상태로 `Job` 을 한다.

```
                                      `wait children                       
+-----+ start  +--------+ complete   +-------------+  finish  +-----------+
| New | -----> | Active | ---------> | Completing  | -------> | Completed |
+-----+        +--------+            +-------------+          +-----------+
                 |  cancel / fail       |                                  
                 |     +----------------+                                  
                 |     |                                                   
                 V     V                                                   
             +------------+                           finish  +-----------+
             | Cancelling | --------------------------------> | Cancelled |
             +------------+                                   +-----------+`
```

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() = coroutineScope {
    val job1 = launch {
        println("launch1: ${Thread.currentThread().name}")
        delay(1000L)
        println("3!")
    }

    val job2 = launch {
        println("launch2: ${Thread.currentThread().name}")
        println("1!")
    }

    val job3 = launch {
        println("launch3: ${Thread.currentThread().name}")
        delay(500L)
        println("2!")  
    }

    delay(800L)
    job1.cancel()
    job2.cancel()
    job3.cancel()
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}

/*
launch1: main @coroutine#2
launch2: main @coroutine#3
1!
launch3: main @coroutine#4
2!
4!
runBlocking: main @coroutine#1
5!
*/
```



#### Job으로 할 수 있는 동작들

* `start` : 현재의 코루틴의 동작 상태를 체크하며, 동작 중인 경우 true, 준비 또는 완료 상태이면 false를 리턴함
* `join` : 현재의 코루틴 동작이 끝날 때까지 대기함. 즉, async { } await 처럼 사용 가능함
* `cancel` : 현재 코루틴을 즉시 종료하도록 유도만 하고 대기하지 않음
* `cancelAndJoin` : 현재 코루틴에 종료하라는 신호를 보내고, 정상 종료할 때까지 대기함
* `cancelChildren` : 코루틴스코프 내에 작성한 자식 코루틴들을 종료함. `cancel` 과 다르게 하위 아이템들만 종료하며, 부모는 취소하지 않음

### ex 15. 취소 불가능한 Job

`launch` 블록에 `Dispatchers`를 이용해서 `Default` 스레드를 지정해서 수행을 시켜봅시다.

```kotlin
import kotlinx.coroutines.*

suspend fun doCount() = coroutineScope {
    val job1 = launch(Dispatchers.Default) {
        var i = 1
        var nextTime = System.currentTimeMillis() + 100L

        while (i <= 10) {
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextTime) {
                println(i)
                nextTime = currentTime + 100L
                i++
            }
        }
    }
    
    delay(200L)
    job1.cancel()
    println("doCount Done!")
}

fun main() = runBlocking {
    doCount()
}

/*
1
doCount Done!
2
3
4
5
6
7
8
9
10
*/
```

두 가지 부분이 신경이 쓰일 것입니다.

1. `job1` 이 취소 혹은 종료가 되었든 다 끝난 이후에 `doCount Done` 을 출력하고 싶음
2. 취소가 되지 않았음

### ex 16. cancel과 join

```kotlin
import kotlinx.coroutines.*

suspend fun doCount() = coroutineScope {
    val job1 = launch(Dispatchers.Default) {
        var i = 1
        var nextTime = System.currentTimeMillis() + 100L

        while (i <= 10) {
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextTime) {
                println(i)
                nextTime = currentTime + 100L
                i++
            }
        }
    }
    
    delay(200L)
    job1.cancel()
    job1.join()
    println("doCount Done!")
}

fun main() = runBlocking {
    doCount()
}

/*
**1
2
3
4
5
6
7
8
9
10
doCount Done!**
*/
```

`cancel` 이후에 `join` 을 호출하여, 실제 코루틴 동작을 다 끝낼때까지 대기한 후 종료하는 것을 확인할 수 있습니다.

### ex 17. cancelAndJoin

`cancel` 과 `join` 의 역할을 한 번에 해주는 `cancelAndJoin` 을 사용해봅시다.

```kotlin
import kotlinx.coroutines.*

suspend fun doCount() = coroutineScope {
    val job1 = launch(Dispatchers.Default) {
        var i = 1
        var nextTime = System.currentTimeMillis() + 100L

        while (i <= 10) {
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextTime) {
                println(i)
                nextTime = currentTime + 100L
                i++
            }
        }
    }
    
    delay(200L)
    job1.cancelAndJoin()
    println("doCount Done!")
}

fun main() = runBlocking {
    doCount()
}

/*
1
2
3
4
5
6
7
8
9
10
doCount Done!
*/
```

### ex 18. cancel 가능한 코루틴

`isActive` 를 호출하면 코루틴의 활성화 여부를 확인할 수 있습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doCount() = coroutineScope {
    val job1 = launch(Dispatchers.Default) {
        var i = 1
        var nextTime = System.currentTimeMillis() + 100L

        while (i <= 10 && isActive) {
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextTime) {
                println(i)
                nextTime = currentTime + 100L
                i++
            }
        }
    }
    
    delay(200L)
    job1.cancelAndJoin()
    println("doCount Done!")
}

fun main() = runBlocking {
    doCount()
}

/*
1
doCount Done!
*/
```

### ex 19. finally를 같이 사용

`launch` 빌더에서 자원을 할당한 경우에는 어떻게 정리해야 할까요?

`suspend` 함수들은 `JobCancellationException` 를 발생하기 때문에 try / catch / finally 로 대응할 수 있습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() = coroutineScope {
    val job1 = launch {
        try {
            println("launch1: ${Thread.currentThread().name}")
            delay(1000L)
            println("3!")
        } finally {
            println("job1 is finishing!")
        }
    }

    val job2 = launch {
        try {
            println("launch2: ${Thread.currentThread().name}")
            delay(1000L)
            println("1!")
        } finally {
            println("job2 is finishing!")
        }
    }

    val job3 = launch {
        try {
            println("launch3: ${Thread.currentThread().name}")
            delay(1000L)
            println("2!")
        } finally {
            println("job3 is finishing!")
        }
    }

    delay(800L)
    job1.cancel()
    job2.cancel()
    job3.cancel()
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}

/*
launch1: main @coroutine#2
launch2: main @coroutine#3
launch3: main @coroutine#4
4!
job1 is finishing!
job2 is finishing!
job3 is finishing!
runBlocking: main @coroutine#1
5!
*/
```

### ex 20. 취소 불가능한 블록

#### withContext

지정된 코루틴 컨텍스트를 사용해서 파라미터인 `suspend` 함수를 호출하고 완료될 때까지 일시 정지한 후 결과를 반환합니다.

`withContext`를 사용하면 코루틴을 실행하는 동안 원하는 코루틴 컨텍스트로 변경할 수 있습니다. 즉, 메인 스레드에서 실행한 코루틴을 다른 스레드로 이동하며 실행할 수 있습니다.

```kotlin
suspend operator fun invoke(): Result<R> {
	return try {
		withContext(Dispatcher.IO) {
			//NETWORK DO SOMETHING ...
		}
	} catch (e: Exception) {
		//Error Handling ...
	}
}
```

`invoke` 함수를 실행하면 네트워크와 관련된 작업 수행은 내부적으로 IO 스레드에서 수행될 것입니다.

```kotlin
uiScope.launch {
	//내부적으로 IO 스레드를 사용하여 네트워크 작업을 처리하지만, 메인 스레드에서 실행해도 문제가 되진 않습니다.
	viewModel.invoke() 
	
	successTask.isVisible = true
}
```

메인 스레드에서 호출하는 함수가 내부적으로는 IO 스레드에서 동작 됩니다. 즉, 메인 스레드에서 어떤 suspend 함수를 호출해도 무방합니다.

`withContext(NonCancellable)` 을 사용하면 취소 불가능한 블록을 만들 수 있습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doOneTwoThree() = coroutineScope {
    val job1 = launch {
        withContext(NonCancellable) {
            println("launch1: ${Thread.currentThread().name}")
            delay(1000L)
            println("3!")
        }
        delay(1000L)
        print("job1: end")
    }

    val job2 = launch {
        withContext(NonCancellable) {
            println("launch1: ${Thread.currentThread().name}")
            delay(1000L)
            println("1!")
        }
        delay(1000L)
        print("job2: end")
    }

    val job3 = launch {
        withContext(NonCancellable) {
            println("launch1: ${Thread.currentThread().name}")
            delay(1000L)
            println("2!")
        }
        delay(1000L)
        print("job3: end")
    }

    delay(800L)
    job1.cancel()
    job2.cancel()
    job3.cancel()
    println("4!")
}

fun main() = runBlocking {
    doOneTwoThree()
    println("runBlocking: ${Thread.currentThread().name}")
    println("5!")
}

/*
launch1: main @coroutine#2
launch1: main @coroutine#3
launch1: main @coroutine#4
4!
3!
1!
2!
runBlocking: main @coroutine#1
5!
*/
```

### ex 21. 타임 아웃

일정 시간이 끝난 후에 종료하고 싶다면 `withTimeout` 을 사용할 수 있습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doCount() = coroutineScope {
    val job1 = launch(Dispatchers.Default) {
        var i = 1
        var nextTime = System.currentTimeMillis() + 100L

        while (i <= 10 && isActive) {
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextTime) {
                println(i)
                nextTime = currentTime + 100L
                i++
            }
        }
    }
}

fun main() = runBlocking {
    withTimeout(500L) {
        doCount()
    }
}

/*
1
2
3
4
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 500 ms
 at (Coroutine boundary. (:-1) 
 at FileKt$main$1$1.invokeSuspend (File.kt:-1) 
 at FileKt$main$1.invokeSuspend (File.kt:-1) 
Caused by: kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 500 ms
at kotlinx.coroutines.TimeoutKt .TimeoutCancellationException(Timeout.kt:184)
at kotlinx.coroutines.TimeoutCoroutine .run(Timeout.kt:154)
at kotlinx.coroutines.EventLoopImplBase$DelayedRunnableTask .run(EventLoop.common.kt:502)
*/
```

일정 시간 후에 취소가 되면 `TimeoutCancellationException` 예외가 발생합니다.

### ex 22. withTimeoutOrNull

`withTimeoutOrNull` 을 이용해서 타임 아웃할 때 null을 반환하게 할 수 있습니다.

```kotlin
import kotlinx.coroutines.*

suspend fun doCount() = coroutineScope {
    val job1 = launch(Dispatchers.Default) {
        var i = 1
        var nextTime = System.currentTimeMillis() + 100L

        while (i <= 10 && isActive) {
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextTime) {
                println(i)
                nextTime = currentTime + 100L
                i++
            }
        }
    }
}

fun main() = runBlocking {
    val result = withTimeoutOrNull(500L) {
        doCount()
        true
    } ?: false
    println(result)
}

/*
1
2
3
4
false
*/
```



💡 **withContext와 coroutineScope는 무슨 차이점이 있을까?**

두 함수 모두 suspend 함수로서 코루틴 내부에서 블록을 중단시키기 때문에 유사해 보일 수 있습니다.

하지만, coroutineScope는 withCotext의 한 유형으로 볼 수 있습니다. 즉, coroutineScope는 withContext(this.coroutineContext)와 본질적으로 같은 의미를 가집니다.

coroutineScope는 무조건 현재 호출한 context를 사용하기 때문에, Dispatcher를 설정할 수 없습니다.

* coroutineScope는 에러 처리등의 목적으로 특정 코드를 하나의 블럭으로 묶고 싶을 때 사용
* withContext는 해당 코드 블럭을 특정 context에서 실행하고 싶을 때 사용하는 용도(ex: network)

### RFC

[Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/)

[Kotlin Coroutines의 Job 동작을 알아보자 |](https://thdev.tech/kotlin/2019/04/08/Init-Coroutines-Job/)

[취소와 타임아웃](https://dalinaum.github.io/coroutines-example/3)

[Coroutine - withContext 와 coroutineScope 의 비교](https://sandn.tistory.com/99)
