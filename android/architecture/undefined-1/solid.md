# SOLID 원칙

![SOLID 원칙](<../../../.gitbook/assets/Untitled (10).png>)

### 대전제

SOLID는 기본적으로 중간 규모(모듈)에 대한 원칙임을 기억해야 한다.

여기서 말하는 중간 규모(모듈)은 클래스 또는 메서드(작은 규모)들이 모여있는 모듈 구조를 뜻한다.

#### 목표

중간 규모의 소프트웨어 구조가 아래와 같은 요구 사항을 만족하도록 구성해야 한다.

* 유지보수성 ⇒ 변경에 유연
* 가독성 ⇒ 이해하기 쉽게(누가 보더라도 모듈 배치, 클래스 이름 등을 보고 파악하기 쉽게)
*   낮은 결합도(Loose couple), 높은 응집도(strong cohesion) ⇒ 여러 소프트웨어 시스템에 사용될 수 있는 큰 컴포넌트 기반이 되어야 함

    * 하나의 모듈 내에서의 높은 응집도: 모듈 내에서 일어나는 구현, 책임들은 그 내부에서 해결되어야 한다.
    * 하나의 모듈 내에서의 낮은 결합도: 해당 클래스에서 사용되는 중요한 일을 처리하기 위해서, 외부(다른) 클래스로부터 도움을 받아서는 안된다.



### 단일 책임 원칙(Single Responsibility Principle)

* 응집도(cohesion)에 대한 기준
* “각 소프트웨어 모듈은 변경의 이유가 단 하나여야만 한다" == “하나의 모듈은 오직 하나의 **액터**에 대해서만 책임져야 한다”
  * 액터는 여러 유스케이스(클래스)들을 변화시킬 수 있는 주체를 뜻한다.
* **하나의 클래스가 하나의 일만 해야한다는 뜻이 아니다**. 하지만 메서드는 하나의 일만 해야 한다.
* 책임은 하나의 일을 뜻하는 것이 아니다.

#### 이 클래스는 왜 잘못된 설계일까?

![](<../../../.gitbook/assets/Untitled (11) (1).png>)

위의 Image 클래스 다이어그램을 보면 잘못 설계된 것을 확인할 수 있다. 이는 **세 개의 메서드가 서로 매우 다른 세 개의 액터를 책임지고 있기 때문이다**.

Image는 주어가 아닌 목적어에 불과하다. 예를 들어 ‘Image가 Screen을 그린다', ‘Image가 Server를 불러온다’는 잘못된 방식이다. 이를 ‘View에 Image를 그린다’, ‘Server에서 Image를 불러온다’로 봐야 올바른 방식이라 할 수 있다.

즉, Image의 책임은 “**액터가 가지고 있는 이미지 정보를 보여주고 제공한다**”에 대한 책임에 한정되어 있어야 한다.

하나의 예를 더 살펴보자.

![](<../../../.gitbook/assets/Untitled (12).png>)

만약 실제로는 또 다른 액터(로컬 캐시 저장소)가 더 있었다면 어땠을까?

이러한 경우에는, 서버 로딩 로직 수정을 위해서 saveToCache 메서드를 수정하면, 같은 메서드를 호출하는 cropByShape 메서드까지 영향을 받을 수 있다.

![](<../../../.gitbook/assets/Untitled (13) (1).png>)

위의 그림과 같이, 세 개의 클래스로 분리하면 해결된다. 하지만 사용자 입장에서는 클래스의 개수가 증가하게 되어 효율이 떨어진다고 생각할 수 있다. 이를 해결하기 위해 Facade 패턴을 사용하면 된다.

#### Facade 패턴

SRP을 적용하면 클래스의 수가 증가될 수 있어, 사용자 측면에서 불편함을 야기할 수 있다.

불편함을 제거하기 위해, 서브 시스템에 있는 인터페이스 집합에 대해서 하나의 통합된 인터페이스를 제공하는 것이 Facade 패턴이다. 즉, 서브 시스템을 좀 더 편리하게 만드는 상위 수준의 인터페이스를 정의하는 것이다.

![](<../../../.gitbook/assets/Untitled (14).png>)

