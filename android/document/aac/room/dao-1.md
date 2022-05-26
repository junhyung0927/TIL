# 비동기 DAO 쿼리 작성

쿼리가 UI를 차단하는 것을 방지하기 위해, Room은 메인 스레드에 대한 데이터베이스 접근을 허용하지 않는다. 이러한 제한이 뜻하는 것은 비동기 DAO 쿼리를 만들어야 한다는 것이다. Room 라이브러리는 비동기 쿼리 실행을 제공하기 위해 여러 개의 다른 프레임워크와의 통합을 포함한다.

DAO 쿼리는 세 가지 카테고리로 나뉜다.

* 데이터베이스의 데이터를 insert, update 또는 delete 하는 One-shot 읽기 쿼리.
* 데이터베이스의 데이터를 한 번만 읽고, 그 때의 데이터베이스의 스냅샷과 함께 결과를 반환하는 One-shot 읽기 쿼리. ex) search
* 기존 데이터베이스 테이블이 변경될 때마다 데이터베이스에서 데이터를 읽고, 이러한 변경 사항을 반영하기 위해 새 값을 내보내는 Observable 가능한 읽기 쿼리. ex) 실시간으로 반응해야 하는 작업들

### Language and framework options

Room은 특정 언어 기능 및 라이브러리와의 상호 운용성을 위한 통합 지원을 제공한다.



| 쿼리 유형               | 코루틴         | Rxjava                                      | Guava                | Jetpack Lifecycle |
| ------------------- | ----------- | ------------------------------------------- | -------------------- | ----------------- |
| Observable read     | Flow\<T>    | Flowable\<T>, Publisher\<T>, Observable\<T> | N/A                  | LiveData\<T>      |
| 한번에?(one shot?) 읽기  | suspend fun | Maybe\<T>, Single\<T>                       | ListenableFuture\<T> | N/A               |
| 한번에? (one shot?) 쓰기 | suspend fun | Single\<T>, Maybe\<T>, Completable\<T>      | ListenableFuture\<T> | N/A               |



#### Kotlin with Flow and coroutines

코틀린은 타사 프레임워크 없이 비동기 쿼리를 쓸 수 있는 언어 기능을 제공한다.

* Room 2.2 이상 버전에서, 코틀린의 Flow 기능을 사용해서 Observable 가능한 쿼리를 작성할 수 있다.
* Room 2.1 이상 버전에서, 코틀린 코루틴 중 suspend 키워드를 사용하여 DAO 쿼리를 비동기 형태로 만들 수 있다.

```kotlin
dependencies {
  def room_version = "2.2.6"

  implementation "androidx.room:room-runtime:$room_version"
  kapt "androidx.room:room-compiler:$room_version"

  // optional - Kotlin Extensions and Coroutines support for Room
  implementation "androidx.room:room-ktx:$room_version"

  // optional - Test helpers
  testImplementation "androidx.room:room-testing:$room_version"
}
```

#### Java with RxJava

앱에서 Java 프로그래밍 언어를 사용하는 경우 RxJava 프레임워크의 특수 반환 타입을 사용하여 비동기 DAO 메서드를 쓸 수 있다.

* One-shot 쿼리의 경우, Room 2.1 이상 버전에서 Completable, Single\<T>, Maybe\<T> 반환 타입을 지원한다.
* Observable 쿼리의 경우, Room에서 Publisher\<T>, Flowable\<T>, Observable\<T> 반환 타입을 지원한다.

```kotlin
dependencies {
  def room_version = "2.2.6"

  implementation "androidx.room:room-runtime:$room_version"
  kapt "androidx.room:room-compiler:$room_version"

  // optional - Kotlin Extensions and Coroutines support for Room
  implementation "androidx.room:room-ktx:$room_version"

  // optional - Test helpers
  testImplementation "androidx.room:room-testing:$room_version"
}
```

#### Java with LiveData and Guava

앱에서 Java 프로그래밍 언어를 사용하지만 RxJava 프레임워크를 사용하지 않는 경우, 다음 방법을 통해 비동기 쿼리를 작성할 수 있다.

