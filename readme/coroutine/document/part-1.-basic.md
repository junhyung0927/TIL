# part 1. basic

```kotlin
fun main(args: Array<String>) {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    Thread.sleep(2000L)
}
```

* GlobalScope.launch{} 코드 블록은 코루틴을 생성하기 위한 코루틴 빌더이며 이렇게 생성되어 실행되는 코루틴은 호출(실행) 스레드를 블록하지 않기 때문에 그대로 두면 메인 함수가 종료되고 메인 함수를 실행한 메인 스레드 역시 종료되어 프로그램이 끝나게 된다.
* 이를 방지하기 위해 임의의 시간을 지정하여 지연시키는 것이다. 이렇게 스레드를 멈추는 역할을 수행하는 함수를 Blocking Function이라 한다.

```kotlin
fun main(args: Array<String>) {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    runBlocking {
        delay(2000L)
    }
}
```

* 중단 함수가 현재 스레드를 멈추게 할 수 있다는 것을 코드에 명시적으로 나타내기 위해 runBlocking{} 블록을 사용할 수 있다.
* runBlocking{} 블록은 주어진 블록이 완료될 때까지 현재 스레드를 멈추는 새로운 코루틴을 생성하여 실행하는 코루틴 빌더이다.
* 코루틴 안에서는 runBlocking{} 의 사용은 권장되지 않으며, 일반적인 함수 코드 블록에서 중단 함수를 호출할 수 있도록 하기 위해서 존재하는 장치이다. => delay() 중단 함수를 사용하기 위해 쓰임

```kotlin
fun main(args: Array<String>) = runBlocking{
  GlobalScope.launch{
    delay(1000L)
    println("World")
  }
  println("Hello,")
  delay(2000L)
}
```

* 메인 함수 자체를 runBlocking{} 코루틴 빌더를 이용해서 작성하면 지연을 위한 delay() 중단 함수의 사용이 보다 자연스러워진다.
* delay()는 중단 함수이며 모든 중단 함수들은 코루틴 안에서만 호출될 수 있다는 제약이 존재한다. GlobalScope.launch{} 코드 블록에서 delay(1000L)을 사용할 수 있었던 이유는 GlobalScope.launch{}가 주어진 코드블록을 수행하는 코루틴을 생성하는 코루틴 빌더이기 때문이다. 그리고 해당 코드 블록은 코루틴 안에서 수행되고 있기 때문이다.
* 위의 모든 예제에서 GlobalScope.launch{} 로 실행 된 코루틴의 수행이 완료될 때 까지 현재 스레드(main 함수)를 대기시키기 위해서 임의의 지연을 주었는데, 실제 프로그램에서는 적절치 못한 방법이다.
* 내부적으로 실행중인 코루틴(자식 코루틴)들이 작업을 완료하고 종료 될 때까지 얼마나 대기해야 할 지 부모 코루틴은 예측할 수 없기 때문이다.

```kotlin
fun main(args: Array<String>) = runBlocking{
  val job = GlobalScope.launch{
    delay(1000L)
    println("GlobalScope")
  }

  println("Main")

  job.join()
}
```

* 위의 문제를 해결하기 위해서 GlobalScope.launch{}의 결과로 반환되는 Job 인스턴스를 사용할 수 있다. 메인 코루틴은 GlobalScope.launch{} 빌더를 이용해 생성한 코루틴이 종료될 때까지 대기한 후 종료된다. 이는 자식 코루틴의 실행 흐름에 연결됨으로써 가능하다.

```kotlin
job.join()
```

* 만약 메인 코루틴 안에서 여러 개의(2개 이상)인 자식 코루틴들이 수행되고, 모든 자식 코루틴들의 종료를 기다리게 구현한다면??

⇒ 자식 코루틴에 대응되는 Job 객체들의 참조를 어딘가에 유지하고 있다가 부모 코루틴이 종료되어야 하는 시점에 실행된 모든 자식 코루틴들의 Job 객체들에 join하여 자식 코루틴들의 종료를 기다려야 할 것이다.

```kotlin
fun main(args: Array<String>) = runBlocking{
	launch{
		delay(1000L)
		println("World!")
	}
	println("Hello,")
}
```

* 위의 문제를 해결하기 위한 방법은 코루틴 스코프를 이용하는 것이다.
* 모든 코루틴은 각자의 스코프를 가진다. runBlocking{} 코루틴 빌더 등을 이용해 생성된 코루틴 블록 안에서 launch{} 코루틴 빌더를 이용하여 새로운 코루틴을 생성한다. 그러면 현재 위치한 부모 코루틴에 join()을 명시적으로 호출할 필요 없이 자식 코루틴들을 실행하고 종료될 때까지 대기 할 수 있다.

