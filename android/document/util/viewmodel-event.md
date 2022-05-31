# ViewModel Event 처리

View(Activity, Fragment)와 ViewModel의 통신 방법 중 LiveData를 사용하여 Observerable 하게 사용하는 것이다.

View는 LiveData의 변경 사항을 구독하고 이에 반응하게 된다. 따라서 화면에 지속적으로 표시되는 데이터에 적합하다.

하지만, navigation, 다이얼로그, 버튼과 같은 일부 데이터는 한 번만 사용해야 하는데, 만약 지속적으로 이벤트를 발생시킨다면 부적합하다.

라이브러리나 Architecture Components를 사용해 이 문제를 해결하는 대신, 디자인 문제에 직면하게 된다. 그러므로 이벤트를 상태의 일부로 취급하는 것을 권장한다.

## SingleLiveEvent

SingleLiveEvent 클래스는 특정 시나리오에 적합한 솔루션으로 Sample로 작성되었다. 한 번만 업데이트를 보내는 LiveData이다.

```kotlin
class TestViewModel: ViewModel {
	private val _navigateToDetails = SingleLiveEvent<Any>()
	
	val navigateToDetails: LiveData<Any>
		get() = _navigateToDetails

	fun clickOnButton(){
		_navigateToDetails.call()
	}
}
```

```kotlin
testViewModel.navigateToDetails.observe(this, Observer{
	...
}) 
```

```kotlin
class SingleLiveEvent<T> : MutableLiveData<T>() {

    private val pending = AtomicBoolean(false)
    private val tag = "VictoryWoo"

		// 2. 내부에 등록된 Observer가 변경에 대한 알림을 받고 observe 함수를 호출한다.
		// pending이 true일 경우, pending을 false로 바꿔준다.
		// 그리고 이벤트가 호출되었다고 알려준다.
    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        if (hasActiveObservers()) {
            Log.w(tag, "Multiple observers registered but only one will be notified of changes.")
        }

        super.observe(owner, Observer { t ->
            if (pending.compareAndSet(true, false)) {
                observer.onChanged(t)
            }
        })
    }

		// 1. LiveData로써 들고 있는 데이터의 값을 변경하는 함수.
		// 값이 변경되면 false였던 pending 변수는 true로 바뀌어 
		// observer가 알아차리고 observe를 호출하여 if문을 처리할 수 있도록 하였다.
    @MainThread
    override fun setValue(t: T?) {
        pending.set(true)
        super.setValue(t)
    }

    @MainThread
    fun call() {
        value = null
    }
}
```

#### 문제점

SingleLiveEvent의 문제점은 내부에서 여러번 호출되는 것을 막고 있기 때문에, 하나의 관찰자로 제한된다. 실수로 둘 이상을 추가하면 하나만 호출되어 어떤 것을 보장하지 않는다.

![](<../../../.gitbook/assets/Untitled (6).png>)

## EventWrapper

**EventWrapper**는 사용자가 getContentIfNotHandled() 함수와 peekContent() 함수를 선택하여 명시적으로 사용할 수 있는 장점을 가지고 있다.

**getContentIfNotHandled**를 사용하면 중복적인 이벤트 처리를 방지할 수 있고, **peekContent**를 사용하면 이벤트 처리 여부에 상관없이 값을 반환한다.

```kotlin
open class Event<out T>(private val content: T) {
    var hasBeenHandled = false
        private set

    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) { // 이벤트가 이미 처리 되었다면 null 반환
            null
        } else {
            hasBeenHandled = true // 이벤트가 처리되었다고 표시한 후에 값을 반환
            content
        }
    }

    //이벤트의 처리 여부에 상관 없이 값을 반환
    fun peekContent(): T = content
}
```

```kotlin
private val _onMyEvent = MutableLiveData<Event<Int>>()
    val onMyEvent: LiveData<Event<Int>>
        get() = _onMyEvent

    fun callOnMyEvent(item: Int) {
        _onMyEvent.value = Event(item)
    }
```

