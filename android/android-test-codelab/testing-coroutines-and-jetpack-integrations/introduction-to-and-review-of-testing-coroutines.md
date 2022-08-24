# Introduction to and review of testing coroutines

## introduction to and review of testing coroutines

![](<../../../.gitbook/assets/Untitled (32).png>)

비동기 코드는 대부분 네트워크 또는 데이터베이스를 호출할 때 처럼 오래 걸리는 작업일 때 사용됩니다. 비동기 코드는 테스트하기 어려운 부분들이 있습니다.

* 비동기 코드는 비결론적(non-deterministic)인 경향이 있습니다. 예를 들어, A와 B 기능을 병렬적으로 실행되는 것을 테스트할 때 A가 먼저 끝날 수도 있고 B가 먼저 끝날 수도 있습니다. 이로 인해 테스트에 결함이 발생할 수 있습니다(일관하지 못한 결과가 있는 테스트)

![](<../../../.gitbook/assets/Untitled (33) (1).png>)

* 테스트 할 때, 비동기 코드에 대해서 동기화 메커니즘을 확인해야 하는 경우가 많습니다. 테스트는 테스팅 스레드에서 실행됩니다. 테스트를 다른 스레드에서 코드를 실행하거나 또는 새로운 코루틴에서 만들어진 스레드에서 실행되면, 이 작업은 테스트 스레드와 별도로 비동기적으로 시작됩니다. 한편, 테스트 코루틴은 명령을 병렬로 계속 실행합니다. 테스트는 먼저 실행된 작업(fired-off) 중 하나가 완료되기 전에 완료될 가능성이 있습니다.

![](<../../../.gitbook/assets/Untitled (34).png>)

동기화 메커니즘은 비동기 작업이 완료될 때까지 테스트 실행을 “대기”하도록 지시하는 방법입니다.

![](<../../../.gitbook/assets/Untitled (35).png>)

코틀린에서, 코드를 비동적으로 실행하기 위한 일반적인 메커니즘은 코루틴입니다. 비동기 코드를 테스팅할 때, 코드를 결정적으로 만들고 동기화 메커니즘을 제공해야 합니다.

* `runBlockingTest` 또는 `runBlocking` 사용
* 로컬 테스트에 `TestCoroutineDispatcher` 사용
* 코루틴 실행을 일시 중지하여 정확한 타이밍의 위치에서 코드 상태를 테스트

먼저 `runBlockingTest` 와 `runBlocking` 의 차이점을 살펴보겠습니다.

### Observe how to run basic coroutines in tests

`suspend` 함수가 포함된 테스트 코드에서는 다음을 따라야합니다.

1. `kotlinx-coroutines-test` 테스트 의존성을 app 단위 build.gradle 파일에 추가합니다.
2. `@ExperimentalCoroutinesApi` 애노테이션을 추가합니다.
3. 테스트가 완료될 때까지 대기하도록 코드를 `runBlockingTest` 로 블록을 만듭니다.

💡 FakeTestRepository와 같은 테스트 더블을 작성할 때는 runBlocking을 사용해야 합니다. 이전 코드랩에서는 runBlocking, runBlockingTest 모두 사용했습니다.



`kotlinx-coroutines-test` 은 테스트 코루틴을 위한 실험적인(배타 느낌) 라이브러리입니다. 이 라이브러리에 `runBlokingTest` 를 포함하고 있으며 코루틴을 테스트하기 위한 유틸리티가 포함되어 있습니다.

테스트에서 코루틴을 실행하려면 항상 `runBlokingTest` 을 사용해야 합니다. 일반적으로 코루틴 테스트할 때 suspend 함수를 호출합니다.

**TaskDetailFragmentTest.kt**

```kotlin
@MediumTest
@RunWith(AndroidJUnit4::class)
@ExperimentalCoroutinesApi // LOOK HERE
class TaskDetailFragmentTest {

    //... Setup and teardown

    @Test
    fun activeTaskDetails_DisplayedInUi() = runBlockingTest{ // LOOK HERE
        // GIVEN - Add active (incomplete) task to the DB.
        val activeTask = Task("Active Task", "AndroidX Rocks", false)
        repository.saveTask(activeTask) // LOOK HERE Example of calling a suspend function

        // WHEN - Details fragment launched to display task.
        val bundle = TaskDetailFragmentArgs(activeTask.id).toBundle()
        launchFragmentInContainer<TaskDetailFragment>(bundle, R.style.AppTheme)

        // THEN - Task details are displayed on the screen.
        // Make sure that the title/description are both shown and correct.
        // Lots of Espresso code...
    }

    // More tests...
}
```