#### Scope builder

* 어떤 코루틴들을 위한 사용자 정의 스코프가 필요한 경우가 있다면 coroutineScope{ } 빌더를 이용할 수 있다.
* coroutineScope{} 빌더를 통해 생성된 코루틴은 모든 자식 코루틴들이 끝날때까지 종료되지 않는 스코프를 정의하는 코루틴이다.
* runBlocking vs coroutineScope ⇒ 이해가 잘 안감.

```kotlin
fun main(args: Array<String>) = runBlocking{
	launch{
		delay(200L)
		println("Task from runBlocking")
	}

	coroutineScope{
		launch{
			delay(500L)
			println("Task from nested launch")
		}
		delay(100L)
		println("Task from coroutine scope")
	}

	println("Coroutine scope is over")
}

//Task from coroutine scope
//Task from runBlocking
//Task from nested launch
//Coroutine scope is over
```

***

#### Extract function refactoring

* 메인 함수 블록 안에 모든 로직을 기술하는 것은 샘플을 위한 코드일 뿐이지 비효율적이다. 이것들을 용도에 맞는 개별 함수로 분리하여 보다 실용적으로 사용할 수 있는 패턴으로 변경해야 한다.

```kotlin
fun main(args: Array<String>) = runBlocking{
	launch{
		doWorld()
	}
	println("Hello,")
}

suspend fun doWorld(){
	delay(1000L)
	println("World!")
}
```

* 코루틴 내부에서 실행되는 중단 함수들은 suspend 키워드를 함수명 앞에 붙여 만들 수 있다. 이 함수들은 일반 함수들과 다르게 delay()와 같은 다른 중단 함수들을 호출 할 수 있다.
  * suspend 키워드를 붙여 만든 함수들 역시 중단 함수이기 때문에 특정 코루틴 컨텍스트 안에서 수행되고, 코루틴 컨텍스트 안에서는 모든 중단 함수를 호출 할 수 있기 때문이다.
* 만약 중단 함수가 현재 스코프에서 수행 될 코루틴 빌더를 포함한다면?
  * 위 예제에서 doWorld() 함수를 CoroutineScope의 확장 함수로 만드는 방법도 있겠지만, 이는 API를 불명확하게 만든다.
  * 명시적으로 CoroutineScope을 필드로 갖는 클래스를 만들고 그 클래스가 해당 suspend 함수를 갖게 하는 방법.
  * 외부(outer) 클래스의 구현을 암시적으로 사용하는 방법.
  * CoroutineScope(coroutineContext) 자체를 생성하여 사용하는 방법은 구조적으로 안전하지 않다. 이 방식을 사용하게 되면 코드 개발자(?)는 더 이상 이 메서드가 실행되는 스코프를 컨트롤 할 방법이 없어지기 때문이다. private API 만이 이 빌더를 사용할 수 있다.

#### Coroutines are light-weight

* 일반적인 스레드 구현으로는 메모리 부족 오류가 발생할 수 있다. 이를 코루틴으로 작성하면 정상적으로 동작한다.

```kotlin
fun main(args:Array<String>) = runBlocking{
	repeat(100_000){
		launch{
			delay(1000L)
			print(".")
		}
	}
}
```

* 위 예제는 십만개의 코루틴을 수행하고 1초 후에 각각의 코루틴들은 점을 출력한다. 이를 스레드로 구현하여 수행한다면 메모리 부족 예외를 발생 시켰을 것이다.

#### Global coroutines are like daemon threads

```kotlin
fun main(args:Array<String>) = runBlocking{
	GlobalScope.launch{
		repeat(1000) { i -> 
			println("I'm sleeping $i ...")
			delay(500L)
		}
	}
	delay(1300L)
}

//I'm sleeping 1 ...
//I'm sleeping 2 ...
//I'm sleeping 3 ...
```

* 위 예제는 500ms 간격으로 천번 출력하는 코루틴 빌더다. 이렇게 무거운(오래걸리는) 코루틴이 수행되는 동안 메인 함수는 보다 짧은 시간을 대기한 후 종료한다.
* GlobalScope에서 실행된 코루틴은 마치 데몬 스레드와 같이 자신이 속한 프로세스의 종료를 지연시키지 않고 프로세스 종료 시 함께 종료되기 때문에 다음과 같이 허용된 시간 동안만 동작한 결과를 출력한다.