```kotlin
//EvnetObserver 사용하지 않을 때
myViewModel.onMyEvent.observe(viewLifecycleOwner, Observer{
		it.getContentIfNotHandled()?.let{
			...
		}
})
////EvnetObserver 사용할 때
myViewModel.onMyEvent.observe(viewLifecycleOwner, EvevtObserver{
		...
})
```

```kotlin
class EventObserver<T>(private val onEventUnhandledContent: (T) -> Unit) : Observer<Event<T>> {
    override fun onChanged(event: Event<T>?) {
        event?.getContentIfNotHandled()?.let { value ->
            onEventUnhandledContent(value)
        }
    }
}
```

SingleLiveEvent는 Observing하는 과정에서 일회성으로 만들기 때문에 하나의 옵저버만 값의 변경을 받을 수 있지만, EventWrapper 방식은 Event가 이를 제어하기 때문에 여러 개의 옵저버를 등록해도 모두 값의 변경을 받을 수 있다.

단, **getContentIfNotHandled()** 메서드는 하나의 옵저버에서만 사용할 수 있고, 나머지는 \*\*peekCount()\*\*로 값을 받아야 한다.&#x20;

사용자가 getContentIfNotHandled() 또는 peekCount()를 사용하여 의도를 지정하는 장점을 가지고 있다. 이벤트를 상태의 일부로 모델링하여, 단순히 소비되었거나 사용하지 않은 메시지로 간주한다.

![](<../../../.gitbook/assets/Untitled (9).png>)

## SharedFlow

![](<../../../.gitbook/assets/Untitled (7).png>)

ViewModel은 원칙적으로 Presenters 레이어에 위치하고 있으므로 **특정 플랫폼과 관계가 없어야 한다.** 즉, android 프레임워크와 같은 `import android.xxx` 인 코드가 없도록 유지 되어야 한다. 이와 같은 글은 안드로이드 공식 블로그에 나와있다.

💡 _**ViewModels and LiveData: Patterns + AntiPatterns**_

![](<../../../.gitbook/assets/Untitled (8).png>)

이상적으로, ViewModel은 Android에 대해서 아무것도 알지 않아야 한다. 따라서 테스트 가능성, leak 안정성, 모듈성이 향상된다. 일반적인 규칙은 ViewModel이 `imports android.*` 가 없는지 확인하는 것이다. presenter 레이어에서도 동일한 규칙을 가지고 있다. _**Don't let ViewModels (and Presenters) know about Android framework classes**_

{% embed url="https://medium.com/androiddevelopers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54" %}

위와 같은 문제를 해결하기 위해 코루틴을 사용하여 LiveData(안드로이드 프레임워크) 종속성으로부터 벗어날 수 있게 된다.

**코루틴의 Flow를 사용해 LiveData → StateFlow, SingleLiveData → SharedFlow로 대체할 수 있다.**

기존에 SingleLiveData를 SharedFlow로 변경하고 observe 대신 collect하는 코드로 변경한다.

#### **📚 이전**

```kotlin
// VieWModel
private val _showToastEvent = MutableSingleLiveData<String>()
val showToastEvent: SingleLiveData<String> = _showToastEvent

// UI
viewModel.showToastEvent.observe { text ->
    // TODO
}
```

#### 👏 이후

```kotlin
// ViewModel
private val _showToastEvent = **MutableSharedFlow**<String>()
val showToastEvent = _showToastEvent.**asSharedFlow**()
// UI
lifecycleScope.launch {
    viewModel.showToastEvent.**collect** { text ->
        // TODO
    }
}
```

## SharedFlow + Sealed class

처리해야 할 이벤트가 다수라면, 기존의 방법을 사용하게 되면 해당하는 이벤트의 개수만큼 SharedFlow를 만들어줬어야 했다. 또한 각각의 이벤트를 collect하는 코드도 따로 작성해줘야 했다.

```kotlin
private val _showToastEvent = MutableSharedFlow<String>()
val showToastEvent = _showToastEvent.asSharedFlow()

private val _firstEvent = MutableSharedFlow<String>()
val firstEvent = _firstEvent.asSharedFlow()

private val _secondEvent = MutableSharedFlow<String>()
val secondEvent = _secondEvent.asSharedFlow()
```

