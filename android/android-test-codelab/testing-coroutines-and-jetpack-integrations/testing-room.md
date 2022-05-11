# Testing Room

이번 챕터에서는 Room 데이터베이스를 어떻게 테스트하는지 배워봅니다. Room DAO(database access object)와 로컬 데이터소스 클래스를 테스트를 어떻게 작성하는지 배워봅시다.

### Add the architecture component testing library to gradle

1. instrumented 테스트를 위해 architecture component 테스팅 라이브러리를 추가합니다: `androidTestImplementation`&#x20;

**app/build.gradle**

```kotlin
androidTestImplementation "androidx.arch.core:core-testing:$archTestingVersion"
```

### Create the TasksDaoTest class

Room DAO는 실제로 Room이 애노테이션 프로세싱(ex: @GET, @Qeuery ..)을 통해 클래스로 변환하는 인터페이스입니다. 일반적으로 인터페이스에 대한 테스트 클래스를 생성하는 것은 의미가 없어 단축키가 없으므로 수동으로 생성해줍니다.

`TasksDaoTest` 클래스 생성

1. androidTest > data > source
2. local 패키지 생성
3. local 패키지 안에 TasksDaoTest.kt 생성

> _**데이터베이스 테스트를 어떤 source set에 넣어야 할까요?**_ 일반적으로 데이터베이스 테스트는 instrumented 테스트를 수행하며, 이는 이러한 테스트가 Android 테스트 source set에 포함됨을 의미합니다. 이러한 테스트를 로컬에서 실행하면 로컬 컴퓨터에 있는 모든 버전의 SQLite가 사용되기 때문에 Android 디바이스와 함께 제공되는 SQLite 버전과 다를 수 있습니다. 또한, Android 디바이스마다 SQLite 버전이 다르므로 이러한 테스트를 디바이스에서 instrumented 테스트로 실행할 있는 것도 유용합니다.

### Set up the TasksDaoTest class

1. `TasksDaoTest` 클래스에 다음과 같은 코드를 작성합니다.

**TasksDaoTest.kt**

```kotlin
@ExperimentalCoroutinesApi
@RunWith(AndroidJUnit4::class)
@SmallTest
class TasksDaoTest {

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

}
```

* `@ExperimentalCoroutinesApi` - runBlockingTest를 사용하므로 필요
* `@SmallTest` - 테스트를 “small run-time” 통합 테스트로 표시합니다(@MediumTest 통합 테스트, @LargeTest end-to end 테스트). 이와 같이 설정하면 테스트를 그룹화하고 실행할 크기를 선택할 수 있습니다. DAO 테스트는 DAO 테스트만 수행하므로 Unit 테스트로 간주되어 작은 테스트라고 할 수 있습니다.
* `RunWith(AndroidJUnit4::class` - AndroidX 테스트를 사용하므로 필요

DAO 인스턴스에 접근하려면 데이터베이스의 인스턴스를 만들어야합니다.

1. `TasksDaoTest` 에 `lateinit` 으로 database 필드를 생성합니다.

```kotlin
private lateinit var database: ToDoDatabase
```

1. 데이터베이스를 초기화하는 `@Before` 함수를 만듭니다.

테스트를 위해 데이터베이스를 초기화 할때 유의할 점

* `Room.inMemoryDatabaseBuilder` 를 사용하여 in-memory 데이터베이스를 생성합니다. 일반적인 데이터베이스는 유지(persist)되어야 합니다. 이에 비해, in-memory는 실제 디스크에 저장되지 않기 때문에 프로세스가 중지되면 완전히 삭제됩니다. 테스트를 할 때는 항상 in-memory 데이터베이스를 사용해야 합니다.
* AndroidX Test 라이브러리인 `ApplicationProvider.getApplicationContext()` \*\*\*\* 함수를 사용해서 application context를 얻습니다.

**TasksDaoTest.kt**

```kotlin
@Before
fun initDb() {
    // Using an in-memory database so that the information stored here disappears when the
    // process is killed.
    database = Room.inMemoryDatabaseBuilder(
        getApplicationContext(),
        ToDoDatabase::class.java
    ).build()
}
```

1. `@After` 함수 내에 `database.close()` 함수를 사용해 데이터베이스 사용이 끝나면 모두 정리한다.

```kotlin
@After
fun closeDb() = database.close()
```

**TasksDaoTest.kt**

