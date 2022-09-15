# 람다로 프로그래밍



## 람다 식과 멤버 참조

### 코드 블록을 함수 인자로 넘기기

“이벤트가 발생하면 이 핸들러를 실행하자", “API로 받아온 데이터 구조의 모든 원소에 이 연산을 적용하자"와 같은 구현을 할 때 자바에서는 무명 내부 클래스를 통해 목적을 달성했다. 무명 내부 클래스를 구현해봤으면 알겠지만 매우 번거로운 것을 알 수 있다(코드를 함수에 넘기거나 변수에 저장와 같은).

함수형 프로그래밍(함수를 값처럼 다루는 접근 방법)을 통해 이를 해결할 수 있다.

_클래스 선언 후 해당 클래스의 인스턴스를 함수에 넘기는 방법(무명 내부 클래스)_ vs _함수를 직접 다른 함수에 전달(람다)_

```java
button.setOnClickListener(new OnClickListener) {
	@Override
	public void onClick(View view) {
		/* 클릭 시 수행할 동작 */
	}
}
```

```kotlin
button.setOnClickListener { /* 클릭 시 수행할 동작 */ }
```

위의 코드를 보다시피, 무명 내부 클래스를 사용했을때 보다 람다를 사용할 때 훨씬 간결하고 가독성이 좋은 것을 알 수 있다.

### 람다와 컬렉션

컬렉션을 다룰 때 람다가 없다면 편리하게 처리할 수 있는 좋은 라이브러리를 제공하기 힘들다. 자바에서는 필요에 따라 컬렉션 기능을 직접 작성하거나 했지만 코틀린에서는 굳이 그럴 필요가 없다.

사람들로 이뤄진 리스트가 있고 그 중에 가장 연장자를 찾는 코드를 살펴보자.

```kotlin
//람다 사용하지 않음
data class Person(val name: String, val age: Int)

fun findTheOldest(people: List<Person>) {
	var maxAge = 0
	var theOldest: Person? = null
	for (person in people) {
		for (person.age > maxAge) {
			maxAge = person.age
			theOldest = person
		}
	}
	println(theOldest)
}
```

```kotlin
//람다 사용
fun findTheOldest(people: List<Person>) {
	//people.maxBy { it.age }
	//멤버참조: people.maxBy { Person::age }
}
```

람다를 사용하지 않고 구현한 코드를 보면 중첩된 for문, 여러 개의 변수등 가독성이 떨어지는 것을 알 수 있다. 이에 반해 모든 컬렉션에 대해서 최대 값을 반환해주는 maxBy 함수를 사용하면 가독성이 매우 좋아진 것을 알 수 있다. 단지 **함수나 프로퍼티를 반환하는 역할을 수행하는 람다는 멤버참조로 대치**할 수 있다.

### 람다 식의 문법

람다는 값으로 여기저기 전달 할 수 있는 동작의 모음이다. 람다를 변수에 저장할 수도 있고, 함수에 인자로 넘기면서 바로 람다를 정의(대부분 이렇게 사용)할 수도 있다.

```kotlin
/*    파라미터    */   /* 본문 */
{ x: Int, y: Int ) -> x + y }
```

코틀린 람다 식은 항상 중괄호로 둘러싸여 있다. 인자 목록 주변에 괄호가 없다는 사실을 기억하자.

```kotlin
//람다를 변수에 저장
val sum = { x: Int, y: Int -> x + y } 
println(sum(1, 2))
>>> 3

//람다 식을 직접 호출
{ println(42) }()
>>> 42
```

굳이 람다를 만들자마자 바로 호출하는 것보단 람다 본문을 직접 실행하는 편이 낫다. 이렇게 코드의 일부분을 블록으로 둘러싸 실행할 필요가 있다면 run을 사용하자(코틀린 표준함수에 대해서는 한꺼번에 설명할 예정).

```kotlin
run { println(42) }
>>> 42
```

실행 시점에 코틀린 람다 호출에는 아무 부가 비용이 들지 않으며, 프로그램의 기본 구성 요소와 비슷한 성능을 낸다(inline 참고).

```kotlin
val people = listOf(Person("준형", 29), Person("길동", 30))

people.maxBy({ p: Person -> p.age })
people.maxBy() { p: Person -> p.age }
people.maxBy { p: Person -> p.age }
```