이러한 중복되는 코드를 방지하기 위해 Event를 전파하는 단 1개의 eventFlow만을 만들고, 해당 이벤트를 Sealed class 형태로 만들어서 상황에 맞게 처리하도록 개선하였다.

```kotlin
lifecycleScope.launch {
    viewModel.eventFlow.collect { event -> handleEvent(event) }
}
...
private fun handleEvent(event: Event) = when (event) {
    is Event.ShowToast -> // TODO
    is Event.Aaa -> // TODO
    is Event.Bbb -> // TODO
}
```

UI에서는 1개의 eventFlow를 collect하고 Event 유형에 맞게 처리하도록 분리하면 된다.

## SharedFlow + Sealed class + Lifecycle

위와 같은 방식도 문제가 되는 경우도 존재한다.

1. ViewModel에서 서버와 통신하면서 위치 데이터를 주기적으로 emit
2. UI에서는 위치데이터가 변경되는것을 감지하고 있다가 변경될때마다 화면에 새로 그림
3. 홈버튼을 눌러서 화면이 Background에 있을때는 화면에 새로 그릴 필요 없음
4. UI가 안보이고 있을때는 데이터를 observe하고 있을 필요 없음

이러한 문제를 해결하기 위한 원시적인 방법은 onStart()에서 collect를 시작하고, onStop()에서 cancel 하는 것이다.

```kotlin
class LocationActivity : AppCompatActivity() {

    // Coroutine listening for Locations
    private var locationUpdatesJob: Job? = null

    override fun onStart() {
        super.onStart()
        locationUpdatesJob = lifecycleScope.launch {
            viewModel.locationFlow().collect {
                // New location! Update the map
            } 
        }
    }

    override fun onStop() {
        // Stop collecting when the View goes to the background
        locationUpdatesJob?.cancel()
        super.onStop()
    }
}
```

Lifecycle에서 제공하는 `repeatOnLifecycle()` 함수를 사용하면 더욱 편리하게 관리할 수 있다.

`repeatOnLifecycle()` 함수를 사용하면 번거롭게 nStart() / onStop()에서 작성을 하지 않아도 Lifecycle에 맞게 collect/cancel을 반복 해준다.

```kotlin
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Create a new coroutine from the lifecycleScope
        // since repeatOnLifecycle is a suspend function
        lifecycleScope.launch {
            // Suspend the coroutine until the lifecycle is DESTROYED.
            // repeatOnLifecycle launches the block in a new coroutine every time the 
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Safely collect from locations when the lifecycle is STARTED
                // and stop collecting when the lifecycle is STOPPED
                someLocationProvider.locations.collect {
                    // New location! Update the map
                }
            }
            // Note: at this point, the lifecycle is DESTROYED!
        }
    }
}
```

\<aside> 💡 STARTED: onStart() 이후나 onPause() 직전에 변경

\</aside>

_헤이딜러_ 회사에서는 repeatOnStarted()라는 확장함수를 만들어서 더욱 간결하게 사용될 수 있도록 하였다.

```kotlin
fun LifecycleOwner.repeatOnStarted(block: suspend CoroutineScope.() -> Unit) {
    lifecycleScope.launch {
        lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED, block)
    }
}
```

```kotlin
@AndroidEntryPoint
class Step5Activity :
    BaseActivity<ActivityStep5Binding, Step5ViewModel>(R.layout.activity_step5) {

    override val viewModel: Step5ViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        /*
        lifecycleScope.launch {
            lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.eventFlow.collect { event -> handleEvent(event) }
            }
        }
        */
        repeatOnStarted {
            viewModel.eventFlow.collect { event -> handleEvent(event) }
        }

    }

    private fun handleEvent(event: Event) = when (event) {
        is Event.ShowToast -> // TODO
        is Event.Aaa -> // TODO
        is Event.Bbb -> // TODO
    }
}
```

## EventFlow + Sealed class + Lifecycle

```kotlin
목록에서 특정 item을 선택하면, 서버에서 상태를 체크한 뒤 상세화면을 실행하는 시나리오
```

