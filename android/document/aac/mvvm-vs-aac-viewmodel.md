---
description: 구글에서 진행한 ViewModel에 대한 안드로이드 개발자 Q&A
---

# MVVM vs AAC ViewModel

![](<../../../.gitbook/assets/Untitled (29) (1).png>)

![](<../../../.gitbook/assets/Untitled (30).png>)

### ViewModel은 왜 이름이 이렇게 만들어진건가?

ViewModel은 이름 뜻 그대로 사용할 데이터를 관리하기 위해서 지어진것이다. 즉, View에 표현해야 하는 데이터를 감싼 컨테이너 같은 개념으로 받아들이는 것이 좋다.



### MVVM vs AAC ViewModel

MVVM ViewModel은 개념(이론)에 가깝게 생각하면 되고 AAC ViewModel은 구현(기능)에 가까운 모델이라고 생각하면 좋다.

MVVM 패턴은 마틴 파울러에 의해 나온 MVP 패턴에서 파생된 패턴이다. MVVM 패턴의 목표는 비즈니스 로직과 프레젠테이션 로직을 UI로 부터 분리하는 것입니다. 당연히 MVVM 패턴은 Android에서만 사용되는 개념이 아닙니다. MVVM 패턴의 ViewModel은 View와 Model 사이에서 데이터를 관리하고 바인딩해주는 요소입니다.

AAC에서의 ViewModel은 안드로이드의 수명 주기를 고려해서 UI 상태 데이터를 관리하는 목적에 맞게 설계되었습니다. AAC의 ViewModel을 사용하면 Activity나 Fragment 같은 수명 주기 때문에 겪었던 상태 데이터 관리 측면에서 겪던 어려움을 간단하게 해결할 수 있습니다.

당연히 AAC ViewModel을 사용했다고 MVVM ViewModel을 사용했다고 이해하면 안된다. AAC ViewModel은 MVVM ViewModel의 하나의 종류이다.

MVP 패턴의 Presenter는 대부분의 비즈니스 로직을 처리한 후 View에게 통지를 하는 방식으로 구현된다. 그런 과정에서 Presenter 한쪽의 의존성이 너무 커지는 문제가 발생한다. 이에 반해 MVVM 패턴은 데이터바인딩을 통해서 View와 ViewModel의 의존을 줄여주어 “View는 디자이너만 관리하고, ViewModel은 개발자만 관리한다"라는 말이 나오기도 한다.

그러면 View와 ViewModel의 의존(결합도)를 깔끔하게 끊어낼 수 있을까? 사실 이런 이슈를 완벽하게 해결할 수 있는 케이스는 대기업이나 대형 플랫폼 같은 경우에는 가능하지만 현실적으로 규모가 크지 않은 프로젝트에서는 힘들다. (즉, Android에서 MVVM은 적합하지 않은 아키텍처???)

마지막으로 Google에서 어떤 의미로 AAC ViewModel을 제안했을까? MVVM 패턴의 ViewModel과 별개로 AAC ViewModel은 UI 상태 데이터를 View의 수명 주기와 관련 없이 재사용 가능하게 관리하는 목적으로만 사용하도록 제안한 것이다.

#### ViewModel 해외 사례

아시아 같은 경우 한국에서와 AAC ViewModel을 사용하는 것과 비슷하게 사용되고 있다(90% 이상). 컴포즈에 따라 MVPVM과 같은 형태로 변경되는 경우도 있지만 현재까지는 비슷한 추세이다.

그랩 같은 경우 MVVM의 ViewModel은 전혀 사용하지 않고, ViewModel을 해외에 거주하시는 개발자들과 논의할때 거의 100%에 가깝게 AAC ViewModel에 내포된 의미와 함께 설명한다. 아직까지 AAC ViewModel에 높은 의존성을 가지고 있다. 간혹, MVVM 아키텍처의 ViewModel을 응용해서 사용하는 경우도 있다.



### View와 ViewModel 간 통신 방식

질문 세미나를 듣고 있는 개발자분들의 설문 조사에 따르면 60%이상은 LiveData를 사용하고 있고, 그 외 rx, flow(coroutine) 등을 사용하고 있었다.

**그렇다면 어떤 방식이 가장 적합할까?**

정답은 없지만, 비동기 처리가 많은 경우에는 LiveData만 사용하지 말고 coroutine과 같은 flow 처리 도구를 사용하는 것이 좋다. flow의 학습이 되어있지 않다면 rx를 사용하는 것도 방법이다.

**도메인 모듈에서 Livedata를 쓰는것이 적합할까?**

쓰지 않는 것을 강력히 권장한다. 그 이유는 도메인 레이어는 어느 플랫폼에도 종속되지 않아야 하는데 LiveData는 안드로이드 플랫폼에서 제공하는 것이므로 종속성을 가지게 된다. 유닛테스트를 한다고 해도 발목이 잡히는 경우가 빈번할 것이다. 결론적으로 장점이 뚜렷히 보이지 않는다.