위와 같이 람다를 표현할 때 여러 가지 방법이 있다. 딱 보면 알 수 있듯이, 가장 마지막 표현식이 제일 가독성이 좋다. 코틀린에는 함수 호출 시 맨 뒤에 있는 인자가 람다 식이면, 그 람다를 괄호 밖으로 빼낼 수 있는 문법 관습이 있다. 가장 마지막 방법처럼 람다가 어떤 함수의 유일한 인자이고 괄호 뒤에 람다를 썼다면 호출 시 빈 괄호를 없애도 된다.

파라미터 타입에 대해서도 살펴보자. 람다 식은 로컬 변수처럼 람다 파라미터의 타입도 추론할 수 있다. 즉, 파라미터 타입을 명시할 필요가 없다. `파라미터 타입 = 컬렉션 원소 타입`

```kotlin
people.maxBy { p:Person -> p.age }
people.maxBy { p -> p.age }
```

파라미터 중 일부의 타입은 지정하고 나머지 파라미터는 타입을 지정하지 않아도 된다. 컴파일러가 파라미터 타입 중 일부를 추론하지 못하거나 타입 정보가 코드를 읽을 때 도움이 된다면 명시하면 된다.

마지막으로 it에 대해서 알아보자. 람다의 파라미터가 하나뿐이고 그 타입을 컴파일러가 추론할 수 있는 경우 it을 사용할 수 있다.

```kotlin
people.maxBy { it.age }
```

**람다를 변수에 저장할 때는 파라미터의 타입을 추론할 문맥이 존재하지 않는다. 따라서 파라미터 타입을 명시해야 한다.**

```kotlin
val getAge = { p: Person -> p.age }
```

### 현재 영역에 있는 변수에 접근

람다를 함수 안에서 정의하면 함수의 파라미터뿐 아니라 람다 정의의 밖에 선언된 로컬 변수까지 **람다에서 모두 사용할 수 있다**.

이해하기 쉽게 forEach 표준 함수를 살펴보자.

```kotlin
fun printMessageWithPrefix(messages: Collection<String>, prefix: String) {
	messages.forEach { // 각 원소에 대해 수행할 작업을 람다로 받음
		println("$prefix $it") // 람다 안에서 파라미터를 사용
	}
}
```

자바와 차이점 중 중요한 점은 코틀린 람다 안에서는 파이널(final) 변수가 아닌 변수에 접근할 수 있다. 또한 람다 안에서 바깥의 변수를 변경해도 된다.

```kotlin
fun printProblemCounts(responses: Collection<String>) {
	var clientErrors = 0
	var serverErrors = 0
	responses.forEach {
		if (it.startsWith("4")) {
			clientErrors++ //람다 안에서 람다 밖의 변수를 변경
		} else if (it.startsWith("5")) {
			serverErrors++ //람다 안에서 람다 밖의 변수를 변경
		}
	}
}
```

이와 같이 **람다 안에서 사용하는 외부 변수를 `람다가 포획(cature)한 변수`** 라고 부른다.

*   클로저(closure)

    클로저는 Outer Scope(상위 함수의 영역)의 변수를 접근할 수 있는 함수를 말한다. 코틀린은 클로저를 지원하기 때문에 익명함수는 함수 밖에서 정의된 변수에 접근할 수 있다.

    ```kotlin
    fun add(x: Int): (Int) -> Int {
    	return fun(y: Int): Int {
    		return x+y //x 변수는 외부의 파라미터이지만 접근 가능
    	}
    }
    ```

기본적으로 함수 안에 정의된 로컬 변수의 생명주기는 함수가 반환되면 끝나지만, **어떤 함수가 자신의 로컬 별수를 포획한 람다를 반환**하거나 **다른 변수에 저장한**다면 로컬 변수와 함수의 생명주기가 달라질 수 있다.

포획한 변수가 있는 람다를 저장해서 함수가 끝난 뒤에 실행해도 람다의 본문 코드는 여전히 포획한 변수를 읽거나 쓸 수 있다.

파이널 변수를 포획한 경우 → 람다 코드를 변수 값과 함께 저장

파이널이 아닌 변수를 포획한 경우 → 변수를 특별한 래퍼로 감싸서 나중에 변경하거나 읽을 수 있게 한 다음, 래퍼에 대한 참조를 람다 코드와 함께 저장

