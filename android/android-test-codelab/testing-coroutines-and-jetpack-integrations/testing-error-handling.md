# Testing Error Handling

테스트에서는 코드가 예상대로 실행될 때와 앱이 에러 및 엣지(edge) 케이스가 발생했을 때 모두 테스트하는 것이 중요합니다. 이 챕터에서는 작업 목록을 로드할 수 없는 경우(ex: 네트워크 중단) 올바른 동작을 확인하는 테스트를 StatisticsViewModelTest에 작성합니다.

### Add an error flag to test double

우선, 가상(인위적)의 에러 케이스가 필요합니다. 이를 위한 한 가지 방법은 flag를 사용해서 테스트 더블을 에러 상태로 “set”할 수 있도록 업데이트 하는 것입니다.

flag가 false라면, 테스트 더블 함수는 정상적으로 작동할 것입니다. 하지만 flag가 true로 설정됐다면, 테스트 더블은 realistic한 에러를 반환할 것입니다. 데이터 로드 에러를 반환 받았다고 예를 들어봅시다. 우선, 에러 flag를 `FakeTestRepository` 에 추가하고, 데이터 로드 에러를 반환 받았으니 flag는 true로 설정되어 코드는 realistic 에러를 반환할 것입니다.

1. test > data > source > `FakeTestRepository`
2. `shouldResturnError` 파라미터를 선언합니다(Boolean). 에러는 디폴트로 발생되지 않기 때문에 초기값은 false로 설정합니다.

**FakeTestRepository.kt**

```kotlin
private var shouldReturnError = false
```

1. 리포짓토리에서 에러를 반환할지에 대한 여부를 변경하는 `setReturnError` 함수를 생성합니다.

**FakeTestRepository.kt**

```kotlin
fun setReturnError(value: Boolean) {
    shouldReturnError = value
}
```

1. `getTask` 와 `getTasks` 함수 안에 if 조건문을 생성하여 `shouldReturnError` 가 true 일때, Result.Error를 반환하도록 작성합니다.

**FakeTestRepository.kt**

```kotlin
// Avoid import conflicts:
import com.example.android.architecture.blueprints.todoapp.data.Result
import com.example.android.architecture.blueprints.todoapp.data.Result.Error
import com.example.android.architecture.blueprints.todoapp.data.Result.Success

...

override suspend fun getTask(taskId: String, forceUpdate: Boolean): Result<Task> {
    if (shouldReturnError) {
        return Result.Error(Exception("Test exception"))
    }
    tasksServiceData[taskId]?.let {
        return Success(it)
    }
    return Error(Exception("Could not find task"))
}

override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
    if (shouldReturnError) {
        return Error(Exception("Test exception"))
    }
    return Success(tasksServiceData.values.toList())
}
```

### Write a test for a returned Error

이제 리포짓토리가 에러를 반환해서 StatisticsViewModel에 문제가 생겼을 때 테스트를 작성할 준비가 완료되었습니다.

StatisticsViewModel에 두 개의 Boolean 타입의 LiveData 파라미터를 생성합니다(error & empty)

**StatisticsViewModel.kt**

```kotlin
class StatisticsViewModel(
    private val tasksRepository: TasksRepository
) : ViewModel() {

    private val tasks: LiveData<Result<List<Task>>> = tasksRepository.observeTasks()

    // Other variables...    
  
    val error: LiveData<Boolean> = tasks.map { it is Error }
    val empty: LiveData<Boolean> = tasks.map { (it as? Success)?.data.isNullOrEmpty() }

    // Rest of the code...    
}
```

이는 `tasks`이 제대로 로드되었는지 여부를 나타냅니다. 에러가 있으면 `error`와 `empty` 값이 모두 참이어야 합니다.

1. StatisticsViewModelTest.kt을 엽니다.
2. 새로운 테스트 `loadStatisticsWhenTasksAreUnavailable_callErrorToDisplay` 를 생성합니다.
3. `tasksRepository.setReturnError()` 를 true로 설정합니다.
4. `statisticsViewModel.empty` 와 `statisticsViewModel.error` 모두 true로 설정되었는지 체크합니다.

**StatisticsViewModelTest.kt**

```kotlin
@Test
fun loadStatisticsWhenTasksAreUnavailable_callErrorToDisplay() {
    // Make the repository return errors.
    tasksRepository.setReturnError(true)
    statisticsViewModel.refresh()

    // Then empty and error are true (which triggers an error message to be shown).
    assertThat(statisticsViewModel.empty.getOrAwaitValue(), `is`(true))
    assertThat(statisticsViewModel.error.getOrAwaitValue(), `is`(true))
}
```
