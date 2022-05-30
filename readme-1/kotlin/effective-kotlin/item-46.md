---
description: 함수 타입 파라미터를 갖는 함수에 inline 한정자를 붙여라
---

# item 46

### 함수를 왜 인라인으로 선언해야 하는 경우

inline 한정자를 붙이지 않은 경우에 decompile을 해보면 main 함수에서 repeat 함수를 그저 호출만 하는 것을 확인할 수 있습니다. 또한, 함수 본문(repeat 함수)으로 점프하고, 본문의 모든 문장을 호출한 뒤에 함수를 호출했던 위치로 다시 점프하는 과정을 거칩니다.

```kotlin
fun main() {
    repeat(10) {
        println(it)
    }
}

fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}
```

```java
public final class InlineKt {
   public static final void main() {
      repeat(10, (Function1)null.INSTANCE);
   }

   public static void main(String[] var0) {
      main();
   }

   public static final void repeat(int times, @NotNull Function1 action) {
      Intrinsics.checkNotNullParameter(action, "action");
      int index = 0;

      for(int var3 = times; index < var3; ++index) {
         action.invoke(index);
      }

   }
}
```

위의 예제처럼, 람다를 사용하는 구현은 똑같은 작업을 수행하는 일반 함수를 사용한 구현보다 덜 효율적입니다. 하지만 inline 한정자를 붙이게 되면 **컴파일 시점에 ‘함수를 호출하는 부분’을 ‘함수의 본문’으로 대체**, 즉 main 함수에 함수를 복사하는 것을 확인할 수 있습니다. 이제 **컴파일러가 그 함수를 호출하는 모든 문장을 함수 본문에 해당하는 바이트코드로 교체**할 것입니다. 또한, 위에서는 점프가 발생했지만 inline 한정자를 붙이면 점프가 일어나지 않습니다.

```kotlin
fun main() {
    repeat(10) {
        println(it)
    }
}

inline fun repeat(times: Int, action: (Int) -> Unit) {
    for (index in 0 until times) {
        action(index)
    }
}
```

```java
public final class InlineKt {
   public static final void main() {
      int times$iv = 10;
      int $i$f$repeat = false;
      int index$iv = 0;

      for(byte var3 = times$iv; index$iv < var3; ++index$iv) {
         int var5 = false;
         System.out.println(index$iv);
      }

   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final void repeat(int times, @NotNull Function1 action) {
      int $i$f$repeat = 0;
      Intrinsics.checkNotNullParameter(action, "action");
      int index = 0;

      for(int var4 = times; index < var4; ++index) {
         action.invoke(index);
      }

   }
}
```

### 타입 아규먼트를 reified로 사용할 수 있다

자바에는 제네릭을 사용할 수 있지만, JVM 바이트 코드에서는 제네릭이 존재하지 않습니다. 자바와 마찬가지로 코틀린 역시 **컴파일을 하면, 제네릭 타입과 관련된 내용이 제거**됩니다. 즉, 제네릭 클래스 인스턴스가 해당 인스턴스를 생성할 때 쓰인 **타입 인자에 대한 정보를 유지하지 않습니다.**

예를 들어, List\<Int> 객체를 만들고 그 안에 정수를 여러개 넣어도 실행 시점에는 오직 List로만 봅니다. 그 List 객체가 어떤 타입의 원소를 저장하는지 실행 시점에는 알 수 없습니다.

List\<String>, List\<Int> 두 개의 객체가 있다고 했을 때, 컴파일러는 두 리스트를 서로 다른 타입으로 인식하지만 실행 시점에는 이 둘을 완전히 같은 객체로 바라봅니다. 이는 컴파일러가 타입 인자를 알고 올바른 타입의 값만 각 리스트에 넣도록 보장하기 때문입니다. 그러므로 List 인지 확인하는 스타 프로젝션은 가능하지만 타입을 명시하게 되면 타입을 검사하지 못합니다.

```kotlin
any is List<Int> // error
any is List<*> // ok

fun <T> printTypeName() {
	print(T::class.simpleName) // error
}
```

이런 제약을 피하기 위해 인라인 함수를 사용합니다. **인라인 함수의 파라미터는 실체화되므로 실행 시점에 인라인 함수의 타입 인자를 알 수 있습니다**. 함수 호출이 본문으로 대체되므로, reified 한정자를 사용해서 타입 파라미터를 사용한 부분이 타입 아규먼트로 대체됩니다.

