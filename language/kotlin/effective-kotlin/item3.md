---
description: 최대한 플랫폼 타입을 사용하지 말라
---

# item3



자바에서는 모든 것이 nullable일 수 있으므로 최대한 안전하게 접근한다면, 이를 nullable로 가정하고 다루어야 합니다. 하지만 코틀린 코드에서 만약 null을 리턴하지 않을 것을 확신한다면 !!를 붙이는 것을 확인할 수 있습니다.

nullable과 관련하여 자주 문제가 되는 제네릭 타입에 대해서 살펴봅시다.

```kotlin
//자바
public class UserRepo {
	public List<User> getUsers() {
		//...
	}
}

//코틀린
val users: List<User> = UserRepo().users!!.filterNotNull()

val users: List<List<User>> = UserRepo().groupedUsers!!.
         .map { it!!.filterNotNull() }
```

위의 코드와 같이, 코틀린이 디폴트로 모든 타입을 nullable로 다룬다면 사용하는 리스트와 리스트 내부의 객체들이 null이 아니라는 것을 일일이 확인해야 합니다. 만약 다른 제네릭 타입이라면, null을 확인하는 것 자체가 복잡한 일이 될 것 입니다. 그래서 코틀린은 **플랫폼 타입**들을 특수하게 다룹니다.

* 플랫폼 타입: 다른 프로그래밍 언어 전달되어서 nullable인지 아닌지 알 수 없는 타입을 뜻함

```kotlin
public class UserReop() {
	public User getUser() {
		//...
	}
}

val repo = UserRepo()
val user1 = repo.user //user1의 타입은 User!
val user2: User = repo.user //user2의 타입은 User
val user3: User? = repo.user //user3의 타입은 User?
```

코틀린 코드에서도 플랫폼 타입을 사용하여 관련된 코드를 작성할 수는 있지만, 저자는 이와 같은 플랫폼 타입은 안전하지 않으므로 지양해야 한다고 합니다.

```kotlin
public class JavaClass {
	public String getValue() {
		return null;
	}
}

fun statedType() {
	val value: String = JavaClass().value //NPE
	//...
	println(value.length)
}

fun platformType() {
	val value = JavaClass().value
	//...
	println(value.length) //NPE
}
```

statedType에서는 자바에서 값을 가져오는 위치에서 NPE가 발생합니다. 컴파일 시점에서 NPE가 발생되기 때문에 코드를 굉장히 쉽게 수정할 수 있습니다.

platformType에서는 값을 활용할 때 NPE가 발생합니다. 위의 예의 표현식 보다 복잡한 표현식을 가지고 있다면 런타임 시 NPE가 발생하기 때문에 이를 찾고 수정하는데 어려움을 겪게 될 것입니다.

이처럼 플랫폼 타입이 다른 곳에서 사용되는 일은 굉장히 위험합니다. 안전한 코드를 원한다면 이러한 부분을 제거하는 것이 좋습니다.

> _어떠한 로직에서든 null을 return 하는 것은 지양하자_