굳이 LiveData를 사용하게 된다면 LifecycleOwner와 연결을 해줘야 하기 때문에 애플리케이션에 영향을 많이 줄 것이다. 대안은 많기에 coroutine이나 rx를 사용하는 것을 매우 권장한다.

또한, Room 데이터베이스의 데이터를 수신할 때는 LiveData를 사용하지 말고 coroutine을 사용하는 것이 좋다. 굳이 안드로이드 플랫폼에 의존성을 갖게 하지 않기 위해 노력하자.



### View와 ViewModel 간 관계(1:1, 1:n, sharedViewModel ...)

**View와 ViewModel 간 1:1 관계가 적합한가?**

예를 들어 하나의 안드로이드 애플리케이션 프로젝트에 200명의 개발자가 협업을 한다고 가정해보자. 운이 좋으면 하나의 View를 맡게 되는 경우도 있겠지만 그럴 확률은 현저히 적을 것이다. 그렇다면 하나의 View에 기능에 따라 역할이 주어져 개발을 할텐데 1:1 관계를 가지게 되면 현실적으로 코드를 합치는 것이 불가능에 가까울 것이다. 그렇다면 자연스럽게 1:n 관계가 형성되게 될 것이다. 소규모 프로젝트에서는 1:1 관계가 형성될 수 있으나 1:n 관계가 형성될 수 있도록 하는 것을 권장한다.

**유스케이스로 분리하는 경우엔?**

ViewModel에서 적합하지 않은 로직을 유스케이스로 분리하지만 View와 ViewModel의 관계성에서는 영향이 없다고 보면 된다. 여기서 고민할 사항은 “로직컬을 어떻게 분리할 것인가” 이다.

**ViewModel이 여러 개가 되어서 통신할때는 어떻게 할 것인가?**

ViewModel 간 데이터 통신이 없다는 가정으로 구현을 해야한다. 공통된 관심사가 있을 경우에는 유스케이스에 repo 인터페이스를 사용하는 것이 좋다.

혹여, ViewModel 간 데이터 처리가 필요하다면 이벤트 처리(Pending 방식으로 다른 ViewModel 호출)로 사용하기도 한다.

**ViewModel과 유스케이스 간 통신**

특정 레포짓토리에서 데이터를 수신하거나 공통된 데이터를 관리할 경우나 n개의 ViewModel에서 동일한 로직을 사용할 경우에, 이러한 로직을 유스케이스로 분리하여 사용한다.

A ViewModel에서 발생한 이벤트에서 B ViewModel에 영향을 주는 경우(애니메이션, 특정 액션이 다른 ViewModel에 피치못하게 영향을 미치는 경우)에는 어쩔 수 없이 옵저버 패턴을 사용해서 구현하는 경우도 있음.

뱅크샐러드 애플리케이션의 경우, 하나의 화면은 많은 섹션으로 분리되어 있어 UI에 대해서 핸들링 할 데이터가 많았다. 대부분 Presenter나 ViewModel에서 가공해서 섹션 별로 UI를 처리하였다. ViewHolder나 View의 item을 가지고 있는 Model을 Helper 클래스로 나눠서 핸들링하면서 처리하였다.

**어떤 경우에 View를 나눠서 사용하는가? View와 ViewModel의 관계를 왜 1:n으로 사용하는 것인가?**

View를 여러 개로 쪼갠다고 보는 것이 아니라, View에 대한 데이터를 분산해서 핸들링 하는 것이다.

하나의 화면에 굉장히 많은 상품의 카테고리를 1:1 관계로 구현한다고 했을 때, UI 데이터 처리 같은 것들을 Helper 클래스로 관리하면 될 것이다.

![](<../../../.gitbook/assets/Untitled (31) (1).png>)

예를 들어 카카오톡 앱을 켜서 마지막 탭을 열어보면 날씨, 카카오 나우, 광고 배너 등 여러 가지의 화면이 있다.

이것을 하나의 ViewModel로 관리할 수도 있지만, 각 섹션 별로 나누어 ViewModel로 관리할 수도 있다.

만약, 특정 배너가 어느 섹션에서도 공유가 될때는 View와 ViewModel이 n:n 관계를 형성하게 될 것이다.



### View의 범위를 어디까지 정해야 할까?

View를 액티비티나 프래그먼트로 볼 것인지, 작은 텍스트뷰도 뷰로 볼 것인지는 개발자의 역량, 상황 규모에 따라 다르다. 즉, View를 작은 단위로 분리할 수도 있고 큰 단위로 분리할 수도 있다.

ViewHolder의 View 타입의 구조 또는 섹션으로 분리하여 그룹핑되어 재활용에 용이한 View가 좋은 케이스라고 볼 수 있다.