```kotlin
inline fun <reified T> printTypeName() {
	print(T::class.simpleName)
}

printTypeName<Int>()
printTypeName<Char>()
printTypeName<String>()

------------------------------------------------------------------------------

// 컴파일 시 
print(Int::class.simpleName) // Int
print(Char::class.simpleName) // Char
print(String::class.simpleName) // String
```

표준 라이브러리 filterIsInstance 함수를 살펴보면, 인자로 받은 컬렉션의 원소 중에서 타입 인자로 지정한 클래스의 인스턴스만을 모아서 리스트를 반환한다.

타입 인자를 실행 시점에 알 수 있으므로 filterIsInstance는 그 타입 인자를 사용해 리스트의 원소 중에 타입 인자와 타입이 일치하는 원소만을 추려낼 수 있습니다.

```kotlin
public inline fun <reified R> kotlin.collections.Iterable<*>.filterIsInstance()
: kotlin.collections.List<@kotlin.internal.NoInfer R> { 
	val destination = mutableListOf<R>()
	for (element in this) {
		if (element is R) { // 각 원소가 타입 인자로 지정한 클래스의 인스턴스인지 검사할 수 있음.
			destination.add(element)
		}
	}
	return destination
}

...

val items = listOf("one", 2, "three")
items.filterIsInstance<String>()
// one, three 
```

_**💡 인라인 함수에서만 실체화한 타입 인자를 쓸 수 있는 이유는??**_

앞서 살펴본 것처럼, 컴파일러는 인라인 함수의 본문을 구현한 바이트코드를 그 함수가 호출되는 모든 시점에 삽입한다. **컴파일러는 실체화한 타입 인자를 사용해서 인라인 함수를 호출하는 각 부분의 정확한 타입의 인자를 알 수 있다**. 타입 파라미터가 아닌 ‘구체적인 타입’을 사용하므로 만들어진 바이트코드는 실행 시점에 벌어지는 타입 소거의 영향을 받지 않는다.

### 함수 타입 파라미터를 가진 함수가 훨씬 빠르게 동작한다

inline 한정자를 붙이면 함수 호출과 리턴을 위해 점프하는 과정과 백스택을 추적하는 과정이 없기 때문에 조금 더 빠르게 동작합니다. 이러한 이유로 표준 라이브러리에 있는 간단한 함수들에는 대부분 inline 한정자가 붙어 있습니다.

```kotlin
inline fun print(message: Any?) {
	System.out.print(message)
}
```

하지만 **함수 파라미터를 가지지 않는 함수에서는 이러한 차이가 성능에 영향을 주지 않는 것**을 주의해야 합니다.

함수 리터럴을 사용해서 만들어진 객체는 어떤 방식으로든 저장되고 유지되어야 합니다. 코틀린/JS에서 함수는 단순한 함수 또는 함수 레퍼런스인 반면, 코틀린/JVM에서는 **JVM 익명 클래스** 또는 **일반 클래스**를 기반으로 함수를 객체로 만들어 냅니다.

```kotlin
val lambda: ()->Unit = {
	...
}

//클래스로 컴파일
Function0<Unit> lambda = new Function0<Unit>() {
	public Unit invoke() {
		...
	}
}

//별도의 파일에 정의되어 있는 일반 클래스로 컴파일
public class Test$lambda implements Function0<Unit> {
	public Unit invoke() {
		...
	}
}

//사용
Function0 lambda = new Test$lambda()
```

두 결과는 큰 차이가 없다는 것을 확인할 수 있습니다.

_**💡 JVM에서 아규먼트가 없는 함수 타입은 Function0 타입으로 변환된다.**_

* ()→Unit ⇒ Function0\<Unit>로 컴파일
* ()→Int ⇒ Function0\<Int>로 컴파일
* (Int)→Int ⇒ Funcation1\<Int, Int>로 컴파일
* (Int, Int)→Int ⇒ Function2\<Int, Int, Int>로 컴파일

위에서 봤듯이, **모든 인터페이스는 모두 코틀린 컴파일러에 의해서 생성됩니다**. 요청이 있을 때 생성되므로, 이를 명시적으로 사용할 수 없습니다.

대신 함수 타입을 사용해서 추가적인 가능성들을 볼 수 있습니다.

```kotlin
class OnClickListener: ()->Unit {
	override fun invoke() {
		...
	}
}
```

