# End-to-End Testing with Data Binding

이전의 코드랩에서는 Unit 테스트와 Intergration 테스트를 작성했습니다. `TaskDetailFragmentTest` 와 같은 Intergration 테스트는 다른 프래그먼트로 이동하거나 액티비티를 생성하지 않고 싱글 프래그먼트 기능 테스트에만 초점을 맞춥니다. 마찬가지로 `TaskLocalDataSourceTest` 는 데이터 계층에서 함께 작동하는 몇 가지 클래스를 테스트 하지만, 실제로는 UI를 검사(체크)하지 않습니다.

End-to-End 테스트(E2E)는 함께 작동하는 기능의 조합을 테스트합니다. E2E는 앱의 많은 부분을 테스트하고 실제로 사용하는 것(시나리오 같은)들을 시뮬레이션 합니다. 대체로 이러한 테스트는 Instrumented 테스트(androidTest source set)입니다.

다음은 Todo 앱과 관련된 E2E 테스트와 Intergration 테스트 간의 차이점을 설명합니다: E2E Test

* 첫 화면에서 앱을 시작합니다.
* 실제 액티비티와 리포짓토리를 생성합니다.
* 함께 작동하는 여러 프래그먼트를 테스트합니다.

E2E 테스트 매우 복잡하게 작성됩니다(난이도 상승). 그래서 쉽게 만들 수 있는 도구와 라이브러리가 제공됩니다.

에스프레소는 E2E 테스트를 작성하는데 일반적으로 사용되는 Android UI 테스트 라이브러리입니다. 앞서 코드랩에서 사용한 바가 있습니다.