*   **변경 가능한 변수 포획하기: 자세한 구현 방법**

    자바에서는 파이널 변수만 포획할 수 있지만 속임수를 통해 변경 가능한 변수를 포획할 수 있다.

    1. 변경 가능한 변수를 저장하는 원소가 단 하나뿐인 배열 선언하기
    2. 변경 가능한 변수를 필드로 하는 클래스를 선언하기

    안에 들어있는 원소는 변경이 가능해도 배열 or 클래스의 인스턴스에 대한 참조를 파이널로 만들면 포획이 가능하다.

    ```kotlin
    class Ref<T>(var value: T) {
    	val counter = Ref(0)
    	val inc = { counter.value++ }
    	//공식적으로는 변경 불가능한 변수를 포획했지만, 그 변수가 가리키는 객체의 필드 값은 바꿀 수 있음

    	...

    	val counter2 = 0
    	val inc2 = { counter++ }
    }
    ```

우리는 다음과 같은 함정에 빠지면 안된다. 람다를 이벤트 핸들러나 다른 비동기적으로 실행되는 코드로 활용하는 경우, 함수 호출이 끝난 다음에 로컬 변수가 변경될 수도 있다.

```kotlin
fun tryToCountButtonClicks(button: Button): Int {
	var clicks = 0 
	button.onClick { clicks++ } 
	return clicks
}
```

위의 코드의 clicks 값은 제대로 동작할까? 예상한대로 항상 0을 반환할 것이다.

onClick 핸들러는 호출될 때마다 clicks의 값을 증가시키지만, 핸들러는 tryToCountButtonClicks가 clicks를 반환한 다음에 호출되기 때문이다.

이러한 함정을 해결하기 위해서는 함수의 내부가 아닌 클래스의 프로퍼티나 전역 프로퍼티 등의 위치로 빼내서 사용해야 한다.

### 멤버 참조

람다를 통해 넘기려는 코드가 이미 함수인 경우에는, 그 함수를 호출하는 람다를 만들면 되지만 이는 중복이다. 이를 해결하기 위해 이중 콜론(::)을 사용해서 함수를 값으로 바꿔 넘길 수 있다.

```kotlin
val getAge = Person::age
```

::을 사용하는 식을 멤버 참조(member reference)라고 부른다. 멤버 참조는 **프로퍼티나 메서드를 단 하나만 호출하는 함수 값을 만들어준다**. 주의할 점은 참조 대상이 함수 혹은 프로퍼티인지와는 관계없이 멤버 참조 뒤에는 괄호를 넣으면 안된다.

```kotlin
/*class*/
Person::age
     /*member*/
```

멤버 참조는 그 멤버를 호출하는 람다와 같은 타입이다. 또한 최상위에 선언된(그리고 다른 클래스의 멤버가 아닌) 함수나 프로퍼티를 참조할 수도 있다.

```kotlin
fun salute() = println("Salute!")
run(::salute)
```

람다가 인자가 여러 개인 다른 함수한테 작업을 위임하는 경우, 람다를 정의하지 않고 **직접 위임 함수에 대한 참조를 제공**하면 편리하다.

```kotlin
val action = { p: Person, message: String -> //sendEmail 함수에게 작업 위임
	sendEmail(p, message)
}
val nextAction = ::sendEmail //람다 대신 멤버 참조를 쓸 수 있음
```

생성자 참조(constructor reference)를 사용하면 클래스 생성 작업을 연기하거나 저장해둘 수 있다.

```kotlin
val createPerson = ::Person //Peson의 인스턴스를 만드는 동작을 값으로 저장
val p = createPerson("준형", 29)
```

확장 함수도 멤버 함수와 같은 방식으로 참조할 수 있다.

```kotlin
fun Person.isAdult() = age >= 21
val predicate = Person::isAdult
```

## 컬렉션 함수형 API

### filter & map

filter 함수는 컬렉션을 이터레이션하면서 주어진 람다에 각 원소를 넘겨 람다가 true를 반환하는 원소만 모은다.

```kotlin
val list = listOf(1,2,3,4)
list.filter { it % 2 == 0 }
>>>[2, 4]
```

결과 값은 입력 컬레션의 원소 중에서 주어진 술어(참/거짓을 반화나는 함수를 술어(predicate)라고 함)를 만족하는 원소만으로 이뤄진 **새로운 컬렉션**이다.

