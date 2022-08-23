# item1

```kotlin
class BankAccount {
    var balance = 0.0
        private set
	
    fun deposit(depositAccount: Double) {
        blance += depositAccount
    }

    @Throws(InsfficientFunds::class)
    fun withdraw(withdrawAmount: Double) {
        if (blance < withdrawAmount) {
            throw InsufficientFunds()
        }
        balance -= withdrawAmount
    }
}

class InsfficientFunds : Exception()
val account = BankAccount()
println(account.blance)  // 0.0
account.deposit(100.0)
println(account.blance)  // 100.0
account.withdraw(50.0)
println(account.blance)  // 50.0
```

위 코드의 BankAccount에는 계좌에 돈이 얼마나 있는지 상태를 나타내는 함수가 있습니다. 이렇게 상태를 갖게 하는 것은 양날의 검입니다. 시간의 변화에 따라서 변하는 요소를 표현할 수 있다는 것은 유용할 수 있지만, 상태를 적절하게 관리하는 것은 개발자를 골치 아프게 합니다.

* **프로젝트를 이해하고 디버깅하기 힘들어집니다.** 상태가 변경되는 관계를 이해하고, 추적 시 어려움을 겪게 될것입니다. 코드 이해도는 높아지고 변경하기도 어려워지는 단점을 가지게 됩니다. 또한 예상치 못한 에러도 겪게 될 것입니다.
* **가변성(mutability)이 존재하면, 코드의 실행을 추론하기 어려워집니다.** 시점에 따라 값이 변경될 수 있고, 현재 어떤 값을 갖고 있는지 알아야 코드의 실행을 예측할 수 있습니다. 또한 특정한 시점에 확인한 값이 계속 동일한 값으로 유지되란 법도 없습니다.
* **멀티스레드 프로그램일 때 동시성 에러가 발생할 수 있습니다.**
* 모든 상태를 테스트해야하기 때문에 **테스트하기가 어렵습니다.**
* **상태 변경이 일어날 때, 이러한 변경을 다른 부분에 알려야 하는 경우**가 있습니다. 예를 들어 정렬되어 있는 리스트에 가변 요소를 추가한다면, 요소에 변경이 일어날 때마다 리스트 전체를 다시 정렬해야 하는 비용이 발생합니다.

```kotlin
//멀티스레드 활용하여 프로퍼티 수정
var num = 0
for (i in 1..1000) {
    thread {
        Thread.sleep(10)
        num += 1
    }
}

//코루틴 사용하여 프로퍼티 수정
suspend fun main() {
    var num = 0
    coroutineScope {
    for (i in 1..1000) {
        launch {
            delay(10)
            num += 1
            }
        }
    }
}

//멀티 스레드 경우
Thread.sleep(5000)
print(num) // 1000이 아닐 확률이 매우 높다
// 실행할 때마다 다른 숫자가 출력된다.

//코루틴 경우
print(num) // 실행할 때마다 다른 숫자가 출력된다.
```

협업 프로젝트에서 위와 같이 코드를 작성하게 되면 동시성 문제를 피할 수 없을것입니다. 일부 연산이 충돌되어 예상치 못한 값이 도출될 수 있으니 동기화 코드를 추가로 구현해야 합니다(변할 수 있는 지점을 줄이자).

```kotlin
val lock = Any()
var num = 0
for (i in 1..1000) {
    thread {
        Thread.sleep(10)
        // 다른 스레드 접근 금지
        synchronized(lock) { 
            num += 1
        }
    }
}

Thread.sleep(1000)
print(num) // 1000
```

**변경이 일어나야 하는 부분을 신중하고 확실하게 결정하고 사용하는게 가장 중요합니다.**

### 코틀린에서 가변성 제한하기

#### 읽기 전용 프로퍼티(val)

일반적으로 코틀린에서 val(value)을 사용해서 값을 변하지 못하게 합니다.

```kotlin
val a = 10
a = 20 //오류
```

하지만 읽기 전용 프로퍼티가 **완전히 변경이 불가능한 것은 아니라는 것**을 주의해야 합니다. 읽기 전용 프로퍼티가 mutable 객체를 담고 있다면, 내부적으로 변할 수 있습니다.

```kotlin
val list = mutableListOf(1,2,3)
list.add(4)
//list = mutableListOf(4,5,6)과 같이 재할당을 불가능하다.

print(list) //1,2,3,4
```

읽기 전용 프로퍼티는 다른 프로퍼티를 활용하는 **사용자 정의 게터**로도 정의할 수 있습니다.

```kotlin
var name: String = "cho"
var surname: String = "junhyung"
val fullName 
    get() = "$name $surname" 

fun main() {
    println(fullName) //cho junhyung
    name = "joo"
    println(fullName) //joo junhyung
}
```

var은 게터와 세터를 모두 제공하지만, val은 변경이 불가능하므로 게터만 제공합니다. 즉, **val을 var로 오버라이드** 할 수 있습니다.

```kotlin
interface Element {
    val active: Boolean
}

class ActualElement: Element {
    override var active: Boolean = false 
}
```

