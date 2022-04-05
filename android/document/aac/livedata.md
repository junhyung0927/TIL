# LiveData

LiveData는 observable 데이터 홀더 클래스이다. 일반적인 observable과 달리, Livedata는 생명 주기를 인식한다. 즉, 액티비티, 프래그먼트, 서비스 등 다른 앱 컴포넌트의 생명 주기를 고려한다. 다른 앱 컴포넌트 생명 주기를 통해 Livedata는 생명주기 상태에 active한 앱 컴포넌트 observers만 업데이트한다.

Observer 클래스로 표현되는 observer의 생명주기가 'Started'나 'Resume' 상태라면 Livedata는 observer를 active 상태로 간주한다. Livedata는 오직 active observer에게만 업데이트 정보를 알린다. LiveData 객체를 보기 위해 등록된 Inactive observer는 변경사항에 관한 알림을 받지 않는다.

LifecycleOwner 인터페이스를 구현하는 객체와 쌍으로 구성된 observer를 등록할 수 있다. 이 관계를 사용하면 observer에 대응되는 Lifecycle 객체의 상태가 'Destroyed'로 변경될 때, observer를 삭제할 수 있다. 특히, 액티비티와 프래그먼트가 Livedata 객체를 안전하게 관찰 할 수 있고, 액티비티와 프래그먼트의 생명주기가 끝나는 즉시, 구독 해제되어 메모리 누수를 걱정하지 않아도 되므로 유용하다.

### LiveData 장점

#### UI와 데이터 동기화

Livedata는 observer 패턴을 사용한다. Livedata는 생명 주기 상태가 변경될 때, Observer 객체에 알려준다. 코드를 통합하여 이러한 Observer 객체에 UI를 업데이트 할 수 있다. 앱 데이터가 변경될 때마다 UI를 업데이트 하는 대신, 변경이 발생할 때마다 observer가 UI를 업데이트 할 수 있다.

#### 메모리 누수 없음

observer는 Lifecycle 객체에 결합되어 있고 생명 주기가 끝나면 자동으로 삭제된다.

#### 'Stopped' 상태의 액티비티와 충돌이 일어나지 않음

백 스택에 있는 액티비티과 같은 경우, observer 생명주기가 Inactive 상태에 있다면 observer는 어떠한 Livedata 이벤트를 받지 않는다.

#### 생명 주기에 대한 추가적인 핸들링을 하지 않아도 됨

UI 컴포넌트는 오직 관련 데이터만 observer 하고, 관찰을 중지하거나 다시 시작하지 않는다. Livedata는 생명주기에 따른 observing을 자동으로 관리 해주기 때문에, UI 컴포넌트는 관련 있는 데이터를 observer 하기만 하면 된다.

#### 최신 데이터 유지

생명 주기가 Inactive되면, 다시 active 될 때 최신 데이터를 수신한다. 예를 들어, 백그라운드에 있었던 액티비티는 포그라운드로 돌아온 직후, 최신 데이터를 받게 된다.

#### 적절한 구성 변경

기기 회전과 같은 구성 변경으로 인해 액티비티 또는 프래그먼트가 다시 생성되면, 다시 사용 가능한 최신 데이터를 받게 된다.

#### 리소스 공유

앱에서 시스템 서비스를 공유 할 수 있도록, 싱글턴 패턴을 사용하면 Livedata 객체를 확장해서 시스템 서비스를 래핑(Wrap)할 수 있다. Livedata 객체가 시스템 서비스에 한 번 연결되면 리소스가 필요한 모든 observer가 Livedata를 볼 수 있다.

### Work with LiveData objects

Livedata 객체를 사용하여 구현하려면 다음 단계를 따르시오.

* certain 타입 데이터를 보유할 LiveData 인스턴스를 만들어야한다. 일반적으로 ViewModel 클래스 내에서 만든다.
* onChanged() 메서드를 정의하는 Observer 객체를 생성한다. Observer 객체는 Livedata 객체가 보유한 데이터가 변경 시 발생하는 작업을 제어한다. 일반적으로 액티비티나 프래그먼트 같은 UI 컨트롤러에 Observer 객체를 만든다.
* observe() 메서드를 사용하여 Livedata 객체에 Observer 객체를 연결한다. observe() 메서드는 LifecycleOwner 객체를 사용한다. Observer 객체가 Livedata 객체를 구독해서 변경사항에 관한 알림을 받는다. 일반적으로 액티비티나 프래그먼트와 같이 UI 컨트롤러에 Observer 객체를 연결한다.

