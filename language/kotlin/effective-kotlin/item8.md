---
description: 적절하게 null을 처리하라
---

# item8

null을 리턴한다는 것은 함수에 따라서 여러 의미를 내포할 수 있습니다.

* String.toIntOrNull()은 String을 Int로 적절하게 변환할 수 없는 경우 null을 리턴
* Iterable\<T>.firstOrNull(() → Boolean)은 주어진 조건에 맞는 요소가 없는 경우 null을 리턴

위의 예제처럼 null은 최대한 명확한 의미를 가지는 것이 좋습니다. 이는 nullable 값을 처리해야 하기 때문입니다.

기본적으로 nullable 타입은 세 가지 방법으로 처리합니다.

* ?. , 스마트 캐스팅, 엘비스 연산자 등을 활용하여 안전하게 처리
* 오류를 throw 함
* 함수 또는 프로퍼티를 리팩터링하여 nullable 타입이 나오지 않게 수정한다.

### null을 안전하게 처리하기

```kotlin
printer?.print() //안전 호출

if (printer != null) printer.print() //스마트 캐스팅
```

null을 안전하게 처리하는 방법 중 가장 많이 사용되는 안전 호출(safe call)과 스마트 캐스팅(smart casting)이 있습니다. 애플리케이션 사용자와 개발자 입장에서도 nullable 값을 처리할 때 안전한 방법입니다.

```kotlin
val printerName1 = printer?.name ?: "Unnamed"
val printerName2 = printer?.name ?: return
val printerName3 = printer?.name ?:
	throw Error("Printer must be named")
```

위는 엘비스 연산자를 사용하여 nullable을 처리한 코드입니다. 엘비스 연산자는 return 또는 throw를 포함한 모든 표현식이 혀용됩니다.

많은 객체는 nullable과 관련된 처리를 지원합니다. 대표적인 예를 들면, 컬렉션 처리에선 무언가 없다는 것을 나타낼 때 null이 아닌 빈 컬렉션을 사용하는 것이 일반적입니다.

스마트 캐스팅은 코틀린의 규약 기능(contracts feature)을 지원합니다.

```kotlin
println("What is your name?")
val name = readLine()
if (!name.isNullOrBlank()) {
	println("Hello ${name.toUpperCase()}")
}

val news: List<News>? = getNews()
if (!news.isNullOrEmpty()) {
	news.forEach { notifyUser(it) }
}
```

### 오류 throw 하기

개발을 하다보면 특정 로직의 결과가 null이 되리라 예상하지 못하는 경우가 종종 있습니다. 이때 개발자는 에러가 발생하면 추적하기 힘들 것입니다.

따라서 개발자가 어떤 코드를 보고 선입견 처럼 "당연히 그럴 것이다”라고 생각되는 부분이 있고, 그 부분에서 문제가 발생할 경우에는 **개발자에게 오류를 강제로 발생시켜 주는 것이 좋습니다.**

```kotlin
fun process(user: User) {
	requireNotNull(user.name)
	val context = checkNotNull(context)
	val networkService =
      getNetworkService(context) ?: throw NoInternetConnection()
	networkService.getData { data, userData ->
		show(data!!, userData!!)
	}
}
```

### not-null assertion(!!)과 관련된 문제

nullable을 처리하는 가장 간단한 방법은 not-null assertion(!!)을 사용하는 것입니다. 하지만 이는 사용하기 쉽지만, 좋은 해결 방법은 아닙니다.

예외가 발생할 때, 어떤 설명도 없는 제네릭 예외가 발생합니다. 또한 코드가 짧고 너무 사용하기 쉽다 보니 남용하게 되는 문제도 발생합니다. !! 타입은 nullable 이지만, null이 나오지 않는다는 것이 확실한 상황에서 사용하는데, **이는 현재 확실하다고 미래에도 확실한 것은 아니기에 위험한 방법입니다.**

예를 들어봅시다. 파라미터로 4개의 숫자를 받고, 이중에서 가장 큰 것을 찾는 함수를 생각해 봅시다.

```kotlin
fun largestOf(a: Int, b: Int, c: Int, d: Int): Int = 
	listOf(a, b, c, d).max()!!
```

위와 같은 간단한 코드에서도 !!은 NPE로 이어질 수 있습니다.

```kotlin
fun largestOf(vararg nums: Int): Int
	nums.max()!!

largestOf() // NPE
```

nullability와 관련된 정보는 숨겨져 있어, 굉장히 쉽게 놓칠 수 있습니다. 이와 같이 코드를 작성하면, 이후에 프로퍼티를 계속해서 언팩(unpack)해야 하므로 사용하기 귀찮아질 것입니다. 또한, 해당 프로퍼티가 이후에 의미 있는 null 값을 가질 가능성 자체를 차단해 버립니다.

