---
description: 예외를 활용해 코드에 제한을 걸어라
---

# item5

코틀린에서 코드의 동작에 제한을 걸 때 다음과 같은 방법을 사용할 수 있습니다.

* require 블록: 아규먼트를 제한할 수 있음
* check 블록: 상태와 관련된 동작을 제한할 수 있음
* assert 블록: 어떤 것이 true인지 확인할 수 있음. assert 블록은 테스트 모드에서만 작동함
* return 또는 throw와 함께 활용하는 엘비스 연산자

```kotlin
// Stack<T>의 일부
fun pop(num: Int = 1): List<T> {
	require(num <= size) {
		"cannot remove more elements than current size"
	}
	check(isOpen) { "cannot pop from cloesd stack" }
	val ret = collection.take(num)
	collection = collection.drop(num)
	assert(ret.size == num)
	return ret
}
```

위와 같이 제한을 걸어 주면 다양한 장점이 발생합니다.

* 문서를 읽지 않은 개발자도 문제를 확인할 수 있음
* 문제가 있을 경우 함수가 예상하지 못한 동작을 하지 않고 예외를 throw 함
* 코드가 어느 정도 자체적으로 검사를 하기 때문에 이와 관련된 단위 테스트를 줄일 수 있음
* 스마트 캐스트 기능을 활용할 수 있게 되므로, 타입 변환을 적게 할 수 있음

### 아규먼트

함수를 정의할 때 **타입 시스템을 활용해서 아규먼트(argument)에 제한**을 거는 코드를 많이 사용합니다.

```kotlin
fun factorial(n: Int): Long {
	require(n >= 0)
	return if (n <= 1) 1 else factorial(n - 1) * n
}

fun findClusters(points: List<Point>): List<Cluster> {
	require(points.isNotEmpty())
	//...
}

fun sendEmail(user: User, message: String) {
	requireNotNull(user.email)
	require(isValidEmail(user.email))
	//...
}
```

require 함수는 조건을 만족하지 못할 때 무조건적으로 IllegalArgumentException을 발생시키므로 제한을 무시할 수 없습니다. 일반적으로 이러한 처리는 함수의 앞부분에 배치되므로, 코드를 읽을 때 쉽게 파악할 수 있습니다.

> _코드를 읽지 않는 사람도 있기 때문에, 반드시 문서에 이러한 제한이 있다고 별도로 표시해야 합니다._

### 상태

어떤 구체적인 조건을 만족할 때만 함수를 사용할 수 있게 해야 할 때가 있습니다.

* 어떤 객체가 미리 초기화되어 있어야만 처리를 하게 하고 싶은 함수
* 사용자가 로그인했을 때만 처리를 하게 하고 싶은 함수
* 객체를 사용할 수 있는 시점에 사용하고 싶은 함수

이러한 상태와 관련된 제한을 걸 때는 일반적으로 check 함수를 사용합니다.

```kotlin
fun speak(text: String) {
	check(isInitialized)
	//...
}

fun getUserInfo(): UserInfo {
	checkNotNull(token)
	//...
}

fun next(): T {
	check(isOpen)
	//...
}
```

check 함수는 require 함수와 비슷하지만, 지정된 예측을 만족하지 못할 때, IllegalStateException을 throw 합니다. 즉, **상태가 올바른지 확인할 때 사용**합니다. 함수 전체에 대한 어떤 예측이 있을 때는 일반적으로 require 함수를 통과한 후 check 함수를 실행합니다.

이러한 상태 확인은 **사용자가 규약을 어기고, 사용하면 안 되는 곳에서 함수를 호출**하고 있다고 의심될 때 합니다. 사용자가 코드를 제대로 사용할 것이라고 믿기 보다는, **항상 문제 상황을 예측하고 문제 상황에 예외를 throw** 하는것이 좋습니다.

### Assert 계열 함수 사용

예를 들어 어떤 함수가 10개의 요소를 리턴한다면, “함수가 10개의 요소를 리턴하는가?”라는 코드는 참을 나타낼 것입니다. 하지만 함수가 올바르게 구현되어 있지 않을 수도 있습니다. 이러한 구현 문제로 발생할 수 있는 추가적인 문제를 예방하기 위해 단위 테스트를 사용합니다.

```kotlin
class StackTest {
	@Test
	fun 'Stack pops correct number of elements'() {
		val stack = Stack(20) { it }
		val ret = stack.pop(10)
		assertEquals(10, ret.size)
	}

	//...
}
```

**단위 테스트는 구현의 정확성을 확인하는 가장 기본적인 방법**입니다. 위의 코드는 stack이 10개의 요소를 pop하면, 10개의 요소가 나온다는 사실을 테스트하고 있습니다. 하지만 현재와 같이 하나의 경우만 테스트해서 모든 상황에서 괜찮을지 알 수 없으니, 모든 pop을 호출하는 위치에서 제대로 동작하는지 확인하는 것이 좋습니다.

```kotlin
fun pop(num: Int = 1): List<T> {
	//...
	assert(ret.size. == num)
	return ret
}
```

이러한 조건은 현재 코틀린/JVM에서만 활성화되며, -ea JVM 옵션을 활성화해야 확인할 수 있습니다. 테스트를 할 때만 활성화되므로, 오류가 발생해도 사용자가 알아차릴 수 없습니다. 만약 작성하고 있는 코드가 정말 심각한 오류고, 심각한 결과를 초래할 수 있는 경우엔 check를 사용하는 것이 좋습니다.

### nullability와 스마트 캐스팅

코틀린에서 require과 check 블록으로 어떤 조건을 확인해서 true가 나왔다면, 해당 조건은 이후로도 true일 거라고 가정합시다.

아래의 예제에서 어떤 사람(person)의 복장(person.outfit)이 드레스(Dress)여야 코드가 정상적으로 진행됩니다. 따라서 만약 이러한 outfit 프로퍼티가 final이라면, outfit 프로퍼티가 Dress로 스마트 캐스트 됩니다.

```kotlin
fun changeDress(person: Person) {
	require(person.outfit is Dress)
	val dress: Dress = person.outfit
	//...
}
```

이러한 특징은 어떤 대상이 null 인지 확인할 때 유용합니다.

```kotlin
class Person(val email: String?)

fun sendEmail(person: Person, message: String) {
	require(person.email != null)
	val email: String = person.email
	//...
}
```

nullability를 목적으로, 오른쪽에 throw 또는 return을 두고 엘비스 연산자를 활용하는 경우가 많습니다. 이러한 코드는 가독성이 좋고, 유연하게 사용 가능합니다.

```kotlin
fun sendEmail(person: Person, text: String) {
	val email: String = person.email ?: return
	//...
}
```

프로퍼티에 문제가 있어 null을 리턴할 경우, 여러 처리를 해야할 때 return/throw와 run 함수를 조합해서 활용하면 좋습니다.

```kotlin
fun sendEmail(person: Person, text: String) {
	val email: String = person.email ?: run {
		log("Email not sent, no email address")
		return 
	}
}
```

이처럼 **return과 throw를 활용한 엘비스 연산자**는 nullable을 확인할 때 굉장이 많이 사용되는 관용적인 방법입니다. 이를 적극적으로 활용하고, 함수의 앞부분에 작성하여 가독성을 높이는 것이 중요합니다.
