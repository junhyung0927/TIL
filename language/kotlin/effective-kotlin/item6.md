---
description: 사용자 정의 오류보다는 표준 오류를 사용하라
---

# item6

item5에서 살펴본 require, check, assert 함수를 사용하면, 대부분의 코틀린 오류를 처리할 수 있지만, 이외에도 예측하지 못한 상황을 나타내야 하는 경우가 있습니다.

JSON 형식을 파싱하는 라이브러리를 구현한다고 가정합시다. 기본적으로 입력된 JSON 파일의 형식에 문제가 있다면, JsonParsingException 등을 발생시키는 것이 좋습니다.

```kotlin
inline fun <reified T> String.readObject(): T {
	//...
	if (incorrectSign) {
		throw JsonParsingException()
	}
	//...
	return result
}
```

표준 라이브러리에는 이를 나타내는 적절한 오류가 없으므로, 사용자 정의 오류를 사용했습니다. 하지만 직접 오류를 정의하는 것보다 최대한 표준 라이브러리의 오류를 사용하는 것이 좋습니다. 표준 라이브러리의 오류는 많은 개발자가 인지하고 있으므로, 이를 재사용하는 것을 지향해야 합니다.

일반적으로 사용되는 예외를 몇 가지 살펴봅시다.

* **IllegalArgumentException & IllegalStateException**: require와 check를 사용해 throw 할 수 있는 예외
* **IndexOutOfBoundsException**: 인덱스 파라미터의 값이 범위를 벗어났다는 것을 나타냄. 일반적으로 컬렉션 또는 배열과 함께 사용됨
* **ConcurrentModificationException**: 동시 수정(concurrent modification)을 금지했는데, 이를 발생했다는 것을 나타냄
* **UnsupportedOperationException**: 사용자가 사용하려고 했던 메서드가 현재 객체에서는 사용할 수 없다는 것을 나타냄. 기본적으로 사용할 수 없는 메서드는 클래스에 없는 것이 좋습니다.
* **NoSuchElementException**: 사용자가 사용하려고 했던 요소가 존재하지 않음을 나타냄.