filter는 컬렉션에서 원치 않는 원소를 제거하지만 원소를 변환할 수는 없다.

map 함수는 주어진 람다를 **컬렉션의 각 원소에 적용한 결과를 모아서 새로운 컬렉션**을 만든다.

```kotlin
val list = listOf(1,2,3,4)
list.map { it * it }
>>>[1, 4, 9, 16]
```

결과 값은 원본 리스트와 개수는 같지만, 각 원소는 주어진 함수에 따라 변환된 새로운 컬렉션이다.

### all, any, count, find ⇒ 컬렉션에 술어 적용

all, any 함수는 컬렉션의 모든 원소가 어떤 조건을 만족하는지 판단하는 연산(컬렉션 안에 어떤 조건을 만족하는 원소가 있는지를 판단하는)이다.

count 함수는 조건을 만족하는 원소의 개수를 반환한다.

find 함수는 조건을 만족하는 첫 번째 원소를 반환한다.

```kotlin
val canBeInClub = { p: Person -> p.age <= 27 }

//모든 원소가 해당 술어를 만족하는가? -> all 
val people = listOf(Person("준형", 29), Person("길동", 26))
people.all(canBeInClub)

//술어를 만족하는 원소가 하나라도 있는가? -> any
people.any(canBeInClub)

//술어를 만족하는 원소의 개수 -> count
people.count(canBeInClub)

//술어를 만족하는 원소를 하나 찾고 싶은가? -> find
people.find(canBeInClub)
```

### groupBy ⇒ 리스트를 여러 그룹으로 이뤄진 맵으로 변경

컬렉션의 모든 원소를 어떤 특성에 따라 여러 그룹으로 나누고 싶을 때 groupBy 함수를 사용하면 된다.

예를 들어, 나이에 따라 나누고 싶을 때의 코드를 살펴보자.

```kotlin
val people = listOf(Person("준형", 29), ... ,Person("철수", 29), Person("길동", 30))
people.groupBy { it.age }
```

groupBy 함수는 컬렉션의 원소를 구분하는 특성(ex: age)이 key이고, key 값에 따른 각 그룹(Person 객체의 모임)이 value인 Map 형태이다. ⇒ 결과 타입: Map\<Key, Value>

### flatMap & flatten ⇒ 중첩된 컬렉션 안의 원소 처리

flatMap 함수는 먼저 인자로 주어진 람다를 컬렉션의 모든 객체에 적용하고(혹은 매핑) 람다를 적용한 결과가 얻어지는 여러 리스트를 한 리스트로 모아주게 한다.

```kotlin
val strings = listOf("abc", "def")
strings.flatMap { it.toList() }
>>>[a,b,c,d,e,f]
```

flatMap 함수를 사용할 때 중복되는 리스트의 값이 있을 경우 없애고 싶을 때 `toSet()` 를 사용하면 된다.

flatMap과 다르게 특별히 변환해야 할 내용이 없고 모두 합칠 때는 flattern 함수를 사용하면 된다.

## 지연 계산(lazy) 컬렉션 연산

앞에서 살펴본 컬렉션 함수들을 연쇄적으로 처리하게 되면 매 단계마다 계산 중간 결과를 새로운 컬렉션에 임시로 담는다. 시퀀스를 사용하면 중간 임시 컬렉션을 사용하지 않고 연쇄적으로 연산할 수 있다.

```kotlin
people
	.map { Person::name } //중간 연산을 할 때마다 새로운 리스트 생성
	.filter { it.startsWith("A") }

people.asSequence() //원본 컬렉션을 시퀀스로 변환
	.map(Person::name) //시퀀스도 컬렉션과 같은 API 제공
	.filter { it.startsWith("A") } //시퀀스도 컬렉션과 같은 API 제공
	.toList() //결과 시퀀스를 다시 리스트로 변환
```

시퀀스를 사용하면 중간 결과를 저장하는 컬렉션이 생기지 않기 때문에 원소가 많은 경우 성능이 눈에 띄게 좋아진다.

코틀린 지연 계산 시퀀스는 Sequence 인터페이스를 통해 이뤄진다. Sequence 인터페이스는 **단지 한 번에 하나씩 열거될 수 있는 원소의 시퀀스를 표현**할 뿐이다.