미래에 코드가 어떻게 변할지는 아무도 알 수 없습니다. !! 연사자를 사용하거나 명시적으로 예외를 발생시키는 형태로 설계하면, **미래의 어느 시점에서 해당 코드가 오류를 발생시킬 수 있다는 것**을 명심해야 합니다.

일반적으로 !! 연산자 사용을 지양해야 합니다. !! 연산자가 의미 있는 경우는 굉장히 드뭅니다. 일반적으로 nullability가 제대로 표현되지 않는 라이브러리를 사용할 때 정도에만 사용해야 합니다. 코틀린 커뮤니티 전체에서도 !! 연산자를 지양하자고 주장하고 있습니다. **!! 연산자를 보면 반드시 조심하고, 무언가가 잘못되어 있을 가능성을 생각합시다.**

### 의미 없는 nullability 피하기

nullability는 어떻게든 적절하게 처리하기 위해 추가 비용이 발생하므로, 필요한 경우가 아니라면 nullability 자체를 피하는 것이 좋습니다.

null은 중요한 메시지를 전달하는 데 사용될 수 있습니다. 따라서 **다른 개발자가 보기에 의미가 없을 때는 null을 사용하지 않는 것이 좋습니다.**

nullability을 피할 때 사용할 수 있는 몇 가지 방법을 소개합니다.

* 클래스에서 nullability에 따라 여러 함수를 만들어서 제공할 수도 있습니다. 대표적인 예로 List\<T>의 get과 getOrNull 함수가 있습니다.
* 어떤 값이 클래스 생성 이후에 확실하게 설정된다는 보장이 있다면, lateinit 프로퍼티와 notNull 델리게이트를 사용하세요.
* 빈 컬렉션 대신 null을 리턴하지 마세요. null은 컬렉션 자체가 없다는 것을 나타내기 때문에, 요소가 부족하는 것을 나타내려면 빈 컬렉션을 사용하세요.
* nullable enum과 None enum 값은 완전히 다른 의미입니다. null enum은 별도로 처리해야 하지만, None enum 정의에 없으므로 필요한 경우에 사용하는 쪽에서 추가해서 활용할 수 있다는 의미입니다(이해 안감).

### lateinit 프로퍼티와 notNull 델리게이트

클래스가 클래스 생성 중에 초기화할 수 없는 프로퍼티를 가지는 것은 드문 일은 아니지만 분명 존재할 수 있습니다. 이러한 프로퍼티는 사용 전에 반드시 초기화해서 사용해야 합니다. 예로 안드로이드의 데이터 바인딩 중 Fragment 마다 사용할 바인딩 객체가 다르기 때문에 lateinit 프로퍼티를 사용합니다.

```kotlin
abstract class BaseFragment<B: ViewDataBinding>(
	@LayoutRes val layoutId: Int
) : Fragment() {
	protected lateinit var bindnig: B
	//...
}
```

이처럼 lateinit 한정자는 프로퍼티가 이후에 설정될 것임을 명시합니다.

물론 lateinit도 비용이 발생하는데, 만약 초기화 전에 값을 사용하려고 하면 당연히 예외가 발생할 것입니다. 이를 방지하기 위해 **처음 사용하기 전에 반드시 초기화가 되어 있을 경우에만 lateinit을 붙이는 것입니다**.

lateinit은 nullable과 비교하면 다음과 같이 차이점이 있습니다.

* !! 연산자로 언팩(unpack)하지 않아도 됨
* 나중에 어떤 의미를 나타내기 위해 null을 사용하고 싶을 때, nullable로 만들 수도 있음
* 프로퍼티가 초기화된 이후에는 초기화되지 않은 상태로 돌아갈 수 없음

**lateinit은 프로퍼티를 처음 사용하기 전에 반드시 초기화될 것**이라고 예상되는 상황에 활용합니다. 예를 들어 안드로이드 Activity의 onCreate가 있습니다.

반대로 lateinit을 사용할 수 없는 경우도 있습니다. JVM에서 Int, Long과 같은 기본 타입과 연결된 타입으로 프로퍼티를 초기화해야 하는 경우입니다. 이런 경우엔 Delegates.notNull을 사용합니다.

```kotlin
class DoctorActivity: Activity() {
	private var doctorId: Int by Delegates.notNull()
	private var fromNotification: Boolean by
       Delegates.notNull()

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		doctorId = intent.extras.getInt(DOCTOR_ID_ARG)
		fromNotification = intent.extras.getBoolean(FROM_NOTIFICATION_ARG)
	}
}
```

프로퍼티 위임을 사용하면, nullability로 발생하는 여러 가지 문제를 안전하게 처리할 수 있습니다.
