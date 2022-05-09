# Coroutines and ViewModels

ViewModel 내에서 코루틴을 사용하는 테스트 방법을 알아봅시다.

모든 코루틴은 `CoroutineScope` 가 필요합니다. 코루틴 스코프는 코루틴의 수명주기를 제어합니다. 스코프를 취소할 때, 스코프 내에서 실행되고 있는 모든 코루틴을 취소됩니다.

ViewModel에서 장기간으로 실행되는 작업을 수행할 수 있으므로 ViewModel 내에서 코루틴을 작성하고 실행하는 경우가 많습니다. 일반적으로 코루틴을 실행하려면 ViewModel에 새로운 `CoroutineScope` 를 일일이 작성하고 구성해야 합니다. 이는 많은 보일러 플레이트 코드를 발생하게 합니다.

이것을 피하고자, `lifecycle-viewmodel-ktx` 는 `viewModelScope()` 확장 함수를 제공합니다.

`viewModelScope()` 는 각 ViewModel와 관련된 `CoroutineScope` 를 리턴합니다. `viewModelScope` 는 특정 ViewModel에서 사용하도록 구성되어 있습니다.

* `viewModelScope()` 는 ViewModel과 연결되어 ViewModel이 끝날 때(onCleared가 호출될 때) 스코프가 취소됩니다. ViewModel이 끝나면 ViewModel과 관련된 모든 코루틴 작업도 끝나게 됩니다. 이로 인해 낭비되는 작업이나 메모리 누수가 방지됩니다.
* `viewModelScope` 는 `Dispatchers.Main` 코루틴 디스패처를 사용합니다. `CoroutineDispatcher` 는 코루틴 코드가 실행되는 스레드를 포함하여 코루틴 실행 방법을 제어합니다. `Dispatchers.Main` 은 UI 또는 메인 스레드에 코루틴을 배치(puts)합니다. ViewModel이 UI를 조작하는 경우가 많기 때문에(UI를 조작한다는 것은 양방향 데이터 바인딩이나 관련된 데이터를 가공하여 Activity 또는 Fragment에 던져주는 것을 의미한다) ViewModel의 코루틴의 디폴트로 적합합니다.

이것은 production 코드에서 잘 동작합니다. 하지만 로컬 테스트(테스트 소스 셋이 로컬 머신에서 테스트하는 경우)의 경우 `Dispatchers.Main` 을 사용하면 이슈가 발생합니다.

*   `Dispatchers.Main` 은 Android의 `Looper.getMainLooper()` 을 사용합니다. 메인 looper는 실제 애플리케이션을 위한 실행 루프입니다. 전체 애플리케이션을 실행하고 있지 않기 때문에 로컬 테스트에서는 기본 루프(looper)를 사용할 수 없습니다.

    \<aside> 💡 getMainLooper()는 메인 스레드(UI Thread)가 사용하는 Main Looper 루퍼를 반환합니다. 이 함수는 호출하는 스레드가 메인 스레드 여부를 떠나 언제든지 Main Looper를 반환합니다.

    \</aside>

위의 문제를 해결하려면, `setMain()` 함수(`kolinx.coroutines.test`)를 사용하여 `TestCoroutineDispatcher` 를 사용하도록 `Dispatchers.Main` 를 수정해야 합니다.

`TestCoroutineDispatcher` 는 테스트용으로 특별히 제작된 디스패처입니다.

### Observe Dispatcher.Main causing an error

작업이 완료되면 snackbar에 올바른 ‘completion message’가 표시되는지 확인하는 테스트를 추가합니다.

1. test > tasks > TasksViewModelTest 열기
2. 새로운 테스트 함수 추가

**TasksViewModelTest.kt**

```kotlin
@Test
fun completeTask_dataAndSnackbarUpdated() {
    // Create an active task and add it to the repository.
    val task = Task("Title", "Description")
    tasksRepository.addTasks(task)

    // Mark the task as complete task.
    tasksViewModel.completeTask(task, true)

    // Verify the task is completed.
   assertThat(tasksRepository.tasksServiceData[task.id]?.isCompleted, `is`(true))

    // Assert that the snackbar has been updated with the correct text.
    val snackbarText: Event<Int> =  tasksViewModel.snackbarText.getOrAwaitValue()
    assertThat(snackbarText.getContentIfNotHandled(), `is`(R.string.task_marked_complete))
}
```

1. Test를 실행하면 실패 로그가 뜸

**"Exception in thread "main" java.lang.IllegalStateException: Module with the Main dispatcher had failed to initialize. For tests Dispatchers.setMain from kotlinx-coroutines-test module can be used."**