<figure><img src="../../../.gitbook/assets/Untitled (13).png" alt=""><figcaption></figcaption></figure>

iterator 메서드를 통해 시퀀스로부터 원소 값을 얻을 수 있다.

Sequence 인터페이스는 **원소가 필요할 때 계산되기 때문에** 중간 처리 결과를 저장하지 않아도 연산을 연쇄적으로 적용하여 효율적으로 계산을 수행할 수 있다.

`asSequence` 확장 함수를 호출하면 어떤 컬렉션이든 시퀀스로 바꿀 수 있다.

위의 스니펫 코드를 살펴보면 시퀀스를 다시 컬렉션(리스트)로 변환하는 것을 알 수 있다. 그냥 컬렉션보다 시퀀스가 더 낫다고 설명했는데 왜 변환해야 하는 것일까? 무조건 시퀀스를 사용하는 것이 좋다고는 할 수 없다.

시퀀스의 원소를 차례로 이터레이션해야 한다면 시퀀스를 직접 사용해도 좋다. 하지만 시퀀스 원소를 인덱스를 사용하여 접근, 다른 API 메서드가 필요한 경우와 같을 때는 시퀀스를 리스트로 변환해야 한다.



💡 언제 시퀀스를 사용하는 것이 좋을까? 큰 컬렉션에 대해서 연산을 연쇄시킬 때는 시퀀스를 사용하는 것을 규칙으로 정하는 것이 좋다. 컬렉션에 들어있는 원소가 많으면 중간 원소를 재배열하는 비용이 커지기 때문에 지연 계산이 좋다.



### 시퀀스 연산 실행: 중간 연산 & 최종 연산

시퀀스에 대한 연산은 중간 연산과 최종 연산으로 나뉜다.

중간 연산 → 다른 시퀀스를 반환. 해당 시퀀스는 최초 시퀀스의 원소를 변환하는 방법을 인지하고 있음

최종 연산 → 결과를 반환. 결과는 최초 컬렉션에 대해 변환을 적용한 시퀀스로부터 일련의 계산을 수행하여 얻을 수 있는 컬렉션, 원소, 숫자, 객체를 뜻한다.

```kotlin
         /*       중간연산            최종연산 */
sequence.map { ... }.filter { ... }.toList()
```

```kotlin
listOf(1,2,3,4).asSequence()
         .map { print("map $it "); it * it }
         .filter { print("filter $it"); it % 2 == 0 }

>>> 아무 것도 출력 되지 않음

listOf(1,2,3,4).asSequence()
         .map { print("map $it "); it * it }
         .filter { print("filter $it "); it % 2 == 0 }
         .toList()

>>> map 1 filter 1 map 2 filter 4 map 3 filter 9 map 4 filter 16
```

위의 코드를 보면 중간 연산만 하는 시퀀스와 최종 연산까지 하는 시퀀스를 볼 수 있다.

중간 연산은 **항상 지연 계산**된다. 그러므로 시퀀스는 최종 연산 없이는 데이터를 처리하지 않고, 지연 계산을 한다.

출력되는 순서를 보면 모든 원소에 대해 순차적으로 진행되는 것을 알 수 있다. 즉, 첫 번째 원소 처리 → 두 번째 원소 처리 … 순으로 적용된다.

따라서 원소에 연산을 차례대로 처리하다가 원하는 결과를 얻게 되면, 그 이후의 원소에 대해서 변환이 이뤄지지 않을 수 있다. 즉, 즉시 계산은 전체 컬렉션에 연산을 적용하지만 지연 계산은 원소를 하나씩 처리한다.

또한, 컬렉션에 대해 수행하는 연산의 순서도 성능에 영향을 끼친다.



💡 자바 스트림 vs 코틀린 시퀀스 코틀린의 일부 시퀀스는 여러 번 순회할 수 있고, 그렇지 못한 시퀀스는 여러 번 순회가 불가능하다. 자바의 스트림은 병렬처리가 가능하기 때문에, 이 부분을 사용하려면 스트림을 사용해야 한다.



### 시퀀스 만들기

시퀀스를 만들 때 `generateSequence` 함수를 사용하면 된다.

`generateSequence` 함수는 2개의 인자(초기값, 시퀀스에 속하게 될 다음 값을 생산하는 함수)를 받는다.