위에서 살펴본 것 처럼, val의 값이 변경될 수 있긴 하지만, 프로퍼티 자체를 변경할 수 없으므로 일반적으로 var 보다는 val을 많이 사용합니다.

**val은 읽기 전용 프로퍼티지만, 변경할 수 없음(immutable)을 의미하는 것은 아니라는 점**을 유의해야 합니다. 만약 완전히 변경할 필요가 없다면 final 프로퍼티를 사용하는 것이 좋습니다.

```kotlin
//스마트 캐스팅
val name: String? = "jun"
val surname: String = "hyung"

val fullName: String?
    get() = name?.let { "$it $surname" }

val fullName2: String? = name?.let { "$it $surname" }

fun main() {
    if (fullName != null) {
        println(fullName.length) //오류
    }

    if (fullName2 != null) {
        println(fullName2.length) //9 
    }
     
}
```

fullName은 게터로 정의했으므로 스마트 캐스트를 할 수 없습니다. 커스텀 게터를 사용하기 때문에, 값을 사용하는 시점의 name에 따라서 다른 결과가 나올 수 있습니다. fullName2처럼 지역 변수가 아닌 프로퍼티(non-local-property)가 final이고, 커스텀 게터를 갖지 않을 경우 스마트 캐스트를 할 수 있습니다.



#### 가변 컬렉션과 읽기 전용 컬렉션 구분하기

Iterable, Collection, Set, List 인터페이스는 읽기 전용이기 때문에 변경을 위한 메서드를 따로 가지지 않습니다. 반면에 MutableIterable, MutableCollection, MutableSet, MutableList 인터페이스는 읽고 쓸 수 있는 컬렉션입니다.

이처럼 **mutable이 붙은 인터페이스**는 대응되는 **읽기 전용 인터페이스를 상속**받아서, 변경을 위한 메서드를 추가한 것입니다.

그렇다고 **읽기 전용 컬렉션이 내부의 값을 변경할 수 없다는 것** 의미는 아닙니다.

```kotlin
inline fun <T,R> Iterable<T>.map(
    transformation: (T) -> R
): List<R> {
    val list = ArrayList<R>()
    for (elem in this) {
    list.add(transformation(elem))
    }
    return list
}
```

위의 컬렉션처럼 **진짜로 불변(immutable)하게 만들지 않고, 읽기 전용으로 설계한 것을 집중해야 합니다.** 이로써 내부적으로 인터페이스를 사용하고 있으므로, 실제 컬렉션을 리턴할 수 있습니다. 따라서 플랫폼 고유의 컬렉션을 사용할 수 있습니다.

이는 **코틀린이 내부적으로 immutable하지 않은 컬렉션을 외부적으로 immutable하게 보이게 만들어서 얻어지는 안정성**입니다.

만약에 개발자가 ‘시스템 해킹’을 시도해서 다운캐스팅을 하게 되면 어떤 문제가 발생할까요? 리스트를 읽기 전용으로 리턴하면, 이를 반드시 읽기 전용으로만 사용해야 합니다. 컬렉션 다운캐스팅은 추상화를 무시하는 행위이고 즉, 이러한 코드는 안전하지 못하고 예측하지 못한 결과를 초래합니다.

```kotlin
val list = listOf(1,2,3)

//절대 이렇게 하지 마세요!
if (list is MutableList) {
    list.add(4)
}
```

![ 실행 결과](<../../../.gitbook/assets/Untitled (27) (1).png>)

**코틀린에서 읽기 전용 컬렉션을 mutable 컬렉션으로 다운캐스팅해서는 안됩니다. 읽기 전용에서 mutable로 변경해야 한다면, copy를 통해서 새로운 mutable 컬렉션을 만드는 list.toMutableList를 활용해야 합니다.**



#### 데이터 클래스의 copy

immutable 객체를 사용하면, 다음과 같은 장점이 존재합니다.

* 한번 정의된 상태가 유지되므로, **코드를 이해하기 쉽습니다.**
* immutable 객체는 공유했을 때 충돌이 발생하지 않기 때문에, **병렬 처리를 안전하게 처리**할 수 있습니다.
* immutable 객체에 대한 참조는 변경되지 않으므로, **쉽게 캐시할 수 있습니다**.
* immutable 객체는 얕은 복사(defensive copy)를 만들 필요가 없고, 깊은 복사를 따로 하지 않아도 됩니다.
* immutable 객체는 다른 객체(mutable or immutable)를 만들 때 활용도가 높습니다. 또한, immutable 객체는 **실행을 더 쉽게 예측**할 수 있습니다.
* immutable 객체는 ‘set’ 또는 ‘map’의 키로 사용할 수 있습니다. mutable 객체는 변경이 가능하기 때문에 사용할 수 없습니다. 이는 세트와 맵이 내부적으로 해시 테이블을 사용하고, **해시 테이블은 처음 요소를 넣을때 요소의 값을 기반으로 버킷을 결정하기 때문입니다.**