ImageFacade 클래스는 메서드의 구현을 보여주지 않고, 어떤 메서드가 있는지만 보여주고 있다. 각 메서드의 구현은 세 개의 클래스에 위임한다.

이렇게 되면 ImageFacade 클래스는 어떠한 액터도 담당하지 않게 되고, 세 개의 클래스가 각각 하나의 액터만을 담당하도록 변경한다. 즉, **변경사항이 생겨도 세 개의 클래스에서만 변경하면 되므로 ImageFacade 클래스에는 전혀 영향을 미치지 않게 된다**.

### 개방-폐쇄 원칙(Open-Closed Principle)

* “소프트웨어 모듈은 **확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다**”
* **“열려 있다”** : **데이터 구조에 필드를 추가하거나, 함수에 새로운 요소를 추가하는 것이 가능**해야 한다. 예를 들어 멤버 변수가 늘어난다고 해서 사용하는 클래스에서 변경이 일어나서는 안된다.
* **“닫혀 있다"** : **내부 코드를 변경해도 이를 사용하는 외부 모듈에서는 영향을 미치지 않아야 한다**. 예를 들어, fileOpen 메서드의 내부 구현이 변경되어도 사용하는 입장(외부 모듈)에서는 그대로 사용하면 된다.
* 클래스 사이에서도 개방 폐쇄 원칙은 중요하다. 하나의 모듈 안에서 여러 개의 클래스가 상호작용 하면서 의존관계가 성립될 수 있기 때문이다.
  * MVP 아키텍처에서 Presenter는 상위 계층이기 때문에 View에 의존하지 않지만, View는 Presenter에 의존한다. 즉, 하위 계층은 상위 계층을 의존할 수 있지만 반대로 성립되서는 안된다.

#### 클래스 단위의 개방-폐쇄 원칙

**왜 모든 멤버변수는 private 혹은 protected 여야 하는가?**

* 예를 들어, 앞서 얘기했던 이미지 클래스가 이미지 데이터도 퍼블릭으로 노출하고 있다면?
* 내부 데이터의 변경이 클래스 사용자에게 의도하지 않게 영향을 미칠 수 있다.

**왜 글로벌 변수는 피해야 하는가?**

* 이를 참조하는 모듈이나 클래스가 외부 값에 의존하게 되면, 예기치 못한 변경이 생길 수도 있기 때문에 폐쇄 원칙이 깨진다.

**왜 추상 클래스(abstract class / interface)를 굳이 따로 만드는 경우가 많은가?**

* 사용자에게 필요한 것만 노출함으로써 폐쇄 원칙을 만족하고, 구현 클래스의 내용을 감춤으로 수정도 용이해진다.

#### 모듈 단위의 개방-폐쇄 원칙

의존 관계의 방향이, 보호하려는 컴포넌트를 향하도록 그려진다. 결과적으로 **높은 수준의 클래스/모듈이 하위 레벨의 변경으로부터 보호**된다.

예를 들어, 서버로부터 매출 정보를 받아와서 분석 보고서를 화면에 그려주는 앱을 생각해보자.

![](<../../../.gitbook/assets/Untitled (15).png>)

만약, ReportRepository Interface가 없다면 ReportGenerator는 Repository와 DataSource를 의존하게 되게 될 것이다. 부모로부터 손자까지 연쇄되어 의존을 하게 되는 형태를 가지게 될 것이다.

또한, RepositoryImpl이 Repository를 통과해서 UseCase까지 의존되는 방향이 위까지 이어질 것이다. 이러한 의존성(방향)을 바꾸기 위해서 Interface를 만들어 주는 것이다.



### 리스코프 치환 원칙(Liskov Substitution Principle)

* **대체 가능한 컴포넌트들을 이용해 시스템을 만들 수 있으려면, 이들의 서브타입들은 반드시 서로 치환 가능해야 한다.**
* 부모 객체와 이를 상속한 자식 객체가 있을 때, 부모 객체를 호출하는 동작에서 자식 객체가 부모 객체를 완전히 대체할 수 있어야 한다는 원칙이다.
* 자식 객체는 부모 객체의 특성을 가지며, 이를 토대로 확장할 수 있다. 하지만 이 과정에서 객체의 의의와 어긋나는 확장으로 인해 잘못된 방향으로 상속되는 경우가 많다.
* 리스코프 치환 원칙은 올바른 상속을 위해 자식 객체의 확장이 부모 객체의 방향을 온전히 따르도록 권고하는 원칙이다.