이 챕터에서는 제대로 E2E 테스트를 작성합니다. [Expresso idling(유휴) 리소스](https://developer.android.com/training/testing/espresso/idling-resource?hl=ko)를 사용해서 길게 실행되는 작업과 데이터바인딩 라이브러리를 모두 포함하는 E2E 테스트를 작성할 수 있습니다.

*   idling resource

    결과가 UI 테스트의 후속 작업에 영향을 미치는 비동기 작업을 나타냅니다. 유휴 리소스를 Espresso에 등록하면 앱을 테스트할 때 이러한 비동기 작업을 더욱 안정적으로 검증할 수 있습니다.

### Turn off animations

**Espresso UI 테스트를 할 때는 애니메이션을 끄고 진행하는 것이 좋습니다.**

1. Settings > Developer options
2. Window animation scale, Transition animation scale, Animator duration scale

### ![](<../../../.gitbook/assets/Untitled (3) (1).png>)

### Create TasksActivity Test

1. androidTest > TasksActivityTest.kt 생성
2. AndroidX 테스트 코드를 사용하기 위해 `@RunWith(AndroidJUnit4::class)` 애노테이션 추가
3. 코드의 많은 부분을 테스트하는 E2E 테스트를 진행하는 것을 나타내기 위해 `@LargeTest` 애노테이션 추가

E2E 테스트는 전체 앱이 실행되는 방식을 모방하고 실제로 사용되는 것을 시뮬레이션합니다. 따라서 리포짓토리 또는 테스트 더블 리포짓토리를 인스턴스화하는 대신 `ServiceLocator`에서 리포짓토리를 생성할 수 있습니다.

1. `TasksRepository`타입의 repository 프로퍼티를 만듭니다.
2. repository 프로퍼티에 `ServiceLocator` 의 `provideTasksRepository` 함수를 호출한 값을 초기화하기 위해 `@Before` 함수에 작성합니다: application context를 얻기 위해 `getApplicationContext` 사용
3. `@Before` 함수에서 리포짓토리의 모든 작업을 삭제해서 각각의 테스트를 실행하기 전에 완전히 지워지도록 합니다.
4. `ServiceLocator` 의 `resetRepository()` 함수를 호출하기 위해 `@After` 함수를 만듭니다.

**TasksActivity.kt**

```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class TasksActivityTest {

    private lateinit var repository: TasksRepository

    @Before
    fun init() {
        repository = ServiceLocator.provideTasksRepository(getApplicationContext())
        runBlocking {
            repository.deleteAllTasks()
        }
    }

    @After
    fun reset() {
        ServiceLocator.resetRepository()
    }
}
```

> clean-up 코드인 테스트 전(Before), 후(After)에 어떤 작업을 넣어야 할까요? @After 함수에는 테스트가 완료된 후에도 실행 중인 코드나 메모리 누수가 발생하지 않기 위한 코드를 작성해야 합니다(ex: 코루틴 정리 or 데이터베이스 종료) @Before 함수에는 테스트에 들어가기 전에 초기화 코드나 코드의 상태를 정확하게 확인(?)하는 코드를 작성해야 합니다.

### Write an End-to-End Espresso Test

1. TasksActivityTest 열기
2. 다음과 같은 스켈레톤 코드를 작성하자

**TasksActivityTest.kt**

```kotlin
@Test
fun editTask() = runBlocking {
    // Set initial state.
    repository.saveTask(Task("TITLE1", "DESCRIPTION"))
    
    // Start up Tasks screen.
    val activityScenario = ActivityScenario.launch(TasksActivity::class.java)

    // Espresso code will go here.

    // Make sure the activity is closed before resetting the db:
    activityScenario.close()
}
```

* `runBlocking` 은 블록에서 실행을 계속하기 전에 모든 일시 중단 함수가 완료될 때까지 대기하는데 사용됩니다. 버그 때문에 `runBlockingTest` 대신 `runBlocking`을 사용하고 있습니다.
* ``[`ActivityScenario`](https://developer.android.com/reference/androidx/test/core/app/ActivityScenario) 는 액티비티를 감싼 후(wrap) 테스트를 위해 android 라이프사이클을 직접 제어할 수 있게 해주는 AndroidX 테스트 라이브러리 클래스입니다. [`FragmentScenario`](https://developer.android.com/reference/kotlin/androidx/fragment/app/testing/FragmentScenario.html) 와 매우 유사합니다.
* `ActivityScenario` 를 사용할 때, `launch` 를 사용해서 액티비티의 시작하고, `close` 를 사용해서 테스트를 끝냅니다.
* `ActivityScenario.launch()` 를 사용하기 전에 반드시 데이터 계층의 상태를 초기화해야 합니다(ex: 리포짓토리의 작업을 추가하는 것)
* 데이터베이스를 사용하는 경우, 테스트 종료 시 데이터베이스를 종료해야 합니다.

> **ActivityScenarioRule?** launch와 close를 호출하는 [ActivityScenarioRule](https://developer.android.com/reference/androidx/test/ext/junit/rules/ActivityScenarioRule)가 있습니다. 앞서 언급했듯이, 리포짓토리에 작업을 추가하는 것과 같은 데이터 상태의 셋업은 AivityScenario.launch()가 호출되기 전에 수행되어야 합니다. 리포짓토리에 작업을 저장하는 것과 같은 추가적인 셋업 코드는 현재 ActivityScenarioRule에서 지원되지 않습니다. 따라서 ActivityScenarioRule을 사용하지 않고 수동으로 launch 및 close를 호출합니다.

위의 스켈레톤 코드는 액티비티와 관련된 테스트를 위한 기본 설정입니다. `ActivityScenario` 를 launch하고 close 할 때까지 Espresso 코드를 작성할 수 있습니다.

1. 이제 Espresso 코드를 작성해보자.

**TasksActivityTest.kt**

```kotlin
import androidx.test.core.app.ActivityScenario
import androidx.test.core.app.ApplicationProvider.getApplicationContext
import androidx.test.espresso.Espresso.onView
import androidx.test.espresso.action.ViewActions.click
import androidx.test.espresso.action.ViewActions.replaceText
import androidx.test.espresso.assertion.ViewAssertions.doesNotExist
import androidx.test.espresso.assertion.ViewAssertions.matches
import androidx.test.espresso.matcher.ViewMatchers.*
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.filters.LargeTest
import com.example.android.architecture.blueprints.todoapp.data.Task
import com.example.android.architecture.blueprints.todoapp.data.source.TasksRepository
import com.example.android.architecture.blueprints.todoapp.tasks.TasksActivity
import kotlinx.coroutines.runBlocking
import org.hamcrest.core.IsNot.not
import org.junit.After
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith

@Test
fun editTask() = runBlocking {

    // Set initial state.
    repository.saveTask(Task("TITLE1", "DESCRIPTION"))
    
    // Start up Tasks screen.
    val activityScenario = ActivityScenario.launch(TasksActivity::class.java)

    // Click on the task on the list and verify that all the data is correct.
    onView(withText("TITLE1")).perform(click())
    onView(withId(R.id.task_detail_title_text)).check(matches(withText("TITLE1")))
    onView(withId(R.id.task_detail_description_text)).check(matches(withText("DESCRIPTION")))
    onView(withId(R.id.task_detail_complete_checkbox)).check(matches(not(isChecked())))

    // Click on the edit button, edit, and save.
    onView(withId(R.id.edit_task_fab)).perform(click())
    onView(withId(R.id.add_task_title_edit_text)).perform(replaceText("NEW TITLE"))
    onView(withId(R.id.add_task_description_edit_text)).perform(replaceText("NEW DESCRIPTION"))
    onView(withId(R.id.save_task_fab)).perform(click())

    // Verify task is displayed on screen in the task list.
    onView(withText("NEW TITLE")).check(matches(isDisplayed()))
    // Verify previous task is not displayed.
    onView(withText("TITLE1")).check(doesNotExist())
    // Make sure the activity is closed before resetting the db.
    activityScenario.close()
}
```

> 위에서 작성한 E2E 테스트는 리포짓토리, navigation 컨트롤러 또는 다른 컴포넌트의 Intergration을 확인하지 않습니다. 이것을 black box 테스트라 합니다. 테스트는 내부적으로 어떻게 구현되어 있는지 알 수 없으며, 주어진 입력에 대한 결과만 알 수 있습니다.

1. 테스트를 5번 실행합니다. 우리가 작성한 테스트는 실패할 확률이 높습니다. 즉, 통과할 수도 있고 실패할 수도 있습니다.

그 이유는 타이밍 또는 동기화 문제 때문입니다. Espresso는 UI 동작과 UI의 결과에 대한 변경 간에 동기화를 합니다. 예를 들어, Espresso에게 사용자 대신 button을 클릭하도록 한 다음, 특정 뷰가 표시되는지 여부를 확인한다고 가정해봅시다.

```kotlin
onView(withId(R.id.next_screen_button)).perform(click()) // Step 1
onView(withId(R.id.screen_description)).check(matches(withText("The next screen"))) // Step 2
```

Espresso는 1 단계에서 클릭을 수행한 다음, 2 단계에서 “The next screen”이라는 텍스트가 있는지 확인하기 전에 새로운 뷰가 표시될 때까지 대기합니다.

그러나 Espresso의 기본(built-in) 동기화 메커니즘은 뷰를 업데이트할 때까지 충분히 대기하지 못하는 상황이 있습니다. 예를 들어, 뷰를 위해 일부 데이터를 로드해야 하는 경우, Espresso는 해당 데이터가 언제 로드가 완료되었는지를 알 수 없습니다. 또한 Espresso는 데이터바인딩 라이브러리가 뷰를 업데이트하는 시기를 알 수 없습니다.

Espresso가 앱이 UI를 업데이트 중 인지에 대한 여부를 알 수 없는 경우에는 유휴(idling) 리소스 동기화 메커니즘을 사용할 수 있습니다. 이는 Espresso에게 앱이 유휴 상태일 때(Espresso가 앱과 계속 상호 작용하고 확인해야 한다는 의미) 또는 그렇지 않은 경우(Espresso가 기다려야 함)를 명시적으로 알려주는 방법이다.

유휴 리소스를 사용하는 일반적인 방법은 다음과 같습니다:

1. application 코드에 유휴 리소스 또는 서브클래스를 싱글톤으로 생성합니다.
2. application 코드(테스트 코드 X)에 `IdlingResource`의 상태를 idle 또는 not idle로 변경하여, 앱이 유휴 상태인지에 대한 여부를 추적하는 로직을 추가합니다.
3. 각 테스트를 진행하기 전에 [`IdlingRegistry.getInstance().register`](https://developer.android.com/reference/androidx/test/espresso/IdlingRegistry.html#register\(androidx.test.espresso.IdlingResource...\)) 를 호출하여 `IdlingResource` 를 등록합니다. `IdlingResource` 를 등록하면, Espresso는 유휴 상태가 될 때까지 대기했다가 다음 Espresso 문으로 이동합니다.
4. 각 테스트를 모두 마치면 [`IdlingRegistry.getInstance().unregister`](https://developer.android.com/reference/androidx/test/espresso/IdlingRegistry.html#unregister) 를 호출하여 `IdlingResource` 의 등록을 취소합니다.

> **Note:**  It is unusual to have testing code in your application. To understand more about why, and methods of removing idling resource code from your application production code, check out [Android testing with Espresso's Idling Resources and testing fidelity](https://medium.com/androiddevelopers/android-testing-with-espressos-idling-resources-and-testing-fidelity-8b8647ed57f4)

### Add Idling Resource to your Gradle file

1. app/build.gradle에 Espresso idling resource 라이브러리 추가

```kotlin
implementation "androidx.test.espresso:espresso-idling-resource:$espressoVersion"
```

1. `returnDefaultValues = true` 와 `testOptions.unitTests.` 추가

```kotlin
testOptions.unitTests {
        includeAndroidResources = true
        returnDefaultValues = true
}
```

`returnDefaultValues = true` 는 application 코드에 유휴 리소스 코드를 추가할 때 Unit 테스트를 계속 실행하기 위해 추가합니다.

### Create an Idling Resource Singleton

두 개의 유휴 리소스를 추가합니다. 첫 번째는 뷰의 데이터바인딩 동기화를 처리하는 리소스이고, 두 번째는 리포짓토리에서 장시간 실행되는 작업을 처리하는 리소스입니다.

1. app > java > main > util에 `EspressoIdlingResource.kt` 파일 추가
2. 다음 코드를 작성

**EspressoIdlingResource.kt**

```kotlin
object EspressoIdlingResource {

    private const val RESOURCE = "GLOBAL"

    @JvmField
    val countingIdlingResource = CountingIdlingResource(RESOURCE)

    fun increment() {
        countingIdlingResource.increment()
    }

    fun decrement() {
        if (!countingIdlingResource.isIdleNow) {
            countingIdlingResource.decrement()
        }
    }
}
```

이 코드는 `countingIdlingResource` 라는 싱글톤 유휴 리소스를 생성합니다.

``[`CountingIdlingResource`](https://developer.android.com/reference/androidx/test/espresso/idling/CountingIdlingResource.html) 클래스를 사용하고 있는데, 다음과 같이 카운터를 증감할 수 있습니다.

* 카운터가 0보다 크면 앱이 작동하는 것으로 인지
* 카운터가 0이면 앱은 유휴 상태로 간주

기본적으로 앱이 작업을 시작할 때마다 카운터를 늘립니다. 해당 작업이 끝나면 카운터를 줄여야 합니다. 따라서 `CountingIdlingResource` 는 수행 중인 작업이 없는 경우 “count”는 0이 됩니다. 싱글톤으로 만들어져, 장시간 작업이 수행되는 앱의 어디에서난 이 유휴 리소스에 접근할 수 있습니다.

### Create wrapEspressoIdlingResource

다음은 EspressoIdlingResource를 사용하는 방법의 예입니다:

```kotlin
EspressoIdlingResource.increment()
try {
     doSomethingThatTakesALongTime()
} finally {
    EspressoIdlingResource.decrement()
}
```

인라인 함수인 `wrapEspressoIdlingResource` 를 만들어 해당 작업을 단순화할 수 있습니다.

1. EspressoIdlingResource 파일에 `wrapEspressoIdlingResource` 코드를 작성합니다.

**EspressoIdlingResource.kt**

```kotlin
inline fun <T> wrapEspressoIdlingResource(function: () -> T): T {
    // Espresso does not work well with coroutines yet. See
    // <https://github.com/Kotlin/kotlinx.coroutines/issues/982>
    EspressoIdlingResource.increment() // Set app as busy.
    return try {
        function()
    } finally {
        EspressoIdlingResource.decrement() // Set app as idle.
    }
}
```

`wrapEspressoIdlingResource` 는 카운트를 증시켜 래핑된 코드를 실행한 다음 카운트를 감소시킵니다.

```kotlin
wrapEspressoIdlingResource {
    doWorkThatTakesALongTime()
}
```

### Use wrapEspressoIdlingResource in DefaultTasksRepository

다음으로, `wrapEspressoIdlingResource` 를 사용해 장시간 실행되는 작업을 마무리합니다. 이 작업의 대부분은 `DefaultTasksRepository` 에서 진행합니다.

1. data > source > DefaultTasksRepository
2. `DefaultTasksRepository` 에 `wrapEspressoIdlingResource` 를 사용해 모든 함수를 wrap 합니다.

**DefaultTasksRepository.kt**

```kotlin
override suspend fun getTasks(forceUpdate: Boolean): Result<List<Task>> {
    wrapEspressoIdlingResource {
        if (forceUpdate) {
            try {
                updateTasksFromRemoteDataSource()
            } catch (ex: Exception) {
                return Result.Error(ex)
            }
        }
        return tasksLocalDataSource.getTasks()
    }
}
```

모든 함수가 정리된 `DefaultTasksRepository` 의 전체 코드는 [다음 링크](https://github.com/googlecodelabs/android-testing/blob/end\_codelab\_3/app/src/main/java/com/example/android/architecture/blueprints/todoapp/data/source/DefaultTasksRepository.kt)에 있습니다.

### Write DataBindlingIdlingResource

Espresso를 사용해 데이터가 로드될 때까지 대기하도록 유휴 리소스를 작성했습니다. 그런 다음, 데이터바인딩에 대한 커스텀 유휴 리소스를 만듭니다.

Espresso는 데이터바인딩 라이브러와 [자동으로 작동하지 않으므로](https://github.com/android/android-test/issues/317) 해당 작업을 수행해야 합니다. 이는 데이터바인딩이 다른 메커니즘(the [Choreographer](https://developer.android.com/reference/android/view/Choreographer) class)을 사용하여 뷰 업데이트를 동기화하기 때문입니다. 그러므로 Espresso는 데이터바인딩을 통해 업데이트된 뷰가 언제 업데이트 되었는지 알 수 없습니다.

이러한 데이터바인딩 유휴 리소스 코드는 복잡하기 때문에, 코드를 제공하고 설명합니다.

1. androidTest > util 패키지 생성
2. android > uitl > DataBindingIdlingResource.kt 생성
3. 코드 작성하기

**DataBindingIdlingResource.kt**

```kotlin
class DataBindingIdlingResource : IdlingResource {
    // List of registered callbacks
    private val idlingCallbacks = mutableListOf<IdlingResource.ResourceCallback>()
    // Give it a unique id to work around an Espresso bug where you cannot register/unregister
    // an idling resource with the same name.
    private val id = UUID.randomUUID().toString()
    // Holds whether isIdle was called and the result was false. We track this to avoid calling
    // onTransitionToIdle callbacks if Espresso never thought we were idle in the first place.
    private var wasNotIdle = false

    lateinit var activity: FragmentActivity

    override fun getName() = "DataBinding $id"

    override fun isIdleNow(): Boolean {
        val idle = !getBindings().any { it.hasPendingBindings() }
        @Suppress("LiftReturnOrAssignment")
        if (idle) {
            if (wasNotIdle) {
                // Notify observers to avoid Espresso race detector.
                idlingCallbacks.forEach { it.onTransitionToIdle() }
            }
            wasNotIdle = false
        } else {
            wasNotIdle = true
            // Check next frame.
            activity.findViewById<View>(android.R.id.content).postDelayed({
                isIdleNow
            }, 16)
        }
        return idle
    }

    override fun registerIdleTransitionCallback(callback: IdlingResource.ResourceCallback) {
        idlingCallbacks.add(callback)
    }

    /**
     * Find all binding classes in all currently available fragments.
     */
    private fun getBindings(): List<ViewDataBinding> {
        val fragments = (activity as? FragmentActivity)
            ?.supportFragmentManager
            ?.fragments

        val bindings =
            fragments?.mapNotNull {
                it.view?.getBinding()
            } ?: emptyList()
        val childrenBindings = fragments?.flatMap { it.childFragmentManager.fragments }
            ?.mapNotNull { it.view?.getBinding() } ?: emptyList()

        return bindings + childrenBindings
    }
}

private fun View.getBinding(): ViewDataBinding? = DataBindingUtil.getBinding(this)

/**
 * Sets the activity from an [ActivityScenario] to be used from [DataBindingIdlingResource].
 */
fun DataBindingIdlingResource.monitorActivity(
    activityScenario: ActivityScenario<out FragmentActivity>
) {
    activityScenario.onActivity {
        this.activity = it
    }
}

/**
 * Sets the fragment from a [FragmentScenario] to be used from [DataBindingIdlingResource].
 */
fun DataBindingIdlingResource.monitorFragment(fragmentScenario: FragmentScenario<out Fragment>) {
    fragmentScenario.onFragment {
        this.activity = it.requireActivity()
    }
}
```

이 코드에서는 많은 작업을 하지만, 일반적으로 `ViewDataBinding`은 데이터바인딩 레이아웃을 사용할 때마다 생성됩니다. [`ViewDataBinding`](https://developer.android.com/reference/android/databinding/ViewDataBinding) 의 `hasPendingBindings` 함수는 데이터바인딩 라이브러리가 데이터의 변경 사항을 반영하기 위해 UI를 업데이트해야 하는지에 대한 여부를 보고합니다.

이 유휴 리소스는 `ViewDataBinding` 에 대해 pending 바인딩이 없는 경우에만 유휴 상태로 간주됩니다.

마지막으로, `DataBindingIdlingResource.monitorFragment` 와 `DataBindingIdlingResource.monitorActivity` 확장 함수는 각각 `FragmentScenario` 와 `ActivityScenario` 에서 수행됩니다. 그런 다음 액티비티를 찾아 `DataBindingIdlingResource` 와 연결하여 레이아웃의 상태를 추적할 수 있습니다.

테스트에서 반드시 이 두 가지 방법 중 하나를 호출해야 합니다. 그렇지 않으면 `DataBindingIdlingResource` 가 레이아웃에 대해 아무것도 알지 못합니다.

### Use Idling Resource in tests

지금까지 두 개의 유휴 리소스를 만들어 “사용 중(busy)” 또는 “유휴(idle)”로 올바르게 설정되었는지 확인했습니다. Espresso는 유휴 리소스가 등록될 때까지만 대기합니다. 따라서, 이제 테스트는 유휴 리소스를 등록하고 등록 취소해야 합니다. `TasksActivityTest` 에서 이 작업을 수행합니다.

1. TasksActivityTest.kt 열기
2. `private DataBindingIdlingResource` 를 인스턴스화 합니다.

**TasksActivityTest.kt**

```kotlin
// An idling resource that waits for Data Binding to have no pending bindings.
private val dataBindingIdlingResource = DataBindingIdlingResource()
```

1. `EspressoIdlingResource.countingIdlingResource` 와 `dataBindingIdlingResource` 를 등록하고 등록 취소하는 @Before 와 @After 함수를 만듭니다.

```kotlin
/**
* Idling resources tell Espresso that the app is idle or busy. This is needed when operations
* are not scheduled in the main Looper (for example when executed on a different thread).
*/
@Before
fun registerIdlingResource() {
  IdlingRegistry.getInstance().register(EspressoIdlingResource.countingIdlingResource)
  IdlingRegistry.getInstance().register(dataBindingIdlingResource)
}

/**
* Unregister your Idling Resource so it can be garbage collected and does not leak any memory.
*/
@After
fun unregisterIdlingResource() {
  IdlingRegistry.getInstance().unregister(EspressoIdlingResource.countingIdlingResource)
  IdlingRegistry.getInstance().unregister(dataBindingIdlingResource)
}
```

`countingIdlingResource` 와 `dataBindingIdlingResource` 모두 앱 코드를 모니터링하여 유휴 상태인지 아닌지에 대한 여부를 확인합니다. 이 리소스를 테스트에 등록하면, 두 개의 리소스 중 하나가 사용 중일 때, Espresso는 다음 명령으로 이동하기 전에 유휴 상태가 될 때까지 대기합니다. 즉, `countingIdlingResource` 의 카운트가 0보다 크거나 pending 중인 데이터바인딩 레이아웃이 있는 경우 Espresso가 대기합니다.

1. 액티비티 시나리오가 실행된 후에 `editTest()` 테스트를 업데이트하여, `monitorActivity` 를 사용해 액티비티를 `dataBindingIdlingResource` 연결합니다.

**TasksActivityTest.kt**

```kotlin
@Test
fun editTask() = runBlocking {
    repository.saveTask(Task("TITLE1", "DESCRIPTION"))

    // Start up Tasks screen.
    val activityScenario = ActivityScenario.launch(TasksActivity::class.java)
    dataBindingIdlingResource.monitorActivity(activityScenario) // LOOK HERE

    // Rest of test...
}
```

1. 이제 다시 5번 테스트를 실행해봅시다. 이제 더 이상 테스트가 실패하지 않는 것을 확인할 수 있습니다.

**전체 TasksActivityTest.kt 코드**

```kotlin
@RunWith(AndroidJUnit4::class)
@LargeTest
class TasksActivityTest {

    private lateinit var repository: TasksRepository

    // An idling resource that waits for Data Binding to have no pending bindings.
    private val dataBindingIdlingResource = DataBindingIdlingResource()

    @Before
    fun init() {
        repository =
            ServiceLocator.provideTasksRepository(
                getApplicationContext()
            )
        runBlocking {
            repository.deleteAllTasks()
        }
    }

    @After
    fun reset() {
        ServiceLocator.resetRepository()
    }

    /**
     * Idling resources tell Espresso that the app is idle or busy. This is needed when operations
     * are not scheduled in the main Looper (for example when executed on a different thread).
     */
    @Before
    fun registerIdlingResource() {
        IdlingRegistry.getInstance().register(EspressoIdlingResource.countingIdlingResource)
        IdlingRegistry.getInstance().register(dataBindingIdlingResource)
    }

    /**
     * Unregister your Idling Resource so it can be garbage collected and does not leak any memory.
     */
    @After
    fun unregisterIdlingResource() {
        IdlingRegistry.getInstance().unregister(EspressoIdlingResource.countingIdlingResource)
        IdlingRegistry.getInstance().unregister(dataBindingIdlingResource)
    }

    @Test
    fun editTask() = runBlocking {

        // Set initial state.
        repository.saveTask(Task("TITLE1", "DESCRIPTION"))
        
        // Start up Tasks screen.
        val activityScenario = ActivityScenario.launch(TasksActivity::class.java)
        dataBindingIdlingResource.monitorActivity(activityScenario)
        // Click on the task on the list and verify that all the data is correct.
        onView(withText("TITLE1")).perform(click())
        onView(withId(R.id.task_detail_title_text)).check(matches(withText("TITLE1")))
        onView(withId(R.id.task_detail_description_text)).check(matches(withText("DESCRIPTION")))
        onView(withId(R.id.task_detail_complete_checkbox)).check(matches(not(isChecked())))

        // Click on the edit button, edit, and save.
        onView(withId(R.id.edit_task_fab)).perform(click())
        onView(withId(R.id.add_task_title_edit_text)).perform(replaceText("NEW TITLE"))
        onView(withId(R.id.add_task_description_edit_text)).perform(replaceText("NEW DESCRIPTION"))
        onView(withId(R.id.save_task_fab)).perform(click())

        // Verify task is displayed on screen in the task list.
        onView(withText("NEW TITLE")).check(matches(isDisplayed()))
        // Verify previous task is not displayed.
        onView(withText("TITLE1")).check(doesNotExist())
        // Make sure the activity is closed before resetting the db.
        activityScenario.close()
    }

}
```

### Write your own test with idling resources

이제 배운 것을 토대로 TasksActivityTest.kt 파일의 `createOneTask_deleteTask()` 테스트 코드를 작성해봅시다.

```kotlin
@Test
fun createOneTask_deleteTask() {

    // 1. Start TasksActivity.
   
    // 2. Add an active task by clicking on the FAB and saving a new task.
    
    // 3. Open the new task in a details view.
    
    // 4. Click delete task in menu.

    // 5. Verify it was deleted.

    // 6. Make sure the activity is closed.
    
}
```
