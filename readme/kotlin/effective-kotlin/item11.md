---
description: 가독성을 목표로 설계하라
---

# item11



로버트 마틴의 “클린 코드”라는 책을 통해 널리 알려진 이야기로 “_개발자가 코드를 작성하는 데는 1분이 걸리지만, 이를 읽는 데는 10분이 걸린다_”가 있습니다.

협업을 하다보면 항상 느끼듯이 개발자는 어떤 코드를 작성하는 것보다 읽는 데 많은 시간을 소모하는 것을 느낄 것입니다.

프로그래밍은 쓰기보다 읽기가 중요하므로, **항상 가독성을 생각하면서 코드를 작성해야 합니다.**

### 인식 부하 감소

```kotlin
// 구현 A
if (person != null && person.isAdult) {
	view.showPerson(person)
} else {
	view.showError()
}

// 구현 B
person?.takeIf { it.isAdult }
     ?.let(view::showPerson)
     ?: view.showError()
```

두 개의 코드를 보고 어떤 것이 가독성이 좋다고 느끼시나요? 코틀린을 어느정도 학습한 사람들은 B를 뽑겠지만, B는 가독성이 좋은 코드라고 할 수 없습니다.

**가독성이란 코드를 읽고 얼마나 빠르게 이해할 수 있는지를 의미**합니다. 코틀린에서 일반적으로 사용되는 관용구가 많다고 해서, 가독성이 좋다고 할 수 없습니다. 숙련된 개발자만을 위한 코드는 좋은 코드가 아닙니다. 또한, 숙련된 코틀린 개발자라고 할지라도, 관용구가 많은 코드가 익숙하지 않다면 이해하는데 시간이 많이 걸릴 것입니다. 즉, A가 B 보다 훨씬 가독성이 좋은 코드라고 할 수 있습니다.

**가독성이 좋은 코드는 수정하기가 용이**합니다.

```kotlin
// 구현 A
if (person != null && person.isAdult) {
	view.showPerson(person)
	view.hideProgressWithSuccess()
} else {
	view.showError()
	view.hideProgress()
}

// 구현 B
person?.takeIf { it.isAdult }
     ?.let {
             view.showPerson(it)
             view.hideProgressWithSuccess()
     } ?: run {
             view.showError()
             view.hideProgress()   
}
```

구현 A: if 블록에 추가만 해주면 됨.

구현 B: 더 이상 함수 참조를 사용할 수 없으므로, 코드를 수정해야 함. 또한, 엘비스 연산자와 표준 함수 run을 사용

또한, **익숙하지 않은 구조를 사용하면, 잘못된 동작을 구현한 코드를 보고 그냥 넘어갈 수 있습니다.**

구현 A와 B의 실행결과가 다른 것을 알고 있었나요?

구현 A는 if 조건을 만족하지 않으면 else 문에 작성된 함수를 실행할 것입니다.

구현 B는 let 문 안의 `view.showPerson(it)` 함수가 null을 리턴한다면 `view.hideProgressWithSuccess()` 함수 뿐만 아니라 `view.showError()` 함수도 동시에 호출할 것입니다.

정리하자면, 기본적으로 **‘인지 부하'를 줄이는 방향**으로 코드를 작성해야 합니다. 즉, 익숙한 코드를 작성하는 것을 지향합시다.

### 극단적으로 코드를 작성하지 않기

앞서 구현 B에서 let 키워드를 사용해서 잘못된 동작이 발생했다고, 무조건 let을 절대로 쓰지 말라는 소리가 아닙니다.

nullable 가변 프로퍼티가 있고, null이 아닐때만 특정 작업을 수행하는 예를 살펴봅시다.

```kotlin
class Person(val name: String)
//불변 프로퍼티
//불변 프로퍼티 일때는 내부적으로 컴파일할 때 if문 메서드를 만들어서 검사하기 때문에, let을 사용하면 코드만 길어진다. 
//val person: Person? = null
//가변 프로퍼티
var person: Person? = null

fun printName() {
	person?.let {
		print(it.name)
	}
}
```

무조건 관용구 많다고 가독성이 떨어진다고는 할 수 없습니다. 당연히 경험이 적은 코틀린 개발자는 이해하기 어렵기에 학습하는 비용이 들 것입니다. 하지만 이 비용은 지불할 만한 가치가 있습니다.

정당한 이유 없이 복잡성을 추가하는 코드는 지양해야 하지만, 그렇지 않은 경우에는 비용을 지불할 만한 가치가 있습니다.

이렇게 얘기하면 어떤 것이 비용이 지불할 만한 코드인가요..? 라고 물을 수 있습니다. 어떤 구조들이 어떤 복잡성을 가져오는지 등을 파악하여 균형을 맞추는 것이 중요합니다.

### 컨벤션

```kotlin
fun main() {
    val abc = "A" { "B" } and "C"
    println(abc)
}

operator fun String.invoke(f: () -> String): String = this + f()

infix fun String.and(s: String) = this + s
```

위의 코드는 코틀린으로 할 수 있는 최악의 코드입니다. 이 코드는 수많은 규칙들을 위반하고 있습니다.

* **연산자는 의미에 맞게 사용하자**(invoke는 이럴 때 사용하는 함수가 아니다)
* ‘람다를 마지막 아규먼트로 사용한다’ 라는 컨벤션을 적용하면, _**코드가 복잡해진다.**_
* and 함수 네이밍은 **실제 함수 내부에서 이루어지는 처리와 맞지 않습니다.**
* 문자열을 결합하는 기능은 **이미 언어에 내장**되어 있습니다.
