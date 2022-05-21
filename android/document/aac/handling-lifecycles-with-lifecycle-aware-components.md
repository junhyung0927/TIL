# Handling Lifecycles with Lifecycle-Aware Components



생명 주기 인식 컴포넌트는 액티비티 및 프래그먼트와 같은 다른 구성요소의 생명 주기 상태 변경에 따라 작업을 실행한다. 이러한 컴포넌트를 사용하면 잘 구성된 경량 코드를 만들어 보다 쉽게 유지를 할 수 있다.

Android Architecture Components에서는 lifecycle을 다루기 위한 androidx.lifecycle 패키지를 제공한다. androidx.lifecycle 패키지는 액티비티와 프래그먼트의 생명 주기에 따른 동작을 정의할 수 있는 클래스와 인터페이스를 제공한다.

화면에 기기 위치를 표시하는 액티비티를 예제로 들어보자. 일반적인 구현은 다음과 같다.

```kotlin
internal class MyLocationListener(
            private val context: Context,
            private val callback: (Location) -> Unit
    ) {

        fun start() {
            // connect to system location service
        }

        fun stop() {
            // disconnect from system location service
        }
    }

    class MyActivity : AppCompatActivity() {
        private lateinit var myLocationListener: MyLocationListener

        override fun onCreate(...) {
            myLocationListener = MyLocationListener(this) { location ->
                // update UI
            }
        }

        public override fun onStart() {
            super.onStart()
            myLocationListener.start()
						// 액티비티 생명 주기의 대응해야 하는 기타 컴포넌트 관리
        }

        public override fun onStop() {
            super.onStop()
            myLocationListener.stop()
						// 액티비티 생명 주기의 대응해야 하는 기타 컴포넌트 관리
        }
    }
```

위 코드 예제는 생명 주기의 현재 상태에 따라 UI 및 다른 컴포넌트를 관리하는 호출이 너무 많이 발생하게 된다. 여러 컴포넌트를 관리하면 onStart() 및 onStop()과 같은 생명 주기 메서드에 많은 양의 코드를 배치하게 되어 유지하기 어려워진다. 예를 들어, onStart()는 컴포넌트 확인과 같은 장기 실행 작업을 진행해야하는 경우가 많다. 이로 인해 액티비티가 종료된 후, location.start()가 호출되면 location.stop()이 호출되지 않으므로 연결이 끊어지지 않을 수도 있다.

### Lifecycle

Lifecycle은 액티비티나 프래그먼트와 같은 컴포넌트의 생명 주기 상태 관련 정보를 포함하며, 다른 객체가 이 상태를 observe 할 수 있게 하는 클래스다. 내부에 두 가지의 enum을 갖는데 아래와 같다.

#### Lifecycle.Event

@OnLifecycleEvent 애노테이션은 다음 생명주기 이벤트에 반응할 수 있다.

* ON\_ANY
  * ON\_ANY 이벤트에 할당된 메서드는 모든 생명 주기 이벤트에 대해 호출된다.
* ON\_CREATE
* ON\_DESTROY
* ON\_PAUSE
* ON\_RESUME
* ON\_START
* ON\_STOP

#### Lifecycle.State

* CREATED: onCreate() 이후나 onStop() 직전에 변경
* DESTROYED: onDestory()가 불리기 직전에 변경
* INITIALIZED: onCreate()가 불리기 직전에 변경
* RESUMED: onResume()이 불린 이후에 변경
* STARTED: onStart() 이후나, onPause() 직전에 변경

![](<../../../.gitbook/assets/Untitled (3).png>)

아래와 같이 Lifecycle 클래스의 addObserver() 메서드를 호출하고 다음과 같이 observer의 인스턴스를 전달하여 observer를 추가 할 수 있다.

```kotlin
class MyObserver : LifecycleObserver {
				
        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        fun connectListener() {
            ...
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        fun disconnectListener() {
            ...
        }
    }
    **myLifecycleOwner.getLifecycle().addObserver(MyObserver())**
```

### LifecycleOwner

LifecycleOwner는 클래스에 Lifecycle이 있음을 나타내는 단일 메서드(getLifecycle()) 인터페이스다. 프래그머트 및 액티비티는 지원 라이브러리 26.1.0 이상에서 LifecycleOwner 인터페이스를 이미 구현하고 있으며, 기본적으로 LifecycleOwners이다. **LifecycleRegistry** 클래스를 사용하고, LifecycleObserver 인터페이스를 구현하여 사용자 지정 클래스를 생명 주기 Owners로 구성할 수도 있다.

* kotlin extension → launched.... 이러한 유용한 함수를 사용하면 좋음.

LifecycleOwner 인터페이스는 프래그먼트 및 AppCompatActivity와 같은 개별 클래스에서 Lifecycle의 ownership을 추출하고 함께 작동하는 컴포넌트를 작성할 수 있게 한다.

생명 주기 Owners의 상태가 변경되면 할당 된 생명 주기 개체가 새 상태로 업데이트된다. 생명 주기 Owners는 다음 5가지 상태 중 하나이다.

* Lifecycle.State.INITIALIZED
* Lifecycle.State.CREATED
* Lifecycle.State.STARTED
* Lifecycle.State.RESUMED
* Lifecycle.State.DESTROYED

따라서 Owners 상태가 특정 생명 주기 단계에 있어야 할 때 현재 상태 객체의 isAtLeast() 방법을 사용할 수도 있다.

```kotlin
internal class MyLocationListener(
            private val context: Context,
            private val lifecycle: Lifecycle,
            private val callback: (Location) -> Unit
    ) {

        private var enabled = false

        @OnLifecycleEvent(Lifecycle.Event.ON_START)
        fun start() {
            if (enabled) {
                // connect
            }
        }

        fun enable() {
            enabled = true
            if (**lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)**) {
                // connect if not connected
            }
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
        fun stop() {
            // disconnect if connected
        }
    }
```

전체 애플리케이션 프로세스의 생명 주기를 관리하려는 경우, ProcessLifcyceOwner를 사용해야 한다.

* 애플리케이션 라이프사이클(ALM)
* 뷰모델 라이프사이클
* 프래그먼트 라이프사이클
* 액티비티 라이프사이클
