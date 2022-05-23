# DAO를 사용해 데이터 접근



Room에서는 SQL을 이용한 직접적인 쿼리 접근 대신에 **DAO(Data Access Object)를 이용하여 데이터베이스에 접근**해야 한다. DAO는 인터페이스 혹은 추상클래스로 구현되어야 한다.

각 DAO에는 앱 데이터베이스의 abstract access를 제공하는 메서드가 포함되어 있으므로, Dao 객체는 Room의 기본 구성요소를 형성한다.

### DAO 메서드 정의

#### Insert

DAO 메서드를 만든 후 @Insert을 추가할 때, Room은 **단일 트랜잭션으로 모든 매개변수를 데이터베이스에 insert**한다.

```kotlin
@Dao
interface MyDao {
		**@Insert(onConflict = OnConflictStrategy.REPLACE)**
		fun insertUsers(vararg users: User)

		@Insert
		fun insertBothUsers(user1: User, user2: User)

		@Insert
		fun insertUsersAndFriends(user: User, friends : List<User>)
}
```

Insert 어노테이션은 **여러 매개변수 타입을 가질 수 있게 한다**. 단일 객체부터 시작해서 배열 형태의 데이터까지 모두 입력이 가능하다. 또한 입력 할때 **옵션 값을 지정하여 충돌이 발생할 경우 어떻게 대처할지도 설정**이 가능하다.

#### Update

Update 메서드는 매개변수로 제공된 Entity를 데이터베이스에서 수정한다. 즉, **전달받은 매개변수의 기본 키 값에 매칭되는 Entitiy를 찾아 갱신**한다.

Update 메서드는 각 Entity의 기본 키와 일치하는 쿼리를 사용한다.

```kotlin
@Dao
interface MyDao {
		@Update
		fun updateUsers(vararg users: User)
}
```

#### Delete

Delete 메서드는 매개변수로 제공된 Entity를 데이터베이스에서 삭제한다. 기본키를 사용하여 삭제할 Entity를 찾는다. 즉, **전달받은 매개변수의 기본 키 값에 매칭되는 entity를 찾아 삭제**한다.

```kotlin
@Dao
interface MyDao{
		@Delete
		fun deleteUsers(vararg users: User)
}
```

### Query 메서드

주로 데이터를 선택할 때 사용한다(다른 구문은 어노테이션을 사용하는게 일반적). Query는 DAO 클래스 중 가장 핵심 부분이며 **데이터를 읽고 쓸 수 있게 해준다**. **컴파일시에 query 검사가 이루어지기 때문에 런타임 오류를 최소화** 할 수 있다.

#### Simple Query

```kotlin
@Dao
interface MyDao {
		@Query("SELECT * FROM user")
		fun loadAllUsers(): Array<User>
}
```

위의 예시 쿼리는 모든 사용자를 로드하는 단순한 쿼리이다. 컴파일할 때 Room은 사용자 테이블의 모든 열을 쿼리하고 있다는 것을 알고 있다. 쿼리에 구문 오류가 포함되어 있거나 데이터베이스에 사용자 테이블이 없으면 Room은 적절한 메시지와 함께 오류를 표시한다.

Simple Query는 List 형태의 리턴값을 받으며 변화를 감지할 수 없다.

#### Return a subset of a table's columns

Room을 사용할 때 Entity의 몇 가지 필드만 가져올 경우가 많다. **UI에 표시되는 컬럼만 가져오면 리소스가 절약되고 쿼리도 더 빨리 완료**될 수 있다.

* UI에는 사용자에 관한 모든 세부정보가 아닌, 사용자의 이름 및 성 만 표시될 때 Entity의 해당 필드만 가져온다.

Room을 사용하면 결과 컬럼을 반환된 객체에 매핑할 수 있는 쿼리에서 어떠한 자바 기반 객체라도 반환할 수 있다.

* POJO(Plain Old Java-based Object)를 생성하여 사용자의 이름 및 성을 가져올 수 있다.

```kotlin
data class NameTuple(
		@ColumnInfo(name = "frist_name") val firstName: String?,
		@ColumnInfo(name = "last_name") val lastName: String?
)
```

이제 쿼리 메서드에서, 위 POJO를 사용할 수 있다.

```kotlin
@Dao
interface MyDao {
		@Query("SELECT first_name, last_name FROM user")
		fun loadFullName(): List<NameTuple>
} 
```

Room은 쿼리에서 first\_name 및 last\_name 열의 값을 반환하고, 이러한 값은 NameTuple 클래스 필드에 매핑 될 수 있다는 것을 인식한다.