* `@ExperimentalCoroutinesApi` 애노테이션을 추가해야 합니다.
* suspend 함수를 호출하는 코드를 `runBlockingTest` 블록으로 감쌉니다.

suspend 함수인 `repository.saveTask(activeTask)` 를 호출하기 때문에 위의 코드에서 `runBlockingTest`를 사용합니다.

`runBlockingTest` 는 결정론적(deterministically)으로 코드를 실행하는 것과 동기화 메커니즘을 제공하는 것을 모두 처리합니다. `runBlockingTest` 는 코드 블록(안에 사용된 코드들)을 가져와 시작하는 모든 코루틴이 완료될때까지 테스트 스레드를 차단합니다. 또한 코루틴에서 즉시 코드를 실행합니다(delay를 건너뛰고). 즉, 결정록적인 순서로 코드를 실행합니다.

`runBlockingTest` 은 기본적으로 테스트 코드 전용의 코루틴 컨텍스트를 제공하여, 코루틴이 non-coroutine 처럼 실행되도록 합니다. 왜냐하면 테스트에서 해당 작업을 수행하는 것은 코드가 매번 동일한 방식으로 실행되는 것이 중요하기 때문입니다(synchronous and deterministic).

### Observe runBlocking in Test Doubles

`runBlocking`은 테스트 클래스와 달리 테스트 더블에서 코루틴을 사용할 때 사용됩니다. `runBlocking` 은 `runBlockingTest` 와 거의 비슷하게 사용되지만, 코드 블록을 감아서(내부) 사용할 수 있습니다.

1. FakeTestRepository.kt를 살펴보자. `runBlocking`은 `kotlinx-coroutines-test` 라이브러리의 일부가 아니므로 `@ExperimentalCoroutinesApi` 애노테이션을 사용할 필요가 없습니다.

**FakeTestRepository.kt**

```kotlin
class FakeTestRepository : TasksRepository {

    var tasksServiceData: LinkedHashMap<String, Task> = LinkedHashMap()

    private val observableTasks = MutableLiveData<Result<List<Task>>>()

    // More code...

    override fun observeTasks(): LiveData<Result<List<Task>>> {
        runBlocking { refreshTasks() } // LOOK HERE
        return observableTasks
    }

    override suspend fun refreshTasks() {
        observableTasks.value = getTasks()
    }
    
    // More code...
}
```

`runBlockingTest`와 비슷하게 `runBlocking`은 refreshTasks가 suspend 함수이기 떄문에 사용됩니다.

#### runBlocking vs runBlockingTest

`runBlocking` 과 `runBlockingTest` 은 모두 현재 스레드를 차단하고 람다에서 실행된 관련 코루틴이 완료될 때까지 대기합니다.

또한, `runBlockingTest` 에는 테스트를 위한 다음과 같은 동작이 있습니다.

1. `delay` 를 스킵하기 때문에 테스트가 더 빠르게 실행됩니다.
2. 코루틴의 끝에 테스트와 관련된 assertions을 추가합니다. 이 assertions은 코루틴을 실행한 후 `runBlocking` 람다가 종료된 후(코루틴 leak의 가능성이 있음)에도 계속 실행되거나 잡히지 않은 예외가 있는 경우 실패합니다.
3. 코루틴 실행에 대한 타이밍 제어(timing control)를 제공합니다.

FakeTestRepository와 같은 테스트 더블에서 runBlocking을 사용하는 이유는 무엇일까? 때때로 테스트 더블에서 코루틴을 사용할 때가 있을 수 있으며, 이 경우 현재 스레드를 차단해야 하기 때문입니다. 테스트 케이스에서 테스트 더블을 사용할 때 **스레드가 차단되고 테스트 전에 코루틴이 완료되도록 해야 합니다**.

그러나 테스트 더블은 가상 테스트 데이터 셋을 정의하는 것이므로, 테스트에 특정한 모든 테스트 기능과 `runBlockingTest` 를 사용할 필요도 없고 사용해서는 안됩니다.