위의 에러는 `Dispatcher.Main` 가 초기화에 실패했음을 나타냅니다. 근본적인 이유는 `Looper.getMainLooper` 의 looper가 없기 때문입니다. 그렇기에 `Dispatcher.setMain` 를 사용하여 에러를 해결하면 됩니다.

### Replace Dispatcher.Main with TestCoroutineDispatcher

`TestCoroutineDispatcher` 는 테스트를 위한 코루틴 디스패처입니다. 작업을 즉시 실행하고 테스트에서 코루틴 실행을 일시 중지한 후 다시 시작할 수 있도록 하는 등 코루틴 실행 타이밍을 제어할 수 있습니다.

1. TasksViewModelTest에서 `val` 의 `TestCoroutineDispatcher` 타입 변수인 `testDispatcher` 를 선언합니다.

디폴트 메인 디스패처 대신 `testDispatcher` 를 사용합니다.

1. 모든 테스트에서 사용하도록 `@Before` 애노테이션을 추가한 함수에 `Dispatchers.setMain(testDispatcher` 를 선언합니다.
2. `Dispatchers.resetMain()` 를 호출한 후 `testDispatcher.cleanupTestCoroutines()` 를 실행하여 각 테스트를 실행한 후에 모든 것은 완료(clean)하는 `@After` 애노테이션을 추가한 함수를 만듭니다.

**TasksViewModelTest.kt**

```kotlin
@ExperimentalCoroutinesApi
val testDispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()

@ExperimentalCoroutinesApi
@Before
fun setupDispatcher() {
    Dispatchers.setMain(testDispatcher)
}

@ExperimentalCoroutinesApi
@After
fun tearDownDispatcher() {
    Dispatchers.resetMain()
    testDispatcher.cleanupTestCoroutines()
}
```

### Add MainCoroutineRule

앱에서 코루틴을 사용하고 있다면, ViewModel에서 코루틴을 호출하는 코드를 포함하는 로컬 테스트는 `viewModelScope` 를 사용할 확률이 높습니다. `TestCoroutineDispatcher` 를 각 테스트 클래스로 설정하고 해제(tear down)하는 코드를 복붙하는 대신 커스텀 JUit rule을 만들어 보일러 플레이트 코드를 방지할 수 있습니다.

`JUit rules` 은 테스트 전, 후 또는 테스트 중에 실행할 수 있는 일반 테스트 코드를 정의할 수 있는 클래스입니다. `@Before` , `@After` 에 있던 코드를 다시 사용할 수 있는 클래스에 넣는 방식입니다.

root에 MainCoroutineRule.kt 클래스를 생성한 후 코드를 작성합니다.

**MainCoroutineRule.kt**

```kotlin
@ExperimentalCoroutinesApi
class MainCoroutineRule(val dispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()):
   TestWatcher(),
   TestCoroutineScope by TestCoroutineScope(dispatcher) {
   override fun starting(description: Description?) {
       super.starting(description)
       Dispatchers.setMain(dispatcher)
   }

   override fun finished(description: Description?) {
       super.finished(description)
       cleanupTestCoroutines()
       Dispatchers.resetMain()
   }
}
```

* `MainCoroutineRule` 은 `TestRule` 인터페이스의 구현체인 `TestWatcher` 을 상속받습니다. 이것이 `MainCoroutineRule` 을 JUnit rule로 만드는 것입니다.
* `starting` 과 `finished` 함수는 @Before과 @After 함수의 코드와 동일한 방식으로 작성하면 됩니다. 동일하게 각 테스트 전 후에 실행됩니다.
* `MainCoroutineRule` 은 `TestCoroutineDispathcer` 에 전달되는 `TestCoroutineScope`를 구현합니다. 이제 `MainCoroutineRule` 에 전달한 `TestCoroutineDispathcer` 를 사용하여 코루틴 타이밍을 제어할 수 있습니다.

### Use your new Junit rule in a test

TasksViewModelTest에 `testDispatcher`, `@Before`, `@After` 코드를 JUnit rule인 새로운 `MainCoroutineRule` 으로 대체합니다.

**TasksViewModelTest.kt**

```kotlin
// REPLACE@ExperimentalCoroutinesApi
val testDispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()

@ExperimentalCoroutinesApi
@Before
fun setupDispatcher() {
    Dispatchers.setMain(testDispatcher)
}

@ExperimentalCoroutinesApi
@After
fun tearDownDispatcher() {
    Dispatchers.resetMain()
    testDispatcher.cleanupTestCoroutines()
}
// WITH
@ExperimentalCoroutinesApi
@get:Rule
var mainCoroutineRule = MainCoroutineRule()
```

