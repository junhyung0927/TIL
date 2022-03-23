---
description: inferred 타입으로 리턴하지 말라
---

# item4

코틀린의 타입 추론(type inference)은 가장 널리 알려진 코틀린의 특징입니다. 다만 무작정 타입 추론을 사용하다간 위험한 상황을 초래할 것입니다.

#### inferred 타입은 오른쪽에 있는 피연산자에 맞게 설정된다

```kotlin
//슈퍼클래스 또는 인터페이스로 타입 추론은 불가능하다
open class Animal
class Zebra: Animal()

fun main() {
	var animal = Zebra()
	animal = Animal() // 에러: Type mismatch
}
```

일반적인 경우에는 위와 같은 문제가 발생하지 않겠지만, 제한된 타입이 설정되었다면 타입을 명시적으로 꼭 지정해야합니다.

```kotlin
open class Animal
class Zebra: Animal()

fun main() {
	var animal: Animal = Zebra()
	animal = Animal() 
}
```

하지만 라이브러리(또는 모듈)를 조작할 수 없는 경우에는 위와 같은 문제를 간단히 해결할 수 없습니다. 그리고 inferred 타입을 노출하면, 위험한 일이 발생할 수 있습니다.

```kotlin
interface CarFactory {
	fun produce(): Car
}

val DEFAULT_CAR: Car = Fiat126P()
```

대부분의 공장에서 Fiat126P라는 자동차를 주로 생성하여 이를 디폴트로 두었다고 가정합시다. 개발자가 코드를 작성하다 보니 DEFAULT\_CAR는 Car로 명시적으로 지정되어 있어 함수의 리턴 타입을 제거했다고 합시다.

```kotlin
interface CarFactory {
	fun produce() = DEFAULT_CAR
}
```

협업하는 또 다른 개발자가 해당 코드를 보다가, “DEFAULT\_CAR는 타입 추론에 의해 자동으로 타입이 결정되겠지?“라고 생각한다고 합시다.

```kotlin
val DEFAULT_CAR = Fiat126P()
```

문제가 보이나요? 이제 이 공장에서는 Fiat126P 이외의 다른 종류의 자동차를 생산하지 못할 것입니다. DEFAULT\_CAR의 타입은 Fiat126P이 되니 말이죠.

위와 같이 인터페이스를 우리가 직접 만들었다면 수월하게 문제를 찾고 해결하겠지만, 외부 API를 사용한다면 문제를 쉽게 해결할 수 없을 것입니다.

**리턴 타입은 API를 잘 모르는 사용자에게 전달해 줄 수 있는 중요한 정보**입니다. 따라서 **리턴 타입은 외부에서 확인할 수 있게 명시적으로 지정해주는 것이 좋습니다**.