또한, **함수 본문을 객체로 랩(wrap)하면, 코드의 속도가 느려집니다**.

```kotlin
inline fun repeat(times: Int, action: (Int)->Unit) {
	for (index in 0 until times) {
		action(index)
	}
}

fun repeat(times: Int, action: (Int)->Unit) {
	for (index in 0 until times) {
		action(index)
	}
}
```

위의 두 개의 함수는 inline 한정자 이외에는 동일한 로직을 가지고 있습니다. 언뜻 보기엔 큰 차이가 없다고 느껴질 수 있습니다. 그렇다면 테스트를 해봅시다.

```kotlin
@Benchmark
fun Inline(blackhole: Blackhole) {
	repeat(100_000_000) {
		blackhole.consume(it)
	}
}

@Benchmark
fun noInline(blackhole: Blackhole) {
	noInlineRepeat(100_000_000) {
		blackhole.consume(it)
	}
}
```

저자의 테스트 결과 첫 번째 코드는 평균 189ms, 두 번째 코드는 477ms로 동작하는 것을 확인할 수 있었습니다.

첫 번째 함수 → 숫자로 반복을 돌면서, 빈 함수를 호출

두 번째 함수 → 숫자로 반복을 돌면서, 객체를 호출하고, 이 객체가 빈 함수를 호출

즉, 이러한 코드의 **실행 방식 차이**로 인해 속도가 발생하는 것입니다.

‘인라인 함수' vs ‘논 인라인 함수’의 가장 중요한 차이점은 무엇일까요? **함수 리터럴 내부에서 지역 변수를 캡처**할 때 확인할 수 있습니다.

캡처된 값은 객체로 래핑(wrapping)해야 하며, 사용할 때마다 객체를 통해 작업이 이루어져야 합니다.

```kotlin
var l = 1L
noinlineRepeat(100_000_000) {
	l += it
}
```

인라인이 아닌 람다 표현식에서는 지역 변수 l을 직접 사용할 수 없습니다. l은 컴파일 과정 중에 레퍼런스 객체로 래핑되고, 람다 표현식 내부에서는 이를 사용합니다.

```kotlin
val a = Ref.LongRef()
a.element = 1L
noinlineRepeat(100_000_000) {
	a.element = a.element + it
}
```

함수가 객체로 컴파일되고, 지역 변수가 래핑되어 발생하는 문제가 누적되면 성능에 많은 영향을 미칠 수 있습니다. 즉, **함수 타입 파라미터를 활용해서 유틸리티 함수를 만들 때(ex: 컬렉션 처리)는 인라인 한정자를 붙이는 것을 지향**해야 합니다.

### 비지역적 리턴(non-local return)을 사용할 수 있다

inline 한정자를 붙여주지 않으면 내부에서 리턴을 사용할 수 없습니다. 이는 **함수 리터럴이 컴파일될 때, 함수가 객체로 래핑되어서 발생하는 문제**입니다. 함수가 다른 클래스에 위치하므로, return을 사용해서 main으로 돌아올 수 없는 것입니다.

![](<../../../.gitbook/assets/Untitled (3).png>)

람다 안에서 return을 사용하면 람다로부터만 반환되는게 아니라 그 람다를 호출하는 함수가 실행을 끝내고 반환됩니다. return이 바깥쪽 함수를 반환시키고 싶다면 람다를 인자로 받는 함수가 inline 함수여야만 합니다.

```kotlin
fun main() {
    val person = listOf(Person("Jun", 29))
    lookForJun(person)
}

fun lookForJun(people: List<Person>) {
    people.forEach {
        if (it.name == "Jun"){
            println("find!!")
            return // forEach는 inline 함수이므로 바깥쪽 함수 lookForJun 함수를 리턴한다.
        }
    }

    println("non local return")
}
```

인라이닝되지 않는 함수에 전달되는 람다 안에서는 return을 사용할 수 없다는 것을 유의하자.

#### 람다로부터 반환: 레이블을 사용한 return

로컬 리턴과 넌로컬 리턴을 구분하기 위해 레이블(label)을 사용해야 합니다.

```kotlin
fun lookForJun(people: List<Person>) {
    people.forEach {
        if (it.name == "Jun"){
           return@forEach // 람다식으로부터 반환
    }

    println("non local return") // 항상 출력
}
```

#### 무명 함수: 기본적으로 로컬 return