```kotlin
@ExperimentalCoroutinesApi
@RunWith(AndroidJUnit4::class)
@SmallTest
class TasksDaoTest {

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    private lateinit var database: ToDoDatabase

    @Before
    fun initDb() {
        // Using an in-memory database so that the information stored here disappears when the
        // process is killed.
        database = Room.inMemoryDatabaseBuilder(
            getApplicationContext(),
            ToDoDatabase::class.java
        ).allowMainThreadQueries().build()
    }

    @After
    fun closeDb() = database.close()

}
```

### Write your first DAO test

첫 번째 DAO 테스트에서 작업을 insert한 후, id 별로 작업을 가져옵니다.

1. TasksDaoTest에 다음과 같이 작성합니다.

```kotlin
@Test
fun insertTaskAndGetById() = runBlockingTest {
    // GIVEN - Insert a task.
    val task = Task("title", "description")
    database.taskDao().insertTask(task)

    // WHEN - Get the task by id from the database.
    val loaded = database.taskDao().getTaskById(task.id)

    // THEN - The loaded data contains the expected values.
    assertThat<Task>(loaded as Task, notNullValue())
    assertThat(loaded.id, `is`(task.id))
    assertThat(loaded.title, `is`(task.title))
    assertThat(loaded.description, `is`(task.description))
    assertThat(loaded.isCompleted, `is`(task.isCompleted))
}
```

* 작업을 생성하고 데이터베이스에 insert 합니다.
* id를 사용하여 작업을 검색합니다.
* 작업을 검색했다면, 모든 프로퍼티가 insert된 작업과 일치하는지 asserts 합니다.

Notice:

* 이번 테스트에서 `runBlockingTest` 를 사용한 이유는 `insertTask` 와 `getTaskById` 가 suspend 함수이기 때문입니다.
* DAO는 데이터베이스 인스턴스를 접근하여 정상적으로 사용할 수 있습니다.

밑의 import를 확인하여 사용하면 됩니다.

**TasksDaoTest.kt**

```kotlin
import androidx.arch.core.executor.testing.InstantTaskExecutorRule
import androidx.room.Room
import androidx.test.core.app.ApplicationProvider.getApplicationContext
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.filters.SmallTest
import com.example.android.architecture.blueprints.todoapp.data.Task
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.runBlockingTest
import org.hamcrest.CoreMatchers.`is`
import org.hamcrest.CoreMatchers.notNullValue
import org.hamcrest.MatcherAssert.assertThat
import org.junit.After
import org.junit.Before
import org.junit.Rule
import org.junit.Test
import org.junit.runner.RunWith
```

1. 이제 테스트가 통과되는지 확인합니다.

### Try it yourself!

이제 DAO 테스트를 직접 작성해봅시다. 작업을 insert하고 업데이트한 후에 업데이트 값이 있는지 확인하는 테스트를 작성합시다.

**TasksDaoTest.kt**

```kotlin
@Test
fun updateTaskAndGetById() {
    // 1. Insert a task into the DAO.

    // 2. Update the task by creating a new task with the same ID but different attributes.
    
    // 3. Check that when you get the task by its ID, it has the updated values.
}
```

### Create an integration test for TasksLocalDataSource

지금까지는 TasksDao에 대한 Unit 테스트만 진행했습니다. 다음은 TasksLocalDataSource를 생성해서 Intergration 테스트를 진행해보겠습니다.

`TasksLocalDataSource` 는 DAO에서 반환된 정보를 가져와서 리포짓토리 클래스에서 예상하는 포멧으로 변환하는 클래스입니다(ex: 반환된 값을 성공 또는 에러 상태로 래핑). 실제 TasksLocalDatasource와 DAO 코드를 모두 테스트하기 때문에 Intergration 테스트로 작성해야 합니다.

`TasksLocalDataSourcTest` 는 DAO 테스트 방식과 매우 유사합니다.

1. TasksLocalDataSource 클래스 열기
2. Genertae → Test → TasksLocalDataSourceTest → androidTest source set

**TasksLocalDataSourceTest.kt**

```kotlin
@ExperimentalCoroutinesApi
@RunWith(AndroidJUnit4::class)
@MediumTest
class TasksLocalDataSourceTest {

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

}
```

TasksLocalDataSource와 DAO 테스트 코드의 실질적인 차이점은 Medium “Intergration” 테스트로 간주될 수 있다는 점입니다. 왜냐하면 TasksLocalDataSourceTest는 TasksLocalDataSource의 코드와 DAO 코드 모두Intergration 테스트를 작성하기 때문입니다.

1. `TasksLocalDataSourceTest` 에 `lateinit` 필드로 테스트에 필요한 두 개의 파라미터를 만듭니다 - `TasksLocalDataSource` , `database`

**TasksLocalDataSourceTest.kt**