\<aside> 🗨️ observeForever(Observer) 메서드를 사용해서 연결된 LifecycleOwner 객체가 없는 observer를 등록 할 수 있다. 이 경우 observer는 항상 active로 간주되며, 항상 수정에 관한 알림을 받는다. removeObserver(Observer) 메서드를 호출하여 이러한 observer를 삭제 할 수 있다.

\</aside>

#### Create LiveData objects

Livedata는 Collections를 구현하는 List와 같은 객체를 비롯하여, 모든 데이터와 함께 사용할 수 있는 래퍼(Wrapper)이다. Livedata 객체는 일반적으로 ViewModel 객체 내에 저장되며, 다음 예에서 보는 것과 같이 getter 메서드를 통해 엑세스된다.

```kotlin
class NameViewModel : ViewModel() {

    // Create a LiveData with a String
    val currentName: **MutableLiveData<String>** by lazy {
        **MutableLiveData<String>()**
    }

    // Rest of the ViewModel...
}
```

처음 생성 시, LiveData 객체의 데이터가 설정되지 않는다.

\<aside> 🗨️ 액티비티 또는 프래그먼트와 달리, ViewModel 객체의 UI를 업데이트하는 LiveData 객체를 저장해야 하며, 그 이유는 다음과 같다.

1. 액티비티와 프래그먼트의 크기가 지나치게 커지지 않게 하기 위해서이다. 요즘은 이러한 UI 컨트롤러가 데이터 표시를 담당하지만, 데이터 상태를 보유하지는 않는다.
2. LiveData 인스턴스를 특정 액티비티나 프래그먼트 인스턴스에서 분리하고 구성 변경에도 LiveData 객체가 유지되도록 하기 위해서다.

\</aside>

#### Observe LiveData objects

대부분의 경우, 앱 컴포넌트의 onCreate() 메서드는 LiveData 객체 Observing을 시작하기 적합한 장소이며, 그 이유는 다음과 같다.

* 시스템이 액티비티나 프래그먼트의 onResume() 메서드에서 중복 호출을 하지 않도록 하기 위해서이다.
* 액티비티나 프래그먼트에 active 상태가 되는 즉시, 표시할 수 있는 데이터가 포함되도록 하기 위해서이다. 앱 구성요소는 'Started' 상태가 되는 즉시, observing 하고 있던 LiveData 객체에서 최신 데이터를 수신한다. 이는 observe할 LiveData가 설정된 경우만 발생한다.

일반적으로, LiveData는 데이터가 변경될 때, active된 observer에게만 업데이트 내용을 전달한다. 예외로, observer가 Inactive 상태에서 active 상태로 변경될 때에도, observer가 업데이트를 받는다. 또한 observer가 Inactive에서 active 상태로 두 번 변경되면, 마지막으로 active 상태가 된 이후 값이 변경된 경우에만 업데이트를 받는다.

다음 예제 코드는 LiveData 객체 observing을 시작하는 방법을 보여준다.

```kotlin
class NameActivity : AppCompatActivity() {

    private val model: NameViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Other code to setup the activity...

        // Create the observer which updates the UI.
        val nameObserver = Observer<String> { newName ->
            // Update the UI, in this case, a TextView.
            nameTextView.text = newName
        }

				// Observe된 livedata는 해당 액티비티의 LifecycleOwner와 observer(nameObserver)로 전달된다.
        model.currentName.observe(this, nameObserver)
    }
}
```

nameObserver를 매개변수로 전달하여 observe()를 호출하면, onChanged()가 즉시 호출되어 mCurrentName에 저장된 최신 값을 제공한다. LiveData 객체가 mCurrentName에 값을 설정하지 않았다면 onChanged()는 호출되지 않는다.

#### Update LiveData objects

