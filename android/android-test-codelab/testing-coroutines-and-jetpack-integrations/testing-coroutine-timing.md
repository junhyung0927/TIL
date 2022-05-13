# Testing Coroutine Timing



`TestCoroutineDispatcher` 의 `pauseDispatcher` 와 `resumeDispatcher` 함수를 사용하여 테스트에서 코루틴의 실행을 어떻게 제어하는지 알아봅니다. 이 방법을 사용해서 `StatisticsViewModel` 의 로딩 인디케이터에 대한 테스트를 작성합니다.

`StatisticsViewModel` 은 모든 데이터를 저장하고 통계 화면의 모든 계산을 수행합니다.

![](<../../../.gitbook/assets/Untitled (1) (1).png>)

### Prepare StatisticsViewModel for testing

우선 이전 코드랩에서 프로세스를 설명했듯이, ViewModel에 fake 리포짓토리를 주입할 수 있는지 확인해야 합니다. 하지만 이는 코루틴 타이밍과 관련이 없으므로 자유롭게 복사/붙여넣기를 하면 됩니다.

1. `StatisticsViewModel` 열기
2. `StatisticsViewModel` 의 생성자를 클래스 내에서 구성하는 대신 TasksRepository에서 가져오도록 변경하여 테스트를 위해 fake 리포짓토리를 주입할 수 있습니다.

**StatisticsViewModel.kt**

```kotlin
// REPLACE
class StatisticsViewModel(application: Application) : AndroidViewModel(application) {

    private val tasksRepository = (application as TodoApplication).taskRepository

    // Rest of class
}

// WITH

class StatisticsViewModel(
    private val tasksRepository: TasksRepository
) : ViewModel() { 
    // Rest of class 
}
```

1. `StatisticsViewModel` 파일의 맨 하단에 `TasksRepository` 를 포함하는 `TasksViewModelFactory` 를 추가합니다.

```kotlin
@Suppress("UNCHECKED_CAST")
class StatisticsViewModelFactory (
    private val tasksRepository: TasksRepository
) : ViewModelProvider.NewInstanceFactory() {
    override fun <T : ViewModel> create(modelClass: Class<T>) =
        (StatisticsViewModel(tasksRepository) as T)
```

1. factory를 사용하게 `StatisticsFragment` 를 업데이트합니다.

```kotlin
// REPLACE
private val viewModel by viewModels<TasksViewModel>()

// WITH

private val viewModel by viewModels<StatisticsViewModel> {
    StatisticsViewModelFactory(
        (requireContext().applicationContext as TodoApplication).taskRepository
    )
}
```

1. 애플리케이션을 실행하고 잘 동작하는지 확인합니다.

### Create StatisticsViewModelTest

이제 `StatisticsViewModelTest` 에 대한 코루틴 실행 도중에 일시 중지되는 테스트를 생성할 준비가 되었습니다.

`StatisticsViewModelTest` 테스트 클래스를 생성합니다.

아래에서 설명하는 단계에 따라 이전 섹션에서 설명한 대로 StatisticsViewModel 테스트를 설정합니다.

1. `InstantTaskExecutorRule` 을 추가합니다. Architecture Components(ViewModel이 속한)에서 사용하는 백그라운드 executor가 각 작업을 동기적으로 실행하는 executor와 스왑됩니다. 이제 테스트는 결정론적임을 보장할 수 있습니다.
2. 코루틴 및 ViewModel을 테스트하고 있으므로 `MainCoroutineRule`을 추가합니다.
3. 테스트에 대한 viewModel 필드를 만들고(StatisticsViewModel) 의존성에 대한 테스트 더블을 생성합니다(FakeTestRepository).
4. subject under 테스트와 의존성을 설정하기 위해 `@Before` 함수를 생성합니다.

**StatisticsViewModelTest.kt**

```kotlin
@ExperimentalCoroutinesApi
class StatisticsViewModelTest {

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    // Set the main coroutines dispatcher for unit testing.
    @ExperimentalCoroutinesApi
    @get:Rule
    var mainCoroutineRule = MainCoroutineRule()

    // Subject under test
    private lateinit var statisticsViewModel: StatisticsViewModel

    // Use a fake repository to be injected into the view model.
    private lateinit var tasksRepository: FakeTestRepository

    @Before
    fun setupStatisticsViewModel() {
        // Initialise the repository with no tasks.
        tasksRepository = FakeTestRepository()

        statisticsViewModel = StatisticsViewModel(tasksRepository)
    }
}
```

### Create a loading indicator test

작업 통계가 로드되면, 데이터가 로드되고 통계 계산이 완료되는 즉시 앱의 로딩 인디케이터는 사라져야 합니다. 통계가 로딩하는 동안 로딩 인디케이터가 표시되는지 확인한 다음, 통계가 로드되면 로딩 인디케이터가 사라지는 테스트를 작성합니다.

StatisticsViewModel 안에 있는 `refresh()` 함수는 로딩 인디케이터가 보여지거나 사라지는 타이밍을 제어합니다.