최종 연산이 수행될 때 부터 연산이 완벽히 끝날 때까지 해당 `Sequence Generator` 는 계속해서 `Sequence` 원소를 생성한다.

### 시퀀스 일부 가져오기

`take(count)` 혹은 `takeWhile(predicate)` 를 사용한다.

`generatorSequence` 에 null을 리턴하는 람다를 사용하면 null이 발생했을 때, 해당 시퀀스가 종료된다.

*   처음 N개의 소수 찾기(`take` 사용)

    ```kotlin
    fun firstNPrimes(count :Int) = generatorSequence(2, ::nextPrime)
    	.take(count)
    	.toList()
    ```
*   주어진 수보다 작은 소수(생성 함수에서 null 리턴)

    ```kotlin
    fun primeLessThan(max: Int): List<Int> =
    	generatorSequence(2) { n -> if (n < max) nextPrime(n) else null }
    		.toList()
    		.dropLast(1) //한계 값을 넘는 소수를 하나 포함하므로 리턴하기 전에 잘라냄
    ```
*   주어진 수보다 작은 소수(takeWhile 사용)

    ```kotlin
    fun primeLessThan(max: Int): List<Int> = 
    	generatorSequence(2, ::nextPrime)
    		.takeWhile { it < max }
    		.toList()
    ```

    #### filter와 takeWhile의 차이

    둘 다 모두 시퀀스를 반환하는 중간 연산이다.

    다만, filter는 전체 원소를 모두 순회하는 반면, takeWhile은 특정 조건을 만족하지 않는 경우(predicate의 결과가 false인 경우) 순회를 멈추고 시퀀스를 반환한다.

    또한, 무한 시퀀스를 사용한 경우 filter는 종료되지 않을 가능성이 있다.

    ## 자바 함수형 인터페이스 활용

    코틀린 람다를 자바 API에 활용할 수 있는지에 대해서 알아본다.

    ```java
    public interface onClickLisnter {
    	void onClick(View view); // ----> { view -> ... } 
    	//람다의 파라미터는 메서드의 파라미터와 대응한다.
    }
    ```

    위의 코틀린 코드에서 타입을 알 수 있는 이유는 onClickListener에 추상 메서드가 단 하나만 있기 때문이다.

    이러한 인터페이스를 함수형 인터페이스 또는 SAM(Single Abstract Method) 인터페이스라고 한다.

    ### 자바 메서드에 람다를 인자로 전달

    함수형 인터페이스를 인자로 원하는 자바 메서드에 코틀린 람다를 전달할 수 있다.

    ```kotlin
    /* java */
    void postponeComputation(int delay, Runnable computation);

    /* kotlin */
    postponeComputation(1000) { println(42) }
    ```

    위 코드처럼 코틀린에서 람다를 자바 메서드에 전달할 수 있다. 컴파일러는 자동으로 람다를 Runnable 인스턴스로 변환해준다.

    ‘Runnable 인스턴스’의 뜻은 Runnable을 구현한 무명 클래스의 인스턴스라는 뜻이다. 컴파일러는 자동으로 무명 클래스와 인스턴스를 만들어준다.

    무명 클래스에 있는 유일한 추상 메서드를 구현할 때 람다 본문을 메서드 본문으로 사용한다.

    ```kotlin
    //Runnbale을 구현하는 무명 객체를 명시적으로 사용
    postponeComputation(1000, object: Runnable {
    	override fun run() {
    		println(42)
    	}
    })

    //람다 사용
    postponeComputation(1000) { println(42) }

    //람다가 주변 영역의 변수를 포획한 경우
    fun handleComputation(id: String) {
    	postponeComputation(1000) { println(42) }
    }
    ```

    무명 객체를 사용하면(객체를 명시적으로 선언하는 경우) 메서드를 호출할 때마다 새로운 객체가 생성된다.

    람다의 경우, 프로그램 전체에서 Runnable의 인스턴스는 단 하나만 만들어진다. 즉, 무명 객체의 메서드를 호출할 때마다 반복해서 사용한다.

    람다가 주변 영역의 변수를 포획한 경우, 매 호출마다 새로운 인스턴스를 생성한다.

    컬렉션을 확장한 메서드에 람다를 넘기는 경우, 코틀린은 앞서 설명한 방법을 사용하지 않고 inline 키워드를 사용한다. inline으로 표시된 코틀린 함수에게 람다를 넘기면 아무런 무명 클래스가 만들어지지 않는다. 그 이유는 람다의 본문이 호출하는 코드에 복사되기 때문에 굳이 무명 클래스를 만들어주지 않아도 되기 때문이다.