LiveData에는 저장된 데이터를 업데이트 해주는 (공개적으로 제공되는) 메서드가 없다. MutableLiveData 클래스는 setValue(T) 및 postValue(T) 메서드를 공개 메서드로 노출하며, LiveData 객체에 저장된 값을 수정하려면 이러한 메서드를 사용하면 된다. 일반적으로 MutableLiveData는 ViewModel에서 사용되며, ViewModel은 변경이 불가능한(immutable) LiveData 객체만 observer에게 노출한다.

*   setValue vs postValue

    LiveData의 setValue()와 postValue()는 둘 다 접근 제한자가 protected로 되어 있기 때문에, 외부에서 LiveData의 값을 변경해 줄 수는 없다. 외부에서 값을 변경해주기 위해서는 LiveData를 상속받은 MutableLiveData를 사용한다.

    **setValue()**

    setValue()는 **메인 스레드**에서 LiveData의 값을 변경해준다. 메인 쓰레드에서 바로 값을 변경해주기 때문에 setValue() 함수를 호출한 뒤, 바로 밑에서 getValue() 함수로 값을 읽어오면 변경된 값을 가져올 수 있다.

    setValue()는 메인 쓰레드에서 값을 dispatch 하기 때문에, 백그라운드에서 setValue()를 호출한다면 오류가 발생한다.

    setValue()가 동작하지 않는다면, 해당 함수가 호출되는 스레드가 메인 스레드인지 체크해야 한다.

    **postValue()**

    postValue()의 설명으로 구글 공식 문서에는 아래와 같이 나와있다.

    > If you need set a value from a background thread, you can use postValue(Object) Posts a task to a main thread to set the given value. If you called this method multiple times before a main thread executed a posted task, only the last value would be dispatched.

    postValue()는 setValue()와 다르게 **백그라운드**에서 값을 변경한다. 백그라운드 스레드에서 동작하다가 메인 스레드에 값을 post 하는 방식으로 사용된다. 함수 내부적으로는 아래와 같은 코드가 실행된다.

    ```kotlin
    new Handler(Looper.mainLooper()).post(() -> setValue())
    ```

    메인 스레드에 적용되기 전에 postValue()가 여러 번 호출된다면, 모든 값이 적용되는 것이 아니라, 가장 최신의 값이 적용된다.

    따라서 postValue()를 호출한 뒤, 바로 getValue()로 값을 읽으라고 한다면 변경된 값을 읽어오지 못 할 가능성이 높다. Handler()를 통해 메인 스레드에 값이 전달되기 전에 getValue()를 호출하기 때문이다.

observer 관계를 설정한 후에는, 아래 예와 같이 사용자가 버튼을 누를 때, 모든 observer를 트리거하는 LiveData 객체의 값을 업데이트 할 수 있다.

```kotlin
button.setOnClickListener {
    val anotherName = "Jun Hyung"
    model.currentName.setValue(anotherName)
}
```

위의 예에서, setValue(T)를 호출하면 observer는 "Jun Hyung" 값과 함께 onChanged() 메서드를 호출한다. 해당 예에서는 버튼이 눌렸을 때를 보여주지만, setValue(),postValue()는 네트워크 요청 또는 데이터베이스 로드 완료에 응답하는 등의 다양한 이유로 mName을 업데이트하기 위해 호출될 수 있다. 모든 경우에 setValue()또는 postValue()를 호출하면 observer가 트리거되고 UI가 업데이트 된다.

#### Use LiveData with Room

Room 영속성 라이브러리는 observable한 쿼리를 지원하며, 이 쿼리는 LiveData 객체를 반환한다. observable한 쿼리는 DAO(Database Access Object)의 일부로 작성된다.

데이터베이스가 업데이트될 때 Room에서는 LiveData 객체를 업데이트 하는 데 필요한 모든 코드를 생성한다. 생성된 코드는 필요할 때 백그라운드 스레드에서 비동적으로 쿼리를 실행한다. 이 패턴은 UI에 표시된 데이터와 데이터베이스에 저장된 데이터의 동기화를 유지하는데 유용한다.

#### LiveData와 함께 코루틴 사용