**StatisticsViewModel.kt**

```kotlin
fun refresh() {
   _dataLoading.value = true
       viewModelScope.launch {
           tasksRepository.refreshTasks()
           _dataLoading.value = false
       }
}
```

\_dataLoading이 true로 설정되고, 나중에 코루틴이 작업을 새로 고침을 완료하면 false로 설정됩니다. 이제 우리는 이 코드가 로딩 인디케이터를 올바르게 업데이터 하는지 확인해야 합니다.

**StatisticsViewModelTest.kt**

```kotlin
@Test
fun loadTasks_loading() {
    
    // Load the task in the view model.
    statisticsViewModel.refresh()

    // Then progress indicator is shown.
    assertThat(statisticsViewModel.dataLoading.getOrAwaitValue(), `is`(true))

    // Then progress indicator is hidden.
    assertThat(statisticsViewModel.dataLoading.getOrAwaitValue(), `is`(false))
}
```

위의 테스트를 진행하면 실패할 것이다. 데이터 로딩이 true, false인 것을 동시에 테스트하기 때문에 의미가 없습니다.

에러 메시지를 살펴보면, 첫 번째 assert 문 때문에 테스트가 실패한 것을 확인할 수 있습니다.

`TestCoroutineDispatcher` 는 작업을 즉시 완전하게 실행합니다. 즉, assert 문이 실행되기 전에 `statisticsViewModel.refresh()` 함수가 완전히 종료됩니다. 테스트를 작성하다 보면 테스트가 빠르게 실행되도록 즉시 실행하려는 경우가 있습니다. 하지만 지금 작성하고 있는 테스트 케이스의 경우, 새로 고침이 실행되는 중간에 로딩 인디케이터의 상태를 확인하려고 하기 때문에 주의해야 합니다.

**StatisticsViewModel.kt**

```kotlin
fun refresh() {
   _dataLoading.value = true
   // YOU WANT TO CHECK HERE...
   viewModelScope.launch {
       tasksRepository.refreshTasks()
       _dataLoading.value = false
       // ...AND CHECK HERE.
   }
}
```

이러한 상황의 경우 `TestCoroutineDispatcher` 의 `pauseDispatcher` 와 `resumeDispatcher` 를 사용할 수 있습니다. `mainCoroutineRule` 도 사용합니다.

`pauseDispatcher` 은 `TestCoroutineDispatcher` 를 일시 중지하기 위한 약칭입니다. 디스패처가 일시 중지되면 새로운 코루틴이 즉시 실행되지 않고 큐에 추가됩니다. 즉, 새로 고침 내의 코드 실행은 코루틴이 실행되기 직전에 일시 중지됩니다.

**StatisticsViewModel.kt**

```kotlin
fun refresh() {
   _dataLoading.value = true
?   // PAUSES EXECUTION HERE
   viewModelScope.launch {
       tasksRepository.refreshTasks()
       _dataLoading.value = false
   }
}
```

`mainCoroutineRule.resumeDispatcher()` 을 호출할 때, 코루틴의 모든 코드가 실행될 것입니다.

테스트에 `pauseDispatcher` 와 `resumeDispatcher` 를 사용하도록 변경하여 코루틴을 실행하기 전에 일시 중지하고 로딩 인디케이터가 표시되는지 확인한 다음, 로딩 인디케이터가 사라졌는지 확인합니다.



❓`pauseDispatcher` 를 실행하면 \_dataLoading.value = true까지만 실행하고 dispatcher를 pause 상태로 바꿀까요?

그 이유는 `mainCoroutineRule` 를 통해 Main 디스패처를 중지시켰기 때문입니다. viewModelScope는 Main 디스패처를 사용하기 때문에 중지했을 때 value가 true를 테스트 통과한 후 resume을 통해 value가 false일 때를 테스트합니다. 만약 Main 디스패처가 아닌 GlobalScope, IO 등으로 사용했을 때는 중지가 안되기 때문에 테스트 실패가 발생합니다.



**StatisticsViewModelTest.kt**

```kotlin
@Test
fun loadTasksloading() {
    // Pause dispatcher so you can verify initial values.
    mainCoroutineRule.pauseDispatcher()

    // Load the task in the view model.
    statisticsViewModel.refresh()

    // Then assert that the progress indicator is shown.
    assertThat(statisticsViewModel.dataLoading.getOrAwaitValue(), `is`(true))

    // Execute pending coroutines actions.
    mainCoroutineRule.resumeDispatcher()

    // Then assert that the progress indicator is hidden.
    assertThat(statisticsViewModel.dataLoading.getOrAwaitValue(), `is`(false))
}
```

이제 테스트가 통과되는지 확인합니다.

> 보다 자세한 타이밍 제어가 필요한 경우, TestCoroutineDispatcher에서 아래와 같은 기능을 제공합니다.

* advanceTimeBy
* advanceUntilIdle
* runCurrent