위에서 살펴본 것처럼 mutable 객체는 예측이 어렵고 위험한 것을 알 수 있습니다. 따라서 **immutable 객체는 자신의 일부를 수정한 새로운 객체를 만들어 내는 메서드를 가져야 합니다.**

```kotlin
class User(
    val name: String,
    val surname: String
) {
    //새로운 객체를 생성해서 수정한 값을 가지게 해야 합니다.
    fun withSurname(surname: String) = User(name, surname)
}

var user = User("jun", "hyung")
user = user.withSurname("jooHyung")
print(user) //User(name = jun, surname = jooHyung)
```

이처럼 모든 프로퍼티를 대상으로 함수를 일일이 구현하는 것은 매우 귀찮은 일입니다. 이럴 땐 **data 한정자(copy)를 사용**하면 됩니다.

copy 메서드를 활용하면, 모든 기본 생성자 프로퍼티가 같은 새로운 객체를 만들어 낼 수 있습니다.

```kotlin
data class User(
    val name: String,
    val surname: String
)

var user = User("jun", "hyung")
user = user.copy(surname = "jooHyung")
print(user) //User(name = jun, surname = jooHyung)
```

위의 코드처럼, 데이터 모델 클래스를 만들어 immutable 객체로 만드는 것은 많은 장점을 가지고 있으므로, 기본적으로 이와 같은 방법을 사용하는 것을 지향해야 합니다.



### 다른 종류의 변경 가능 지점

list1과 list2의 장단점을 무엇일까요?

```kotlin
val list1: MutableList<Int> = mutableListOf()
var list2: List<Int> = listOf()

list1.add(1)
list2 = list2 + 1

list1 += 1 // list1.plusAssign(1)로 변경
list2 += 2 // list2 = list2.plus(1)로 변경
```

#### 변경 가능 지점

list1은 리스트 구현 내부에서 변경 가능 지점이 존재합니다. 멀티스레드 프로그램을 처리하는 경우, **내부적으로 적절한 동기화가 되어 있는지 확실하게 알 수 없으므로 위험**합니다.

list2는 프로퍼티 자체가 변경 가능 지점입니다. 따라서 멀티스레드 처리의 안정성이 더 높다고 할 수 있습니다(물론 잘못 만들면 일부 요소가 손실될 수 있습니다).

#### mutable 프로퍼티

mutable 리스트 대신 mutable 프로퍼티를 사용하는 형태는 커스텀 세터(또는 이를 활용하는 delegate)를 활용해서 변경을 추적할 수 있습니다.

mutable 컬렉션은 observe 할 수 있게 만드려면, 추가적인 구현이 필요합니다. 따라서 **mutable 프로퍼티에 읽기 전용 컬렉션을 넣어 사용하는 것이 편**리합니다.

```kotlin
var person = listOf<Person>()
    private set
```

mutable 컬렉션을 사용하는 것이 처음에는 더 간편하게 느껴지겠지만, **mutable 프로퍼티를 사용하면 객체 변경을 제어하기가 더욱 수월**합니다.

```kotlin
//이것만큼은 피하자!! 최악의 방법!!
var list3 = mutableListOf<Int>()
```

위 코드처럼 사용하면, 변경될 수 있는 모든 지점에 대한 동기화를 구현해야 하므로 피하도록 하자

### 변경 가능 지점 노출하지 말기

상태를 나타내는 mutable 객체를 외부에 노출하는 것은 굉장히 위험합니다.

```kotlin
data class User(val name: String) 

class UserRepository {
    private val storedUsers: MutableMap<Int, String> = 
    	mutableMapOf()

    fun loadAll(): MutableMap<Int, String> {
        return storedUsers // 접근이 가능하면 안되는데 가능하다.
    }
    //...
}
```

loadAll을 사용해서 private 상태인 UserRepository를 수정할 수 있습니다.

```kotlin
val userRepository = UserRepository()

val storedUsers = userRepository.loadAll()
storedUsers[4] = "Kirill"
//...

print(userRepository.loadAll()) // {4=Kirill}
```

위와 같은 코드는 위험한 상황이 발생할 가능성이 높습니다. 이를 처리하기 위한 방법을 알아봅시다.

#### Defensive copying

리턴되는 mutable 객체를 복제하는 것입니다. 이때 data 한정자로 만들어지는 copy 메서드를 활용하는 것이 좋습니다.

```kotlin
class UserHodler {
    private val user: MutableUser()

    fun get(): MutableUser {
    return user.copy()
    }

    //...
}
```

#### 가변성 제한

컬렉션은 객체를 읽기 전용 슈퍼타입으로 업캐스트하여 가변성을 제한할 수 있습니다.

```kotlin
data class User(val name: String)

class UserRepository {
    private val storedUsers: MutableMap<Int, String> = mutableMapOf()
	
    fun loadAll(): Map<Int, String> {
        return storedUsers
    }	
}
```

item1에서 언급했던 것처럼 가능하다면 무조건 가변성을 제한하는 것이 좋습니다.