1. 만약, 특정 item을 선택한 뒤
2. 서버 상태 체크를 끝나기 전 홈버튼을 눌러 앱이 백그라운드로 내려갔다면
3. 상세화면을 실행하는 event를 emit해도 onStop() 상태이기 때문에 이벤트를 받지 못함

위와 같은 문제가 발생하여 event를 observe하고 있는 곳이 아무데도 없다면, 해당 event는 유실되어서 event가 발생했던 것을 나중에라도 알 수 없게 된다.

그래서 Event가 발생했을 때 이를 캐시하고 있다가 event의 consume 여부에 따라서 새로 구독하고 observer가 있을 때 event를 전파할 지 여부를 결정해주는 별도의 EventFlow가 필요하다.

```kotlin
interface EventFlow<out T> : Flow<T> {

    companion object {

        const val DEFAULT_REPLAY: Int = 3
    }
}

interface MutableEventFlow<T> : EventFlow<T>, FlowCollector<T>

@Suppress("FunctionName")
fun <T> MutableEventFlow(
    replay: Int = EventFlow.DEFAULT_REPLAY
): MutableEventFlow<T> = EventFlowImpl(replay)

fun <T> MutableEventFlow<T>.asEventFlow(): EventFlow<T> = ReadOnlyEventFlow(this)

private class ReadOnlyEventFlow<T>(flow: EventFlow<T>) : EventFlow<T> by flow

private class EventFlowImpl<T>(
    replay: Int
) : MutableEventFlow<T> {

    private val flow: MutableSharedFlow<EventFlowSlot<T>> = MutableSharedFlow(replay = replay)

    @InternalCoroutinesApi
    override suspend fun collect(collector: FlowCollector<T>) = flow
        .collect { slot ->
            if (!slot.markConsumed()) {
                collector.emit(slot.value)
            }
        }

    override suspend fun emit(value: T) {
        flow.emit(EventFlowSlot(value))
    }
}

private class EventFlowSlot<T>(val value: T) {

    private val consumed: AtomicBoolean = AtomicBoolean(false)

    fun markConsumed(): Boolean = consumed.getAndSet(true)
}
```

* consume 되지 않은 Event 1개를 캐시하고 있다는 코드

```kotlin
private val _eventFlow = MutableEventFlow<Event>()
val eventFlow = _eventFlow.asEventFlow()
```

### RFC

[https://woovictory.github.io/2020/07/08/Android-SingleLiveEventToEventWrapper/](https://woovictory.github.io/2020/07/08/Android-SingleLiveEventToEventWrapper/)

[SingleLiveEvent](https://www.notion.so/SingleLiveEvent-5944ac1fba8c4b3f9cc3859d59c41520)

[https://dunkey2615.tistory.com/222](https://dunkey2615.tistory.com/222)

[https://medium.com/prnd/mvvm의-viewmodel에서-이벤트를-처리하는-방법-6가지-31bb183a88ce](https://medium.com/prnd/mvvm%EC%9D%98-viewmodel%EC%97%90%EC%84%9C-%EC%9D%B4%EB%B2%A4%ED%8A%B8%EB%A5%BC-%EC%B2%98%EB%A6%AC%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95-6%EA%B0%80%EC%A7%80-31bb183a88ce)

ViewModel 이벤트 처리 관련 글

* [LiveData with SnackBar, Navigation and other events (the SingleLiveEvent case)](https://medium.com/androiddevelopers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)
* [LiveData with single events](https://proandroiddev.com/livedata-with-single-events-2395dea972a8)
* [Android SingleLiveEvent Redux with Kotlin Flow](https://proandroiddev.com/android-singleliveevent-redux-with-kotlin-flow-b755c70bb055)
* [A safer way to collect flows from Android UIs](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)
* [repeatOnLifecycle API design story](https://medium.com/androiddevelopers/repeatonlifecycle-api-design-story-8670d1a7d333)

[https://myungpyo.medium.com/stateflow-와-sharedflow-32fdb49f9a32](https://myungpyo.medium.com/stateflow-%EC%99%80-sharedflow-32fdb49f9a32)
