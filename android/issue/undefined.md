# 에러 핸들링

앱을 사용하다 보면 예기치 못하는 에러가 발생할 경우가 빈번합니다.

물론 이러한 모든 경우에 대한 핸들링이 되어있는 경우에는 괜찮지만, 그렇기에는 비용이 매우 많이 들어가고 어렵습니다.

이런 오류를 감지하고 처리하기 위해 Ted Park 님의 포스팅을 참고하여 오류 리포팅 서비스를 만들어봅시다.

제가 속한 크루에서 구현 중인 **제주로드 애플리케이션**에서는 다음과 같은 에러 페이지로 사용자에게 제공합니다.

![](<../../.gitbook/assets/스크린샷 2022-06-14 오후 8.01.38.png>)

HTTP 상태 코드와 같은 2xx, 5xx와 같은 에러는 데이터 계층에서 Interceptor에 Exception Class를 만들어 핸들링을 해주고 있습니다.

{% embed url="https://gist.github.com/junhyung0927/a850f55278a8eb1004f47434f70fd7f5" %}
\

{% endembed %}

유스케이스 계층에서 서버 에러(ex: 5xx)가 발생하면 미리 정의한 익셉션을 받아 뷰모델에서 ui state의 상태를 변경하여 ui에 알려줍니다. 이때 ui는 상태에 따른 에러 페이지를 사용자에게 제공하는 방식으로 구현되어 있습니다.

하지만 미리 정의한 익셉션이 아닌 예기치 못한 에러가 발생하면 앱은 바로 죽을 것입니다. 만약 실제 서비스를 배포중이였다면 사용자는 이유도 모른채 앱이 종료되어 ‘이게 뭐야?’라는 반응을 가지며 앱을 삭제할 수도 있을 것입니다. 즉, 고객 이탈률을 증가시키는 위험하고 치명적인 영향을 가져다 주겠죠.

### 글로벌하게 에러를 잡아서 처리 할 수는 없을까?

앱을 만들 때는 사용자 + 개발자에게 좋은 경험을 제공할 수 있도록 하는 것이 중요합니다. Ted Park 님의 포스팅에 따르면 아래와 같은 시나리오를 살펴볼 수 있습니다.

1. 오류 발생 시 즉각적인 오류 리포팅으로 개발자가 알 수 있도록 함
2. 사용자에게 문제가 생겼음을 알림
3. 이전에 열려있었던 화면을 다시 실행할 수 있도록 제공

그래서 제주로드 애플리케이션에서 위의 오른쪽 그림과 같은 화면을 보여주고 다시 이전 화면(액티비티)를 실행할 수 있도록 제공하도록 하고 있습니다.

만약 특정 로직에 의해서 예기치 못한 에러가 발생했다고 가정합시다. 그렇다면 에러 페이지에서 retry 버튼을 누를 때 이전 화면(액티비티)의 메서드를 기억하고 있다가 그 로직을 다시 실행해야 할 것입니다. 이러한 과정을 유연하게 처리하고자 **에러 페이지 → 이전 화면(액티비티) 다시 실행** 하여 특정 로직과 관계 없이 모든 로직(이전 화면에 한해서)refresh 하게 구현하였습니다.

자, 이제 에러 화면에 대한 구현 원리를 알아봅시다.

1. Global하게 처리하는 Handler 생성
   1. ActivityLifecycle Callback을 재정의하여 생명주기에 따라 액티비티 처리
2. 에러가 발생할 경우 Application Class에서 Global Handler catch
   1. lastactivity를 저장한 후 GlobalErrorActivity 실행
   2. retry 버튼 누르면 이전에 실행했었던 activity 정보를 가져와서 실행

#### Application class에서 Handler 설정

Global하게 에러를 처리하기 위해서 application class에 `setCrashHandler` 함수를 추가해줍니다.

이제 전체 Application 생먕주기를 따라가며 어디서든 에러가 나면 핸들러가 catch하여 처리할 것입니다.

{% embed url="https://gist.github.com/junhyung0927/448bba59502eefec1bd60c4c370242c1" %}

#### 이전 화면(액티비티) 정보 관리하기

Activity 생명주기를 abstract class로 만들어서 `GlobalErrorExceptionHandler` 클래스에서 필요한 생명 주기에 대한 처리를 구현합니다.

{% embed url="https://gist.github.com/junhyung0927/7f3b191b0c9249100e255964144a9b5d" %}

{% embed url="https://gist.github.com/junhyung0927/17423e0539dcffd77d483c53dce527e2" %}

**onActivityCreated**

액티비티가 생성되면 `lastActivity` 에 생성한 것을 저장합니다. 당연히 `GlobalErrorActivity` (에러 액티비티)라면 넘어가야겠죠?

**onActivityStarted**

액티비티가 `Stated` 상태 일 때, `activityCount` 를 증가시켜주고 `Created` 와 동일하게 `lastActivitiy` 에 현재 액티비티를 저장합니다.

**onActivityStopped**

다른 액티비티로 넘어갈 때, 이전 액티비티는 `Stopped` 상태가 됩니다. 이때 `activityCount` 를 감소시킵니다. 만약 `activityCount` 가 0 미만 일 때는, `lastActivity` 를 null로 설정합니다.

![](<../../.gitbook/assets/Untitled (10).png>)

예를 들어봅시다. 제주로드 앱에서는 splash view → main view → detail view 의 로직을 가지고 있습니다. 아래의 그림은 main view 에서 detail view로 진입할 때 인위적으로 RuntimeException을 던지게 해봤습니다.

splash view를 거쳐 갔으니 `activityCount`는 1일 것이고, main view는 `Started` 상태를 거쳐 카운트가 증가한 것을 볼 수 있습니다. 근데 detail view의 `Created` 상태일 때 인위적으로 에러를 던졌으니 main view는 `Stopped` 상태를 거쳐 카운트를 감소시킨 것을 확인할 수 있습니다. 그렇다면 `lastActivity` 는 main view를 가지고 있을 것입니다.

#### 에러 발생 처리

* 에러가 발생하면 GlobalErrorActivity를 실행
* 마지막으로 실행된 화면 정보도 Intent를 통해서 전달
* 개발/QA 모드 일때를 고려하여 오류 내용도 함께 넣어서 전달

{% embed url="https://gist.github.com/junhyung0927/65a66f479accb85e2b401e52e5c4ea69" %}

#### 에러 화면 실행

{% embed url="https://gist.github.com/junhyung0927/916f8f9b19bdea9315db81310e560081" %}

`GlobalErrorActivity` 클래스를 실행하면서, 이전 액티비티에 대한 정보와 에러 정보를 함께 담아 전달합니다.

Intent Flag를 설정하는 이유는 실행하는 에러 액티비티를 새로운 테스트로 설정하고, 실행 액티비티 외의 테스크를 모두 삭제하기 위해서 입니다.

#### 새로 고침하여 이전 화면으로 돌아가기

{% embed url="https://gist.github.com/junhyung0927/345f8a003c5360bb79af54cfdf0527d2" %}
\\
{% endembed %}

`GlobalErrorActivity` 의 새로 고침 버튼을 누르면 `lastActivityIntent` 의 정보를 참고하여 이전 화면으로 돌아갑니다.



### RFC

{% embed url="https://medium.com/prnd/%EC%95%84%EB%A6%84%EB%8B%B5%EA%B2%8C-%EC%95%B1-%EC%98%A4%EB%A5%98-%EC%B2%98%EB%A6%AC%ED%95%98%EA%B8%B0-8bf9a46df515" %}