* Jetpack의 LiveData 래퍼 클래스를 사용하여 비동기 Observable 가능한 쿼리를 작성할 수 있다.
* 비동기 One-shot 쿼리를 쓰기 위한 Guava의 ListenableFuture\<T> 래퍼를 사용할 수 있다.

```kotlin
dependencies {
  def room_version = "2.2.6"

  implementation "androidx.room:room-runtime:$room_version"
  kapt "androidx.room:room-compiler:$room_version"

  // optional - Kotlin Extensions and Coroutines support for Room
  implementation "androidx.room:room-ktx:$room_version"

  // optional - Test helpers
  testImplementation "androidx.room:room-testing:$room_version"
}
```

### Write asynchronous one-shot queries

One-shot 쿼리는 오직 한 번만 실행되고 실행 시 데이터 스냅샷을 가져오는 데이터베이스 작업이다. 다음 코드는 비동기 One-shot 쿼리의 몇 가지 예다.

```kotlin
//코루틴
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(vararg users: User)

    @Update
    suspend fun updateUsers(vararg users: User)

    @Delete
    suspend fun deleteUsers(vararg users: User)

    @Query("SELECT * FROM user WHERE id = :id")
    suspend fun loadUserById(id: Int): User

    @Query("SELECT * from user WHERE region IN (:regions)")
    suspend fun loadUsersByRegion(regions: List<String>): List<User>
}
```

```kotlin
//RxJava
@Dao
public interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public Completable insertUsers(List<User> users);

    @Update
    public Completable updateUsers(List<User> users);

    @Delete
    public Completable deleteUsers(List<User> users);

    @Query("SELECT * FROM user WHERE id = :id")
    public Single<User> loadUserById(int id);

    @Query("SELECT * from user WHERE region IN (:regions)")
    public Single<List<User>> loadUsersByRegion(List<String> regions);
}
```

```kotlin
//Guava & Livedata
@Dao
public interface UserDao {
    // Returns the number of users inserted.
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public ListenableFuture<Integer> insertUsers(List<User> users);

    // Returns the number of users updated.
    @Update
    public ListenableFuture<Integer> updateUsers(List<User> users);

    // Returns the number of users deleted.
    @Delete
    public ListenableFuture<Integer> deleteUsers(List<User> users);

    @Query("SELECT * FROM user WHERE id = :id")
    public ListenableFuture<User> loadUserById(int id);

    @Query("SELECT * from user WHERE region IN (:regions)")
    public ListenableFuture<List<User>> loadUsersByRegion(List<String> regions);
}
```

🗨️ 스냅샷이란, 특정 시간에 데이터 저장 장치의 상태를 별도의 파일이나 이미지로 저장하는 기술로, 스냅샷 기능을 이용하여 데이터를 저장하면 유실된 데이터 복원과 일정 시점의 상태로 데이터를 복원할 수 있다.



### Write observable queries

Observable 쿼리는 쿼리에 의해 참조되는 테이블에 변경 사항이 있을 때마다 새로운 값을 내보내는 읽기 작업이다. 이 방법을 사용하면 기존 데이터베이스의 엔터티가 insert, update, delete 될 때, 표시된 엔터티 목록을 최신 상태로 유지할 수 있다. 다음 코드는 Observable 가능한 쿼리의 몇 가지 예다.

```kotlin
//코루틴
@Dao
interface UserDao {
    @Query("SELECT * FROM user WHERE id = :id")
    fun loadUserById(id: Int): Flow<User>

    @Query("SELECT * from user WHERE region IN (:regions)")
    fun loadUsersByRegion(regions: List<String>): Flow<List<User>>
}
```

```kotlin
//RxJava
@Dao
public interface UserDao {
    @Query("SELECT * FROM user WHERE id = :id")
    public Flowable<User> loadUserById(int id);

    @Query("SELECT * from user WHERE region IN (:regions)")
    public Flowable<List<User>> loadUsersByRegion(List<String> regions);
}
```

```kotlin
//Guava & Livedata
@Dao
public interface UserDao {
    @Query("SELECT * FROM user WHERE id = :id")
    public LiveData<User> loadUserById(int id);

    @Query("SELECT * from user WHERE region IN (:regions)")
    public LiveData<List<User>> loadUsersByRegion(List<String> regions);
}
```