#### 리스코프 치환 원칙의 좋은 예

![](<../../../.gitbook/assets/Untitled (16).png>)

Repository의 행위가 DataSource의 자식들 중 어떤 것을 사용해도 동일한 역할을 한다.

⇒ **부모에서 정한 행위의 원칙을 자식에서 그대로 지키고 있기 때문에** 행위 상속이라고도 부른다.

#### 리스코프 치환 원칙의 나쁜 예

![](<../../../.gitbook/assets/Untitled (17).png>)

언뜻 보기에는 잘못된 것이 없다고 생각할 수 있다. 아래의 코드를 살펴보자.

```kotlin
open class Rectangle {
    open var width = 0
    open var height = 0

    val area: Int
        get() = width * height
}

class Square : Rectangle() {
    override var width: Int
        get() = super.width
        set(value) {
            super.width = value
            super.height = value
        }

    override var height: Int
        get() = super.height
        set(value) {
            super.width = value
            super.height = value
        }
}

fun main() {
    val rectangle: Rectangle = Square()
    rectangle.width = 3
    rectangle.height = 4
    check(rectangle.area == 12)
}

Exception in thread "main" java.lang.IllegalStateException: Check failed.
	at Char4.RectangleKt.main(Rectangle.kt:31)
	at Char4.RectangleKt.main(Rectangle.kt)
```

Rectangle 클래스는 직사각형을 구현한 객체이다. width, height를 지정, 반환할 수 있으며 area 메서드를 사용해서 넓이를 계산할 수 있다.

직사각형의 범주를 넓게 생각해보면 정사각형도 하나의 종류이니, **직사각형을 상속해서 정사각형 객체를 빠르게 만들 수 있을거라 판단할 수 있다.**

리스코프 치환 원칙에 의하면, 자식 객체는 부모 객체에 완전히 대체할 수 있다고 했으므로 rectangle.area는 12가 반환되어야 한다. 하지만 결과는 16이 나올 것이다. Square 클래스의 height의 set feild에서 width/height 모두 4로 할당했기 때문이다. 즉, 이 객체는 리스코프 치환 원칙에 위배되는 코드이다.

그렇다면 이 코드를 어떻게 리스코프 치환 원칙에 위배되지 않게 구성할 수 있을까?

답은 **올바른 상속과 구현**에 있다. 직사각형과 정사각형은 상속의 관계가 성립되기 어렵다. 따라서 더 상위 개념인 사각형 객체를 구현하고 정사각형, 직사각형이 이를 상속받게 수정해보자.

```kotlin
interface Shape {
    val width: Int
    val height: Int

    val area: Int
        get() = width * height
}

class Rectangle(override val width: Int, override val height: Int) : Shape

class Square(private val length: Int) : Shape {
    override val width: Int
        get() = length
    override val height: Int
        get() = length
}

fun main() {
    val rectangle: Shape = Rectangle(width = 3, height = 5)
    val square: Shape = Square(5)
    
    println("${rectangle.area}, ${square.area}")
}

//15, 25
```

더 이상 Rectangle과 Square는 상속 관계가 아니므로, 리스코프 치환 원칙의 영향에서 벗어났다.

#### 왜 상속보다 조합(composition over inheritance)인가?

이는 상속으로 인한 부작용 때문이다. 자식은 부모가 갖고 있는 행위를 정확히 이해하고, 이 행위가 깨지지 않도록 할 의무가 있다.

상속을 사용한 코드에 대해서 예기치 못한 확장이 일어났을 때, 자식 클래스는 오버라이드한 메서드를 수정해야 한다. 즉, **부모 클래스의 내부 구조를 잘 알고 있어야 되고**, **내부 구현은 자식 클래스에게 노출되어 캡슐화가 약해지고** **결합도가 증가**하게 된다.