### SAM 생성자: 람다를 함수형 인터페이스로 명시적으로 변경

SAM 생성자는 **람다를 함수형 인터페이스의 인스턴스로 변환**할 수 있게 컴파일러가 자동으로 생성한 함수다.

```kotlin
fun createAllDoneRunnable(): Runnable {
	return Runnable { println("All Done") }
}

fun main() {
	createAllDoneRunnable().run()
}
```

함수형 인터페이스(Runnable)의 인스턴스를 반환하는 메서드가 있다면 람다를 직접 반환할 수 없다. 그래서 반환하고픈 람다를 SAM 생성자로 감싸야 한다.

SAM 생성자의 이름은 사용하려는 함수형 인터페이스의 이름과 같다.

SAM 생성자는 함수형 인터페이스의 유일한 추상 메서드의 본문에 사용할 람다만을 인자로 받아, 함수형 인터페이스를 구현하는 클래스의 인스턴스를 반환한다.

람다로 생성한 함수형 인터페이스 인스턴스를 변수에 저장해야 하는 경우에도 SAM 생성자를 사용할 수 있다.

```kotlin
val listener = OnClickListener { view ->
	val text = when(view.id) {
		R.id.button1 -> "First Button"
		R.id.button2 -> "Second Button"
		else -> "Unknown Button"
	}
	toast(text)
}
```

💡 람다와 리스너 등록/해제하기 람다는 무명 객체와 달리 인스턴스 자신을 가리키는 this가 없다는 사실에 유의해야 한다. 즉, 람다를 변환한 무명 클래스의 인스턴스를 참조할 방법이 없다. 람다 안에서 this는 그 람다를 둘러싼 클래스의 인스턴스를 가리킨다.



함수형 인터페이스를 요구하는 메서드를 호출할 때 대부분의 SAM 변환을 컴파일러가 자동으로 해주지만, 가끔 오버로드한 메서드 중에서 어떤 타입의 메서드를 선택하여 람다를 변환해 넘겨줘야 할지 모호할 때가 있다.

이럴때 역시 SAM 생성자를 사용해서 명시적으로 넘겨줘 컴파일 에러를 피할 수 있다.

## 수신 객체 지정 람다: with & apply

수신 객체 지정 람다란, 수신 객체를 명시하지 않고 람다의 본문 안에서 다른 객체의 메서드를 호출할 수 있게 해주는 기능이다.

### with 함수

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

with 함수는 첫 번째 인자로 받은 객체를 두 번째 인자로 받은 람다의 수신 객체로 만든다.

`with(firstObject, { … })`

인자로 받은 람다 본문에서는 this를 사용하여 그 수신 객체에 접근할 수 있다.

with 함수가 반환하는 값은 람다 코드를 실행한 결과이며, 그 결과는 람다 식의 본문에 있는 마지막 식의 값이다.

```kotlin
fun alphabet() = with(StringBuilder()) {
	for (letter in 'A'...'Z') {
		append(letter)
	}
	
	append("\\n Now I know the alphabet")
	toString()
}
```

*   수신 객체 지정 람다 vs 확장 함수

    확장 함수의 `this`는 그 함수가 확장하는 타입의 인스턴스를 가리킨다. 그리고 수신 객체 this의 멤버를 호출할 때는 `this.` 를 생략할 수 있다.

    정확한 비교는 아니지만, 어떤 의미에서는 확장 함수를 수신 객체 지정 함수라 할 수 있다.

    `일반 함수 == 일반 람다` , `확장 함수 == 수신 객체 지정 람다`

### apply 함수

apply 함수는 항상 자신에게 전달된 객체(수신 객체)를 반환한다.

```kotlin
fun alphabet() = StringBuilder().apply {
	for (letter in 'A'...'Z') {
		append(letter)
	}

	append("\\n Now I know the alphabet")
}.toString()
```

apply의 수신 객체 ⇒ 전달 받은 람다의 수신 객체(StringBuiler Object)

apply 함수는 객체의 인스턴스를 만들면서 즉시 프로퍼티 중 일부를 초기화 할 때 주로 사용된다.