JUnit rule을 사용하기 위해 새로운 rule을 인스턴스화 한 후 @get:Rule 애노테이션을 추가합니다.

### Use MainCoroutineRule for repository testing

이전에 배운 의존성 주입을 사용하면, 테스트에서 production 버전을 클래스의 테스트 버전으로 대체할 수 있습니다. 생성자 의존성 주입을 사용했었습니다.

**DefaultTasksRepository.kt**

```kotlin
class DefaultTasksRepository(
    private val tasksRemoteDataSource: TasksDataSource,
    private val tasksLocalDataSource: TasksDataSource,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO) : TasksRepository { ... }
```

위의 코드는 local 및 remote 데이터 소스와 `CoroutineDispatcher`를 주입합니다. 디스패처가 주입되었으므로 테스트에서 `TestCoroutineDispatcher`를 사용할 수 있습니다. 코루틴을 사용할 때는 하드 코딩을 하지 말고 `CoroutineDispatcher` 을 주입하는 것이 좋습니다.

1. test > data > source > DefaultTasksRepositoryTest.kt
2. `DefaultTasksRepositoryTest` 내에 `MainCoroutineRule` 을 추가합니다.

**DefaultTasksRepositoryTest.kt**

```kotlin
// Set the main coroutines dispatcher for unit testing.
@ExperimentalCoroutinesApi
@get:Rule
var mainCoroutineRule = MainCoroutineRule()
```

1. test 속에 있는 레포짓토리를 정의할 때 `Dispatcher.Unconfined` 대신 `Dispatcher.Main`을 사용합니다. `TestCoroutineDispatcher` 와 `Dispatcher.Unconfined` 는 비슷하게 작업을 즉시 실행합니다. 하지만, 실행을 일시 중지할 수 있는 것과 같은 `TestCoroutineDispatcher` 의 테스트 장점을 포함되지 않습니다.
2.

**DefaultTasksRepositoryTest.kt**

```kotlin
@Before
fun createRepository() {
    tasksRemoteDataSource = FakeDataSource(remoteTasks.toMutableList())
    tasksLocalDataSource = FakeDataSource(localTasks.toMutableList())
    // Get a reference to the class under test.
    tasksRepository = DefaultTasksRepository(
    // HERE Swap Dispatcher.Unconfined
        tasksRemoteDataSource, tasksLocalDataSource, Dispatchers.Main
    )
}
```

위의 코드에서 `MainCoroutineRule` 이 `Dispatcher.Main` 이 `TestCoroutineDispatcher` 와 스왑된다는 것을 기억해야 합니다.

일반적으로 테스트가 실행될 때 `TestCoroutineDispatcher` 는 오직 하나만 생성됩니다. `runBlockingTest` 를 호출할 때마다, `TestCoroutineDispatcher` 를 지정하지 않으면 새로운 `TestCoroutineDispatcher` 가 생성됩니다. `MainCoroutineRule` 에는 `TestCoroutineDispatcher` 가 포함됩니다.

따라서 뜻하지 않게 `TestCoroutineDispatcher` 인스턴스를 여러 개 만들지 않도록, `runBlockingTest` 를 실행하는 대신 `[mainCoroutineRule.runBlockingTest](<https://www.youtube.com/watch?t=747&v=KMb0Fs8rCRs&feature=youtu.be>)` 를 사용해야 합니다.

1. `runBlockingTest` 를 `mainCoroutineRule.runBlockingTest` 로 대체합니다.

**DefaultTasksRepositoryTest.kt**

```kotlin
// REPLACE
fun getTasks_requestsAllTasksFromRemoteDataSource() = runBlockingTest {

// WITH
fun getTasks_requestsAllTasksFromRemoteDataSource() = mainCoroutineRule.runBlockingTest {
```

이제 테스트에 적합한 디스패처인 TestCoroutineDispatcher를 코드에서 사용하고 있습니다. 다음으로 코루틴 실행 타이밍을 제어하는 TestCoroutineDispatcher의 추가 기능을 사용하는 방법에 대해서 알아보겠습니다.

***

[\[Android\] Looper & Handler 기초 개념](https://velog.io/@haero\_kim/Android-Looper-Handler-%EA%B8%B0%EC%B4%88-%EA%B0%9C%EB%85%90)

[getMainLooper 사용하기](https://codetravel.tistory.com/19)
