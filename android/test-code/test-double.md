# Test Double

영화를 촬영할 때 위험한 씬이나 액션 씬을 주연 배우가 아닌 스턴트 배우로 대체하는 것을 많이 볼 수 있다. 개발을 할 때도 마찬가지로, 실제 네트워킹 또는 데이터베이스를 사용하지 않고 Test Double을 사용해서 구현하는 것이 안전하다. 이로써 어떤 로직을 테스트할 때 리파짓토리 또는 데이터베이스에 영향을 받지 않을 것이다. 이렇게 테스트하려는 객체와 연관된 객체를 사용하기가 어렵고 모호할 때 대신해서 사용할 수 있게 해주는 객체를 Test Double이라 한다.

#### Fake

* 복잡한 로직이나 객체 내부에서 필요로 하는 다른 외부 객체들의 동작을 단순화하여 구현한 객체
* 동작의 구현을 가지고 있지만 실제 프로덕션에는 적합하지 않은 객체

즉, 동작은 하지만 실제 사용되는 객체처럼 정교하게 동작하지 않는 객체를 뜻함

#### Mock

호출에 대한 기대를 명세하고 내용에 따라 동작하도록 프로그래밍된 객체이다. 즉, 호출된 메서드를 추적해서 메서드가 올바르게 호출되었는지 여부에 따라 테스트를 통과하거나 통과하지 못하게 하는 Test Double이다.

Mockito 프레임워크가 대표적인 Mock 프레임워크이다.

#### Dummy

* 인스턴스화 된 객체가 필요하지만 기능을 필요하지 않는 경우에 사용
* Dummy 객체의 메서드가 호출되었을 때 정상 동작은 보장되지 않음
* 객체는 전달되지만 사용되지 않는 객체

즉, 인스턴스화된 객체가 필요해서 구현한 가짜 객체일 뿐이고, 생성된 Dummy 객체는 정상적인 동작을 보장하지 않는다.

```kotlin
interface TasksRepository {
	fun getTasks()
}
```

```kotlin
class DefaultTasksDummyRepository: TasksRepository {
	override fun getTasks() {
		//아무런 동작을 하지 않음
	}
}
```

실제 객체는 TasksRepository 인터페이스의 구현체를 필요로 하지만, 특정 테스트에서는 해당 구현체의 동작이 전혀 필요하지 않을 경우도 있다. 실제 객체가 Task를 저장만할 경우에는 테스트 환경에서 getTasks의 구현체가 전혀 필요로 하지 않기 때문이다.

이런 경우에는 getTasks()의 동작이 테스트에 영향을 받지 않기 때문에 Dummy 객체라 부른다.

#### Stub

* Dummy 객체가 실제로 동작하는 것처럼 보이게 만들어 놓은 객체
* 인터페이스 또는 기본 클래스가 최소한으로 구현된 상태
* 테스트에서 호출된 요청에 대해 미리 준비해둔 결과를 제공

즉, 테스트를 위해 프로그래밍 된 내용에 대해서만 준비된 결과를 제공하는 객체이다.

**Stub 기반 테스트 vs Mock 기반 테스트 (마틴 파울러의 글에서 인용)**

```java
public interface MailService {
	  public void send (Message msg);
}
public class MailServiceStub implements MailService {
	  private List<Message> messages = new ArrayList<Message>();
	  public void send (Message msg) {
		    messages.add(msg);
	  }
	  public int numberSent() {
		    return messages.size();
	  }
}
```

위의 코드에 대해 Stub 기반 테스트 코드를 작성해보자

```java
class OrderStateTester...

  public void testOrderSendsMailIfUnfilled() {
	    Order order = new Order(TALISKER, 51);
	    MailServiceStub mailer = new MailServiceStub();
	    order.setMailer(mailer);
	    order.fill(warehouse);
	    assertEquals(1, mailer.numberSent());
  }
```

다음은 Mock 기반 테스트 코드를 작성해보자

```java
class OrderInteractionTester...

  public void testOrderSendsMailIfUnfilled() {
	    Order order = new Order(TALISKER, 51);
	    Mock warehouse = mock(Warehouse.class);
	    Mock mailer = mock(MailService.class);
	    order.setMailer((MailService) mailer.proxy());
	
	    mailer.expects(once()).method("send");
	    warehouse.expects(once()).method("hasInventory")
	      .withAnyArguments()
	      .will(returnValue(false));
	
	    order.fill((Warehouse) warehouse.proxy());
  }
}
```

이를 비교해보면, Stub 기반 테스트 코드는 **상태 기반** 테스트이고, Mock 기반 테스트 코드는 **행위 기반** 테스트인 것을 확인할 수 있다.

만약, Mock을 사용했지만 Stub의 코드와 같이 작성했다면, 사실 Stub 기반 테스트 코드를 작성했던 것이라 할 수 있다.

#### Spy

* Stub의 역할을 가지면서 호출된 내용에 대해 약간의 정보를 기록함
* Test Double로 구현된 객체에 자기 자신이 호출 되었을 때 확인이 필요한 부분을 기록하도록 구현함
* 실제 객체처럼 동작 시킬 수도 있고, 필요한 부분에 대해서는 Stub 기반으로 만들어 동작을 지정할 수도 있음

즉, 실제 객체로도 사용할 수 있고 Stub 객체로도 활용할 수 있으며 필요한 경우 특정 메서드가 제대로 호출되었는지 여부를 확인할 수 있다.

```kotlin
fun MailService {
	val sendMailCount: Long = 0
	val mails: Collection<Mail> = arrayListOf()

	fun sendMail(mail: Mail) {
		sendMailCount++
		mails.add(mail)
	}

	fun getSendMailCount: Long {
		return sendMailCount
	}
}
```

위의 예시를 보면, MailService는 sendMail을 호출할 때마다 보낸 메일을 저장하고 몇 번 보냈는지를 체크한다. 메일을 보낸 횟수를 나타내는 getSendMailCount를 호출하면 sendMailCount 파라미터에 저장된 값을 반환한다. 이처럼 자기 자신이 호출된 상황을 확인할 수 있는 객체가 Spy이다.

***

### RFC

[Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
