# 개요

ROOM은 SQLite에 대한 추상화 레이어를 제공하여 원할한 데이터베이스 접근을 지원하는 동시에 SQLite와 연동성이 좋고 활용성이 높고, 많은 양의 구조화된 데이터를 처리하는 앱의 데이터를 로컬로 유지하는 장점을 가질 수 있다.

기기가 네트워크에 접근할 수 없을 때 오프라인 상태인 동안에도 사용자는 콘텐츠를 탐색할 수 있다. 나중에 기기가 온라인 상태가 되면 사용자가 시작한 콘텐츠 변경사항이 서버에 동기화된다.

앱에서 ROOM을 사용하려면 앱의 build.gradle 파일에 항목을 추가해줘야한다.

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

### ROOM의 구성요소

#### Entitiy

* 데이터베이스 내의 **테이블**을 나타낸다. 테이블명, 컬럼 명, 칼럼 타입, 기본 키, 외래 키 등을 정의할 수 있게 하는 다양한 어노테이션을 제공한다.

#### Dao

* 데이터베이스에 접근하는데 필요한 **메서드**를 포함한다. 데이터 삽입, 삭제, 업데이트, 쿼리등을 사용할 수 있게 하는 다양한 어노테이션을 제공한다.

#### Database

* 데이터베이스 홀더를 포함하며 **앱의 지속적인 관계형 데이터의 기본 연결**을 위한 **기본 액세스 포인트 역할**을 한다.
*   @Database 어노테이션이 지정된 클래스는 다음 조건을 충족해야 한다.

    * RoomDatabase를 확장하는 추상 클래스이어야 한다.
    * 어노테이션 내에 데이터베이스와 연결된 Entity의 목록을 포함해야 한다.
    * Arguments가 0개이고 @Dao 클래스를 반환하는 추상 메서드를 포함해야 한다.



![](<../../../../.gitbook/assets/Untitled (1) (2).png>)

앱은 ROOM 데이터베이스를 사용하여 데이터베이스와 연결된 Data Access Objects 또는 DAO를 가져온다. 후에 앱은 각 DAO를 사용하여 데이터베이스에서 Entity를 가져오고 Entity의 변경사항을 다시 데이터베이스에 저장한다. 마지막으로 앱은 Entity를 사용하여 데이터베이스 내의 테이블 열에 해당하는 값을 가져오고 설정한다.

**User(Entity)**

```kotlin
@Entity
data class User(
    @PrimaryKey val uid: Int,
    @ColumnInfo(name = "first_name") val firstName: String?,
    @ColumnInfo(name = "last_name") val lastName: String?
)
```

**UserDao(DAO)**

```kotlin
Dao
interface UserDao {
		**@Query("SELECT * FROM user")**
		fun getAll(): List<User>
	
		@Query("SELECT * FROM WHERE uid IN (:userIds)")
		fun loadAllByIds(userIds: IntArray): List<User>

		@Query("SELECT * FROM user WHERE first_name LIKE :first AND " +
		"last_name LIKE :last LIMIT 1")
		fun findByName(first: String, last: String): User

		@Insert
		fun insertAll(vararg users: User)

		@Delect
		fun delete(user: User)
}
```

**AppDatabase(Datebase)**

```kotlin
@Database(entities = arrayOf(User::class), version = 1)**
abstract class AppDatabase : RoomDatabase() {
		abstract fun userDao(): UserDao
}
```

Entitiy, DAO, Database를 생성한 후, 다음의 코드를 사용하여 생성한 데이터베이스 인스턴스를 가져온다.

```kotlin
val db = Room.databaseBuilder(
				applicationContext,
				AppDatabase::class.java, "database-name"
		).build()
```

***

### RFC

[https://developer.android.com/training/data-storage/room](https://developer.android.com/training/data-storage/room)