```kotlin
private lateinit var localDataSource: TasksLocalDataSource
private lateinit var database: ToDoDatabase
```

1. @Before 함수를 만들어 database와 datasource를 초기화합니다.
2. DAO 테스트와 동일한 방식으로 database를 생성하고, `inMemoryDatabaseBuilder` 와 `ApplicationProvider.getApplicationContext()` 함수를 사용합니다.
3. `allowMainThreadQueries` 를 추가합니다. 보통 Room은 메인 스레드에서 데이터베이스 쿼리를 실행할 수 없습니다. `allowMainThreadQueries` 를 호출하면 이러한 검사를 하지 않습니다. production code에서는 이와 같이 작성하면 안됩니다.
4. database 및 Dispatcher.Main을 사용하여 `TasksLocalDataSource`를 인스턴스화 합니다. 이렇게 하면 메인 스레드에서 쿼리가 실행됩니다(`allowMainThreadQueries` 때문에 이와 같이 실행됨).

**TasksLocalDataSourceTest.kt**

```kotlin
@Before
fun setup() {
    // Using an in-memory database for testing, because it doesn't survive killing the process.
    database = Room.inMemoryDatabaseBuilder(
        ApplicationProvider.getApplicationContext(),
        ToDoDatabase::class.java
    )
        .allowMainThreadQueries()
        .build()

    localDataSource =
        TasksLocalDataSource(
            database.taskDao(),
            Dispatchers.Main
        )
}
```

1. @After 메서드를 사용해서 모든 작업이 완료된 후 `database.close` 호출하여 정리합니다.

**TasksLocalDataSourceTest.kt**

```kotlin
@ExperimentalCoroutinesApi
@RunWith(AndroidJUnit4::class)
@MediumTest
class TasksLocalDataSourceTest {

    private lateinit var localDataSource: TasksLocalDataSource
    private lateinit var database: ToDoDatabase

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    @Before
    fun setup() {
        // Using an in-memory database for testing, because it doesn't survive killing the process.
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            ToDoDatabase::class.java
        )
            .allowMainThreadQueries()
            .build()

        localDataSource =
            TasksLocalDataSource(
                database.taskDao(),
                Dispatchers.Main
            )
    }

    @After
    fun cleanUp() {
        database.close()
    }
    
}
```

### Write your first TasksLocalDataSourceTest

**TasksLocalDataSourceTest.kt**

```kotlin
import com.example.android.architecture.blueprints.todoapp.data.source.TasksDataSource
import com.example.android.architecture.blueprints.todoapp.data.Result.Success
import com.example.android.architecture.blueprints.todoapp.data.Task
import com.example.android.architecture.blueprints.todoapp.data.succeeded
import kotlinx.coroutines.runBlocking
import org.hamcrest.CoreMatchers.`is`
import org.junit.Assert.assertThat
import org.junit.Test

...

// runBlocking is used here because of <https://github.com/Kotlin/kotlinx.coroutines/issues/1204>
// TODO: Replace with runBlockingTest once issue is resolved
@Test
fun saveTask_retrievesTask() = runBlocking {
    // GIVEN - A new task saved in the database.
    val newTask = Task("title", "description", false)
    localDataSource.saveTask(newTask)

    // WHEN  - Task retrieved by ID.
    val result = localDataSource.getTask(newTask.id)

    // THEN - Same task is returned.
    assertThat(result.succeeded, `is`(true))
    result as Success
    assertThat(result.data.title, `is`("title"))
    assertThat(result.data.description, `is`("description"))
    assertThat(result.data.isCompleted, `is`(false))
}
```

DAO 테스트와 매우 유사하게 작성한 것을 확인할 수 있습니다.

* 작업을 생성한 후 데이터베이스에 insert 합니다.
* id를 사용해서 작업을 검색합니다.
* 작업을 검색되었다면 모든 프로퍼티를 insert된 작업과 일치함을 asserts 합니다.

DAO 테스트와 유일한 차이점은 localDataSource가 리포짓토리가 기대하는 포멧이 sealed `Result` 클래스의 인스턴스를 반환한다는 것입니다. 아래의 코드는 결과를 `Success` 로 캐스팅하는 것을 볼 수 있습니다.

```kotlin
assertThat(result.succeeded, `is`(true))
result as Success
```

### Write your own local data source test

위에서 배운 것을 토대로 직접 작성해보자

**TasksLocalDataSourceTest.kt**

```kotlin
@Test
fun completeTask_retrievedTaskIsComplete(){
    // 1. Save a new active task in the local data source.

    // 2. Mark it as complete.

    // 3. Check that the task can be retrieved from the local data source and is complete.

}
```

****