LiveData에는 Kotlin Coroutines을 사용할 수 있다. 자세한 내용은 ['Use Kotlin coroutines with Android Architecture Components'](https://developer.android.com/topic/libraries/architecture/coroutines)를 참고해라.

### Extend LiveData

observer의 생명 주기가 'Started', 'Resume' 상태일 때, LiveData는 observer를 active 상태로 간주한다. 즉, LiveData가 observer를 호출하여, 불필요한 Observer Event가 일어나는 경우도 존재한다.

다음 예제 코드는 LiveData 클래스를 확장하는 방법을 보여준다.

```jsx
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }
}
```

* **onActive()** 메서드는 LiveData 객체에 active 상태의 observer가 있을 때 호출된다. 즉, 이 메서드에서 stock price의 업데이트 observing을 시작해야한다.
* **onInactive()** 메서드는 LiveData 객체에 active 상태의 observer가 없을 때 호출된다. 수신 대기 중인 observer가 없으므로 StockManager 서비스에 연결된 상태를 유지할 필요가 없다.
* **setValue(T)** 메서드는 LiveData 인스턴스의 값을 업데이트하고 모든 active 상태의 observer에게 변경사항을 알려준다.

다음은 StockLiveData 클래스를 사용한 예제이다.

```jsx
public class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        val myPriceListener: LiveData<BigDecimal> = ...
        myPriceListener.observe(viewLifecycleOwner, Observer<BigDecimal> 
						{ price: BigDecimal? ->
            // Update the UI.
		        })
    }
}
```

observe() 메서드는 프래그먼트 뷰와 연결된 LifecycleOwner를 첫 번째 인수로 전달한다. 즉, observer는 owner와 연결된 Lifecycle 객체에 결합하는 것이고, 그 의미는 다음과 같다.

* Lifecycle 객체가 active 상태가 아니면 값이 변경되더라도 observer가 호출되지 않는다.
* Lifecycle 객체가 제거된 후 observer는 자동으로 삭제된다.

LiveData 객체가 생명주기를 인식한다는 것은, 여러 액티비티, 프래그먼트, 서비스 간에 객체를 공유할 수 있다는 의미이다. 이를 간단하게 유지하려면 다음과 같이 LiveData 클래스를 싱글톤으로 구현하면 된다.

```jsx
class StockLiveData(symbol: String) : LiveData<BigDecimal>() {
    private val stockManager: StockManager = StockManager(symbol)

    private val listener = { price: BigDecimal ->
        value = price
    }

    override fun onActive() {
        stockManager.requestPriceUpdates(listener)
    }

    override fun onInactive() {
        stockManager.removeUpdates(listener)
    }

    **companion object {
        private lateinit var sInstance: StockLiveData

        @MainThread
        fun get(symbol: String): StockLiveData {
            sInstance = 
									if (::sInstance.isInitialized) sInstance 
									else StockLiveData(symbol)
            return sInstance
        }
    }**
}
```

이렇게 싱글톤으로 구현된 것을, 다음과 같이 프래그먼트에서 클래스를 사용할 수 있다.

```jsx
class MyFragment : Fragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        StockLiveData.get(symbol).observe(viewLifecycleOwner, Observer<BigDecimal> 
						{ price: BigDecimal? ->
            // Update the UI.
		        })

    }
```

*   SingleLiveEvent

    LiveData를 사용하다보면, View의 재활성화(Start, Resume 상태로 재진입)가 되면서 LiveData가 observe를 호출하여, 불필요한 Observer Event 까지 일어나는 경우가 존재한다.

    이를 방지하기 위해 기존 LiveData를 상속하여 만들어낸 것이 SingleLiveEvent이다.

    postValue나 setValue, call 등을 통하여 setValue 함수를 거쳐야만이 Observer을 통하여 데이터를 전달 할 수 있으며, 이는 Configuration Changed 등의 LifecycleOwner의 재활성화 상태가 와도, 원치 않는 이벤트가 일어나는 것을 방지 할 수 있도록 도와준다.

    ```jsx
    import android.util.Log
    import androidx.annotation.MainThread
    import androidx.lifecycle.LifecycleOwner
    import androidx.lifecycle.MutableLiveData
    import androidx.lifecycle.Observer
    import java.util.concurrent.atomic.AtomicBoolean

    class SingleLiveEvent<T> : MutableLiveData<T>() {
        /**
        * 멀티쓰레딩 환경에서 동시성을 보장하는 AtomicBoolean.
        * false로 초기화되어 있음
        */
        private val pending = AtomicBoolean(false)
        
        /**
        * View(Activity or Fragment 등 LifeCycleOwner)가 활성화 상태가 되거나
        * setValue로 값이 바뀌었을 때 호출되는 observe 함수.
        * pending.compareAndSet(true, false)라는 것은, 위의 pending 변수가
        * true면 if문 내의 로직을 처리하고 false로 바꾼다는 것이다.
        *
        * 아래의 setValue를 통해서만 pending값이 true로 바뀌기 때문에,
        * Configuration Changed가 일어나도 pending값은 false이기 때문에 observe가
        * 데이터를 전달하지 않는다.
        */
        @MainThread
        override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
            if (hasActiveObservers()) {
                Log.w(TAG, "Multiple observers registered but only one will be notified of changes.")
            }
            // Observe the internal MutableLiveData
            super.observe(owner, Observer { t ->
                if (pending.compareAndSet(true, false)) {
                    observer.onChanged(t)
                }
            })
        }

        /**
        * LiveData로써 들고있는 데이터의 값을 변경하는 함수.
        * 여기서는 pending(AtomicBoolean)의 변수는 true로 바꾸어 
        * observe내의 if문을 처리할 수 있도록 하였음.
        */
        @MainThread
        override fun setValue(t: T?) {
            pending.set(true)
            super.setValue(t)
        }
        
        /**
        * 데이터의 속성을 지정해주지 않아도 call만으로 setValue를 호출 가능
        */
        @MainThread
        fun call() {
            value = null
        }
        
        companion object {
            private val TAG = "SingleLiveEvent"
        }
    }
    ```

    [https://zladnrms.tistory.com/146](https://zladnrms.tistory.com/146)

### Transform LiveData

observer에게 LiveData 객체를 전달하기 전에 객체에 저장된 값을 변경하고 싶거나, 다른 객체의 값에 따라 다른 LiveData 인스턴스를 반환해야 하는 경우가 있다.

Lifecycle 패키지는 이러한 시나리오를 지원하는 도우미 메서드가 포함된 Transformations 클래스를 제공한다.

**Transformations.map()**

LiveData 객체에 저장된 값에 함수를 적용하고, 그 결과를 다운스트림으로 전파한다.

```kotlin
val userLiveData: LiveData<User> = UserLiveData()
val userName: LiveData<String> = Transformations.map(userLiveData) {
    user -> "${user.name} ${user.lastName}"
}
//LiveData -> Value(ex: String ...)
```

**Transformations.switchMap()**

map()과 마찬가지로 LiveData 객체에 저장된 값에 함수를 적용하고 결과를 unwraps 하여 다운스트림으로 전달한다.

```kotlin
private fun getUser(id: String): LiveData<User> {
  ...
}

val userId: LiveData<String> = ...
val user = Transformations.switchMap(userId) { id -> getUser(id) }
//user가 observing을 하지 않는 상태면 아무 값도 변경되지 않고, observing이 시작되는 경우만 변경이 이루어지기 때문에 변환이 느리게 계산된다.
//LiveData -> LiveData
```

변환 메서드를 사용하여 observer의 생명 주기 전반에 걸쳐 정보를 전달할 수 있다. observer가 반환된 LiveData 객체를 관찰하고 있지 않다면, 변환은 계산되지 않는다. 변환은 느리게 계산되기 때문에, 생명 주기 관련 동작은 추가적인 명시적 호출이나 종속 항목 없이 암시적으로 전달된다.

ViewModel 객체 내에 Lifecycle 객체가 필요하다고 생각되면, 변환이 더 좋은 해결 방법이 될 수 있다. 예를 들어, 주소를 받아서 그 주소의 우편번호를 반환하는 UI 구성요소가 있는 경우, 다음 예제 코드와 같이, 이 구성요소를 위한 기본 ViewModel을 구현할 수 있다.

```jsx
class MyViewModel(private val repository: PostalCodeRepository) : ViewModel() {

    private fun getPostalCode(address: String): LiveData<String> {
        // DON'T DO THIS
        return repository.getPostCode(address)
    }
}
```

### RFC

[https://developer.android.com/topic/libraries/architecture/livedata](https://developer.android.com/topic/libraries/architecture/livedata)