쿼리에서 너무 많은 컬럼을 반환하거나 NameTuple 클래스에 없는 열을 반환하면 Room은 경고를 표시한다.

#### 쿼리에 매개변수 전달

예를 들어, 특정 나이보다 나이가 더 많은 사용자만 표시하는 작업을 하려면 **매개변수를 쿼리에 전달**해야 한다.

```kotlin
@Dao
interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge")
    fun loadAllUsersOlderThan(minAge: Int): Array<User>
}
```

위 쿼리가 컴파일 시 처리되면 Room은 ' :minAge ' 바인드 매개변수를 minAge 메서드 매개변수와 일치시킨다. room은 매개변수 이름을 사용하여 일치시킨다. 불일치 시 컴파일될 때 오류가 발생한다.

여러 매개변수를 전달하거나 여러 번 참조할 수 도 있다.

```kotlin
@Dao
interface MyDao {
    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    fun loadAllUsersBetweenAges(minAge: Int, maxAge: Int): Array<User>

    @Query("SELECT * FROM user WHERE first_name LIKE :search " +
               "OR last_name LIKE :search")
    fun findUserWithName(search: String): List<User>
}
```

#### Pass a collection of parameters to a query

일부 쿼리에서는 런타임 때까지 알려지지 않은 정확한 수의 매개변수와 함께 가변적인 수의 매개변수를 전달할 수도 있다.

Room은 매개변수가 언제 컬렉션을 나타내는지 인식하고, 제공된 매개변수 수에 따라 런타임 시에 자동으로 확장한다.

```kotlin
@Dao
interface MyDao {
		@Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
		fun loadUsersFromRegions(regions: List<String>): List<NameTuple>
}
```

#### Query multiple tables

일부 쿼리에서는 결과를 계산하기 위해 여러 테이블에 접근해야 할 수 있다. Room을 사용하면 어떤 쿼리든 작성할 수 있으므로 테이블을 JOIN 할 수 있다. 또한 응답이 Flowable 또는 LiveData와 같은 식별 가능한 데이터 유형이면 Room은 쿼리에서 참조된 모든 테이블의 무효화를 Observing한다.

다음 코드는 테이블 JOIN을 실행하여 책을 빌리는 사용자가 포함된 테이블과 현재 대출 중인 책에 관한 데이터가 포함된 테이블 간에 정보를 통합하는 방법을 보여준다.

```kotlin
@Dao
interface MyDao {
    @Query(
        "SELECT * FROM book " +
        "INNER JOIN loan ON loan.book_id = book.id " +
        "INNER JOIN user ON user.id = loan.user_id " +
        "WHERE user.name LIKE :userName"
    )
    fun findBooksBorrowedByNameSync(userName: String): List<Book>
}
```

### Query return types

#### Flow를 사용하는 반응형 쿼리

Room 2.2 이상에서는 Kotlin의 Flow 기능을 사용하여 앱의 UI를 최신 상태로 유지 할 수 있다. 기본 데이터가 변경될 때 UI가 자동으로 업데이트 되도록 하려면 Flow 객체를 반환하는 쿼리 메서드를 작성하면 된다.

```kotlin
@Query("SELECT * FROM User")
		fun getAllUsers(): Flow<List<User>>
```

테이블의 데이터가 변경될 때마다 반환된 Flow 객체가 쿼리를 다시 트리거하고, 전체 결과를 다시 보낸다.

Flow를 사용하는 반응형 쿼리에는 중요한 제한사항이 존재한다. Flow 객체는 행이 결과셋에 있든 없든, 테이블의 행이 업데이트될 때마다 쿼리를 다시 실행한다. distinctUntilChanged() 연산자를 반환된 Flow 객체에 적용하여 실제 쿼리 결과가 변경될 때만 UI에 알림을 보낼 수 있다.

#### Paginated queries with the Paging library

Room은 페이징 라이브러리를 통해서 페이지화된 쿼리를 지원한다. Room 2.3.0-alpha01 또는 더 높은 버전에서, DAO는 페이징 3와 함께 PagingSource 객체를 반환할 수 있다.

```kotlin
@Dao
interface UserDao {
		@Query("SELECT * FROM users WHERE label LIKE :query")
		fun pagingSource(query: String): PagingSource<Int, User>
}
```

#### Direct cursor access

만약 앱 로직 중에서 반환하는 행에 직접 접근해야 하는 경우, 다음 예와 같이 Cursor 객체를 반환하는 DAO 방법을 write 할 수 있다.

```kotlin
@Dao
interface UserDao {
		@Query("SELECT * FROM user WHERE age > :minAge LIMIT 5")
		fun loadRawUsersOlderThan(minAge: Int):Cursor
}
```