또한, 액티비티나 프래그먼트를 View라고 보기엔 힘들다. xml를 View라고 바라보는게 올바른 것이다. 이전에도 말했듯이 Viwe를 효율적으로 어떻게 나누냐는 개발자의 역량, 상황, 규모에 따라 다르다.

액티비티에 인플레이트된 컴포넌트들을 View라고 볼 수도 있다. 컴포넌트에 리스너를 달아 이벤트 처리를 액티비티나 프래그먼트에서 사용하는 것이 이상적인 경우다.

View를 나누는 기준이 중요한 것이 아닌, View가 어떠한 관심사를 가지고 있고 어떻게 분리를 할 것인가에 따라 개발 방향이 달라진다. 로직을 어디까지 분리해야 하는 가를 고민하는 것이 더 중요하다.

**ViewModel에서 특정 로직을 처리할 때 context를 래핑해서 가져와 처리하는 것이 위험하지 않은가? applicationContext를 사용해야 하는것인가?**

화면 전환 같은 구성 변경이 일어날 때 액티비티는 onCreate() → onDestory()가 발생하는데, ViewModel은 보다 큰 수명 주기를 가지고 있고 액티비티의 수명 주기에 영향을 미치지 않으므로 context를 계속 가지고 있을 것이다. 이때 메모리 릭이 발생할 확률이 매우 높다.

만약, 앱 내부에서 언어를 변경해야 하는 경우에 동적으로 알아서 변경이 되는 경우가 있을 것이다. 이때 applicatinoContext가 변경될 수 있다. 물론 메모리 릭이 발생하지 않겠지만, 변경된 구성 데이터에 대한 것을 ViewModel이 인지하지 못하고 사용할 경우가 존재하므로 이럴 땐 피하는 것이 좋다.

ViewModel에 대한 테스트 코드를 작성할 때는 View 또는 application 의존성을 최대한 끊어내는 것을 권장한다. ViewModel은 빠르게 테스트를 돌리는 것이 좋다(실제 클래스를 넘기는 것이 아닌 인터페이스를 넘긴다?)

[https://github.com/android-alatan/LifecycleComponents](https://github.com/android-alatan/LifecycleComponents)

#### 퓨어한 ViewModel을 구현한다면? 구성 변경 시 데이터를 보존할 때 savedStateInstance를 사용하면 직렬/역직렬화를 필수적으로 해야하므로 메모리에 넣는 것보다 효율이 떨어지는데 어떻게 관리하는 것이 좋은 방법인가?

savedStateInstance Handler를 사용해야 한다. AAC ViewModel을 오해하면 안되는 것이 MVVM ViewModel 또한 큰 데이터를 관리하는 것을 지양해야 한다.

savedStateInstance Handler의 데이터를 복원할 때, 결국 bundle 객체를 사용해야 한다. 큰 데이터를 관리할때 bundle 객체는 크기에 한계가 있어, 다른 방법을 사용해야 한다고 생각하면 안된다. 애초에 ViewModel에서 관리하면 안되는 데이터를 관리하는 것일 수 있기 때문이다. ViewModel에는 빠르게 경량화 할 수 있는 데이터만 관리하는 것이 좋다.

물론 처음부터 UI를 그리는 것도 방법이지만, 상황에 따라 다르기 때문에 판단하여 사용해야 한다.



### 이외 질문

**MutableLiveData를 사용할 때 양방향 데이터 바인딩을 처리할 때 어떻게 사용하는 것이 좋은가?**

**ViewHolder에서 이벤트를 처리할 때, 어댑터에 ViewModel을 보내 클릭 이벤트를 처리하는 경우나 콜백 인터페이스를 만들어서 액티비티/프래그먼트에서 이벤트를 받아 ViewModel로 이벤트를 보내는 경우도 보았는데 어떤 경우가 정석인가?**

가급적이면 어댑터에 ViewModel을 넘겨서 사용하지 않는 것을 권장하고 있다. 이는 RecyclerView의 어댑터와 ViewModel 간 의존성이 높이기 때문에 지양해야 한다.

정말 어쩔 수 없는 경우에는 전체 ViewModel을 넘기는 것이 아닌 하나의 로우(별개 기능정도)만 사용하는 별도의 ViewModel을 만들어서 데이터나 상태를 관리하는 것이 아닌 특정 로직을 수행하는 메서드만 관리하도록 사용하는 것이 좋다.

[https://developer.android.com/jetpack/guide/ui-layer/events#recyclerview-events](https://developer.android.com/jetpack/guide/ui-layer/events#recyclerview-events)

**WebView만 띄우는 액티비티 / 프래그먼트에서도 ViewModel을 사용해야 하나요? - 필요없다.**

**ViewModel에서 View의 수명주기를 관리한다는 점에서 LifecycleObserver, 즉 제어권을 넘겨서 핸들링 하고 싶다면? → 절대로 하면 안되는 패턴이다.**
