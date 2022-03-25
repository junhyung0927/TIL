---
description: 결과 부족이 발생할 경우 null과 Failure를 사용하라
---

# item7

프로젝트를 만들다보면 함수가 원하는 결과를 만들어 낼 수 없을 때가 있습니다.

* 서버로부터 데이터를 읽고 싶을 때, 인터넷 연결 문제가 발생한 경우
* 조건에 맞는 첫 번째 요소를 찾으려 했을 때, 조건에 맞는 요소가 없는 경우
* 텍스트를 파싱해서 객체를 만드려고 했는데, 텍스트의 형식이 맞지 않는 경우

이러한 상황을 처리하는 두 가지 메커니즘이 존재합니다.

* null 또는 ‘실패를 나타내는 sealed 클래스(일반적으로 Failure라는 이름을 붙임)’를 리턴함
* 예외를 throw 함

위의 두 가지 메커니즘 간 중요한 차이점이 있습니다.

_예외를 throw 함_

1. 예외는 정보를 전달하는 방법으로 사용해선 안됨. 즉, 예외는 잘못된 특별한 상황을 나타내야함
2. 예외는 예외적인 상황이 발생했을 때 사용하는 것이 좋음

이러한 이유를 정리하면 다음과 같습니다.

* 많은 개발자가 예외가 전파되는 과정을 정확히 추적하지 못함
* 코틀린의 모든 예외는 ‘unchecked’ 예외임. 따라서 사용자가 처리하지 않을 수도 있으며, 이와 관련된 내용은 문서에도 제대로 드러나지 않음.
* 예외는 예외적인 상황을 처리하기 위해서 만들어졌으며 명시적인 테스트(explict test) 만큼 빠르게 동작하지 않음
* try-catch 블록 내부에 코드를 배치하면, 컴파일러가 할 수 있는 최적화가 제한됨

_null 또는 ‘실패를 나타내는 sealed 클래스(일반적으로 Failure라는 이름을 붙임)’를 리턴함_

반면, null과 Failure는 예상되는 오류를 표현할 때 굉장히 유용합니다. 이는 명시적으로 효율적이며 간단한 방법으로 처리 가능합니다.

따라서 **충분히 예측할 수 있는 범위의 오류는 null과 Failure를 사용**하고, **예측하기 어려운 예외적인 범위의 오류는 예외를 throw**해서 처리하는 것이 좋습니다.

```kotlin
inline fun <reified T> String.readObjectOrNull(): T? {
	//...
	if(incorrectSign) {
		return null
	}
	//...
	return result
}

inline fun <reified T> String.readObject(): Result<T> {
	//...
	if(incorrectSign) {
		return Failure(JsonParsingException())
	}
	//...
	return Success(result)
}

sealed class Result<out T>
class Success<out T>(val result: T): Result<T>()
class Failure(val throwable: Throwable): Result<Nothing>()

class JsonParsingException: Exception()
```

위와 같은 방법으로 표시되는 오류는 다루기 쉬우며 놓치기 어렵습니다. null을 처리할 때, safe call 또는 엘비스 연산자와 같은 다양한 널 안전성(null-safety) 기능을 활용해야 합니다.

```kotlin
val age = userText.readObjectOrNull<Person>()?.age ?: -1

val person = userText.readObjectOrNull<Person>()
val age = when(person) {
	is Success -> person.age
	is Failure -> -1
}
```

이러한 오류 처리 방식은 try-catch 블록보다 효율적이며, 사용하기 쉽고 더 명확합니다.

**try-catch 방식의 예외는 놓칠 수 있으며, 전체 애플리케이션을 중지 시킬 수 있습니다. 하지만 null 값과 sealed result 클래스는 명시적으로 처리해야 하며, 애플리케이션의 흐름을 중지하지 않습니다.**

앞서 살펴본 null과 sealed result 클래스의 차이점은 무엇일까?

추가적인 정보를 전달해야 한다면 sealed result를 사용하고, 그렇지 않으면 null을 사용하는 것이 일반적입니다.

> _Failure는 처리할 때 필요한 정보를 가질 수 있음_

개발자는 항상 자신이 요소를 안전하고 잘 다룰 수 있을거라 착각합니다. 따라서 **nullable을 리턴하는 것을 피해**야 합니다. 개발자에게 **null이 발생할 수 있다는 경고**를 하고 싶다면, getOrNull 등의 함수를 사용해서 **무엇이 리턴되는지 예측할 수 있게 하는 것이 좋습니다**.
