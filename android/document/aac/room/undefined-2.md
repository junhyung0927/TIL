# 데이터베이스에 뷰 만들기



Room은 쿼리 지원 클래스를 View로 참조하며, View는 DAO에서 사용될 때 데이터 객체와 동일하게 작동한다.

🗨️ Entity와 동일하게 View에 관해 SELECT문을 실행할 수 있으나, View에 관해 INSERT, UPDATE, DELETE문을 실행할 수 없다.

### 뷰 만들기

뷰를 만들려면 @DatabaseView 애노테이션을 클래스에 추가해야 한다. 애노테이션의 값을 클래스가 나타내야 하는 쿼리로 설정한다.

```kotlin
@DatabaseView(
**"SELECT user.id, user.name, user.department" +
"department.name AS departmentName FROM user" +
"INNER JOIN department ON user.departmentId = department.id"**)
data class UserDetail(
	val id: Long,
	val name: String?,
	val departmentId: Long,
	val departmentName: String?
)
```

### 데이터베이스와 뷰 연결

해당 View를 앱 데이터베이스의 일부로 포함시키려면 @Database 애노테이션에 views 속성을 포함해야 한다.

```kotlin
@Database(entities = arrayOf(User::class),
          views = arrayOf(UserDetail::class)**, version = 1)
abstract class AppDatabase: RoomDatabase() {
    abstract fun userDao(): UserDao
}
```
