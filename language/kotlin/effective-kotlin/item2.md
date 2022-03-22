---
description: 변수의 스코프를 최소화하라
---

# item2

상태를 정의할 때는 변수와 프로퍼티의 스코프를 최소화하는 것이 좋습니다.

* 프로퍼티보다는 지역 변수를 사용하자
* 최대한 좁은 스코프를 갖게 변수를 사용하자

```kotlin
//나쁜 예 -> for 스코프 외 밖에서도 사용 가능
var user: User
for (i in user.indices) {
	user = users[i]
	print("User at $i is $user")
}

//조금 나은 예 -> user의 스코프를 외부에서도 사용가능함(for문 안에서 변수 선언)
for (i in users.indices) {
	val user = users[i]
	print("User at $i is $user")
}

//가장 좋은 예 -> user의 스코프를 외부에서도 사용가능함(조건에서 변수 선언)
for ((i, user) in users.withIndex()) {
	print("User at $i is $user")
}
```

그렇다면 왜 스코프를 좁게 만드는 것이 좋을까요?

첫 번째 이유는 **프로그램을 추적하고 관리하기가 수월**하기 때문입니다. 복잡한 애플리케이션의 코드를 분석한다고 가정할 때, 우리는 프로퍼티들이 좁은 스코프를 가지고 있다면 변경을 추적하기 더욱 쉬울 것입니다. 즉, 쉽게 추적이 되어야 코드를 이해하고 변경하는 것이 수월합니다.

두 번째 이유는 **변수의 스코프 범위가 너무 넓으면, 다른 개발자에 의해 변수가 잘못 사용**될 수 있습니다. 예를 들어, 어떠한 로직이 존재할 때 내부 메서드 안에서만 사용될 a 프로퍼티를 전역 변수로 선언한다면 다른 개발자가 이를 인지하지 못하고 다른 메서드에서 사용할 경우가 존재할 것입니다. 즉, 또 다른 개발자들은 해당 코드를 이해하기 굉장히 어려울 것입니다.

변수는 읽고 쓰기 전용 여부와 상관 없이, **변수를 정의할 때 초기화**하는 것이 좋습니다.

```kotlin
//나쁜 예
fun updateWeather(degrees: Int) {
	val description: String
	val color: Int
	if (degrees < 5) {
		description = "cold"
		color = Color.BLUE
	} else if (degrees < 23) {
		description = "mild"
		color = Color.YELLOW
	} else {
		description = "hot"
		color = Color.RED
	}
	//...
}

//보다 나은 예
fun updateWeather(degrees: Int) {
	val (description, color) = when {
		degrees < 5 -> "cold" to Color.BLUE
		degrees < 23 -> "mild" to Color.YELLOW
		else -> "hot" to Color.RED
	}
	//...
}
```

### 캡쳐링

알고리즘 문제를 하나 풀어보자

* 2부터 시작하는 숫자 리스트(=(2..100)등)를 만듭니다.
* 첫 번째 요소를 선택합니다. 이는 소수입니다.
* 남아 있는 숫자 중에서 2번에서 선택한 소수로 나눌 수 있는 모든 숫자를 제거합니다.

```kotlin
var numbers = (2..100).toList()
val primes = mutableListOf<Int>()
while (numbers.isNotEmpty()) {
	val prime = numbers.first()
	primes.add(prime)
	numbers = numbers.filter { it % prime != 0 }
}
print(primes) // [2, 3, 5, 7, 11, 13, 17 ... , 79, 83, 89, 97]
```

시퀀스를 활용하는 예제로 조금 더 확장해봅시다.

```kotlin
val primes: Sequence<Int> = sequence {
	var numbers = generateSequence(2) { it + 1 }
	
	while(true) {
		val prime = numbers.first()
		yield(prime)

		numbers = numbers.drop(1)
         .filter { it % prime != 0 }
	}
}

print(primes.take(10).toList())
// [2,3,5,7,11,13,17,19,23,29]
```

아래와 같이 최적화를 하면 어떻게 결과가 나올까??

```kotlin
val primes: Sequence<Int> = sequence {
	var numbers = generateSequence(2) { it + 1 }
	
	var prime: Int
	while(true) {
		prime = numbers.first()
		yield(prime)
		numbers = numbers.drop(1)
         .filter { it % prime != 0 }
	}
}

print(primes.take(10).toList())
// [2,3,5,6,7,8,9,10,11,12]
```

결과를 보면 이상한 것을 볼 수 있습니다. 이는 **prime 변수를 캡처**했기 때문입니다. 반복문 내부에서 filter를 활용해서 prime으로 나눌 수 있는 숫자를 필터링합니다. 하지만 시퀀스를 활용하므로 필터링이 지연됩니다. 따라서 최종적인 prime 값으로만 필터링된 것입니다. 즉, prime이 2로 설정되어 있을 때 필터링된 4를 제외하면, drop만 동작하므로 그냥 연속된 숫자만 출력됩니다.

이러한 문제가 발생할 수 있으므로, 잠재적인 캡처 문제를 유의해야 합니다. **가변성을 피하고 스코프 범위를 좁게 만들면**, 이러한 문제를 피할 수 있습니다.