이에 반면, 합성을 사용해 멤버 변수로 선언하게 되면 확장이 일어나더라도 **의존하는 객체를 교체만 하면 되기 때문에 결합도가 낮아진다**. 또한, **사용자는 메세지를 통해 요청하기 때문에 객체의 내부 구성을 알 필요가 없다**.

### 인터페이스 분리 원칙(Interface Segregation Principle)

**각 소프트웨어 모듈은 자신이 사용하지 않는 것에 의존하지 않아야 한다.**

특정 Operation을 제공하는 OPS 클래스가 있다고 가정해보자. 이것을 사용하는 세 명의 User가 있는데, 만약 하나의 클래스만 사용하고 있다면 인터페이스 분리 원칙에 위반되는 행동이다.

![](<../../../.gitbook/assets/Untitled (18).png>)

인터페이스 분리 원칙을 지키기 위해, User가 실제로 사용하는 로직을 담당하는 각각의 Usecase Interface를 생성하고, 이를 구현하는 클래스들이 있게 구성하면 된다.

#### 모듈 단위의 인터페이스 분리 원칙

**외부 모듈에 필요한 “인터페이스”만 노출한다.**

![](<../../../.gitbook/assets/Untitled (19).png>)

UseCase와 DataSource 사이에 왜 ReportRepository 인터페이스가 필요할까? 이는 transitive dependency(추이 종속성)을 막기 위해서다. 추이 종속성을 가지고 되면 “**자신이 직접 사용하지 않는 요소에도 절대로 의존해서는 안된다**”는 소프트웨어 원칙을 위반하게 된다.

UseCase → Repository → DataSource 처럼 종속성을 가지게 되면 불필요한 의존성이 생기게 된다. 이를 방지하기 위해 Interface를 만들어 해결한다.

### 의존성 역전 원칙(Dependency Inversion Principle)

**높은 수준의 코드는 낮은 수준의 세부사항 구현에 절대로 의존해서는 안 되며, 그 반대여야 한다.**

![](<../../../.gitbook/assets/Untitled (20).png>)

오른쪽 다이어그램을 보면, Client(높은 수준)는 Service(낮은 수준)를 의존하고 있다. 하지만 Service는 변화하기 쉬운 클래스이므로 변화가 생길때 마다 이를 의존하고 있는 Client도 변화하는 문제점을 가지고 있습니다.

이를 해결하기 위해, Service 인터페이스(변하지 않는 것)를 두고 구현체 클래스인 ServiceImpl(변하기 쉬운 것)를 만들어 줍니다. Client는 더 이상 구현체 클래스를 의존하지 않고 있기 때문에 변화에 유연하게 대체할 수 있게 됩니다.

#### 추상 팩토리 패턴을 추가하면 매우 유용

![](<../../../.gitbook/assets/Untitled (21) (1).png>)

Client는 ServiceImpl을 사용하려면 당연히 Service를 생성해줘야 합니다. 이때 추상 팩토리 패턴을 사용하면 유용하게 처리할 수 있습니다.

추상 팩토리 패턴이란 구체적인 클래스에 의존하지 않고 서로 연관되거나 의존적인 객체들의 조합을 만드는 인터페이스를 제공하는 패턴입니다. 즉, **관련성이 있는 여러 종류의 객체를 일관된 방식으로 생성하는 경우에 유용**합니다.

Service Factory 인터페이스를 생성하고 구현 클래스를 만들게 되면, ServiceFactoryImpl 클래스가 ServiceImpl의 생성을 담당하므로 Client는 생성과 사용을 두 개의 인터페이스만 알고 있으면 처리가 가능합니다. 즉, **클라이언트의 입장에서 클라이언트가 추상화된 인터페이스를 통해 객체를 생성**할 수 있도록 해줍니다.



### RFC

_**강사룡의 앱 안정성 및 확장성 강화를 위한 Android 아키텍처**_

[\[OOP\] 객체지향 5원칙(SOLID) - 리스코프 치환 원칙 (Liskov Subsitution Principle) - 𝝅번째 알파카의 개발 낙서장](https://blog.itcode.dev/posts/2021/08/15/liskov-subsitution-principle)