무명 함수 안에서 레이블이 붙지 않은 return 식은 무명 함수 자체를 반환시킬 뿐, 무명 함수를 둘러싼 다른 함수를 반환시키지 않습니다.

무명 함수는 fun을 사용해 정의되므로 return 기본 규칙에 따라 함수 자신이 바로 가장 안쪽에 있는 정의된 함수를 return 합니다.

```kotlin
fun lookForJun(people: List<Person>) {
	people.forEach(fun(person)) {
		if (person.name == "Jun") return //fun(person)에 대해 return
	}
}

fun lookForJun(people: List<Person>) {
	people.forEach {
		if (person.name == "Jun") return //lookForJun에 대해 return
	}
}
```

### inline 한정자의 비용

재귀적으로 사용하면, 무한하게 대체되는 문제가 발생합니다.

```kotlin
inline fun a() { b() }
inline fun b() { c() }
inline fun c() { a() }
//함수 내부에 코드가 대체되기 때문에 무한히 대체되는 문제가 발생합니다.
```

인라인 함수는 구현을 숨길 수 없으므로, 클래스에서도 거의 사용되지 않습니다.

```kotlin
internal inline fun read() {
	val reader = Reader() //error
	...
}

private class Reader {
	...
}
```

inline 한정자를 남용하면, 코드의 크기가 쉽게 커질 수 있으므로 주의해야 합니다.

람다를 사용하는 모든 함수를 인라이닝 할 수 없습니다. 함수가 인라이닝될 때 그 함수에 인자로 전달된 람다 식의 본문은 결과 코드에 직접 들어갈 수 있습니다. 하지만 이렇게 **람다가 본문에 직접 펼쳐지기(대체) 때문에 함수가 파라미터로 전달받은 람다를 본문에 사용하는 방식**이 한정적일 수 밖에 없습니다.

인라인 함수의 본문에서 람다 식을 바로 호출하거나 람다 식을 인자로 전달받아 바로 호출하는 경우에는 그 람다를 인라이닝 할 수 있습니다.

그런 경우가 아니라면 “Illegal useage of inline-parameter” 메시지와 함께 인라이닝을 금지시킵니다. 파라미터로 받은 람다를 다른 변수에 저장하고 나중에 그 변수를 사용한다면, **람다를 표현하는 객체가 어딘가는 존재해야 하기 때문에** 람다를 인라이닝할 수 없습니다.

```kotlin
fun <T, R> Sequence<T>.map(transform: (T)->R): Sequence<R> {
	return TransformingSequence(this, transform)
}
```

위의 map 함수를 보면 transform 파라미터로 전달받은 함수 값을 호출하는 대신, TransformingSequence 클래스의 생성자에게 그 함수 값을 넘기는 것을 볼 수 있습니다.

이런 기능을 지원하려면 map에 전달되는 transform 인자를 일반적인(인라이닝하지 않은) 함수 표현식으로 만들 수 밖에 없습니다. 즉, tramsform을 함수 인터페이스를 구현하는 무명 클래스 인스턴스로 만들어야만 합니다.

### crossinline과 noinline

#### crossinline

아규먼트로 인라인 함수를 받지만, 비지역적 리턴을 하는 함수는 받을 수 없게 만듭니다.

인라인으로 만들지 않은 다른 람다 표현식과 조합해서 사용할 때 문제가 발생하는 경우 활용합니다.

![](<../../../.gitbook/assets/Untitled (4) (1).png>)

함수를 인자로 받아 setOnClickListener 내부에서 호출해야 하지만, inline 함수는 인자로 받은 함수를 다른 실행 컨텍스트를 통해 호출할 때는 함수 내부에서 non-local 흐름을 제어할 수 없습니다.

![](<../../../.gitbook/assets/Untitled (5).png>)

이럴 때 사용하는 것이 crossinline 키워드를 사용하면 됩니다.

#### noinline

아규먼트로 인라인 함수를 받을 수 없게 만듭니다.

인라인 함수가 아닌 함수를 아규먼트로 사용하고 싶을 때 활용합니다.

```kotlin
inline fun <reified T : Activity> Context.startActivity(noinline action: Intent.() -> Unit) {
    startActivity(Intent(this, T::class.java).apply(action))
}
```

Intent 객체인 action을 받아 이동하고자 하는 activity를 실행시켜야 합니다. 즉, 아규먼트로 action을 사용해야 하기 때문에 noinline 키워드를 붙여줍니다.
