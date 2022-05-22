# Entity

## Room Entitiy를 사용하여 데이터 정의

ROOM을 사용할 때 관련 필드 세트를 Entity로 정의한다. 각 Entity의 경우 연결된 Database 객체 내에 테이블을 만들어 Entity를 보유한다. Database 클래스의 **entities 배열을 통해 Entity 클래스를 참조**한다.

* **`@Database(entities = [Todo::class], version = 1)`**

```kotlin
@Entity
data class User(
		@PrimaryKey var id: Int,
		var firstName: String?,
		var lastName: String?
)
```

### Use a primary key

**각 Entity는 하나 이상의 필드를 기본 키**로 정의해야한다. 필드가 하나만 있는 경우에도 @PrimaryKey 어노테이션을 사용하여 필드에 명시해야한다. **PK 값은 아예 별개의 필드값으로 생성하고 autoGenerate = true**로 설정해주는게 바람직하다.

Entity에 복합 기본 키가 있다면 다음과 같이 @Entity의 primaryKeys 속성을 사용할 수 있다.

```kotlin
@Entity(primaryKeys = arrayOf("firstName", "lastName"))
data class User(
		val firstName: String?,
		val lastName: String?
)
```

* @PrimaryKey로 PK값을 지정할 수 있고 중복시 오류 발생
*   autoGenerate = true를 통하여 자동으로 Key 값 생성

    * true로 설정할 경우 유니크한 아이디 값을 자동으로 생성해준다.

    ```kotlin
    @Entity
    class UserEntity(
    		@PrimaryKey(autoGenerate = true) var userId: Int = 0
    )
    ```
*   @ColumnInfo로 컬럼 명 지정 가능

    ```kotlin
    @Entity(tableName = "users")
    data class User (
    		@PrimaryKey val id: Int,
    		@ColumnInfo(name = "first_name") val firstName: String?,
    		@ColumnInfo(name = "last_name") val lastName: String?

    )
    ```

기본적으로 Room은 클래스 이름을 데이터베이스 테이블 이름으로 사용한다. 테이블의 이름을 커스텀하고 싶다면 @Entity의 tableName 속성을 설정하면 된다.

```kotlin
@Entity(tableName = "users")
data class User(
		...
)
```

### Ignore fields

유지하고 싶지 않은 필드가 Entity에 있다면 @Ignore를 사용하여 필드에 추가하면 된다.

```kotlin
@Entity
data class User(
		@PrimaryKey val id: Int,
    val firstName: String?,
    val lastName: String?,
    **@Ignore val picture: Bitmap?**
)
```

Entity가 부모 Entity의 필드를 상속하는 경우, 일반적으로 **@Entity 속성의 ignoredColumns 속성**을 사용하는 것이 더 편리하다.

```kotlin
open class User {
		var picture: Bitmap? = null
}

**@Entity(ignoredColumns = arrayOf("picture"))**
data class RemoteUser(
		@PrimaryKey val id: Int,
		val hasVpn: Boolean
) : User()
```

### Provide table search support

#### full-text 검색

**FTS(전체 텍스트 검색)**를 통해 데이터베이스 정보에 매우 빠르게 접근해야 한다면, FTS3 또는 FTS4 SQLite 확장 모듈을 사용하는 **가상 테이블을 통해 Entity를 찾을 수 있다**.

* Room 2.1.0 이상에서 제공되는 해당 기능을 사용하려면 지정된 Entity에 @Fts3 또는 @Fts4를 추가해야 한다.

```kotlin
// Use `@Fts3` only if your app has strict disk space requirements or if you
// require compatibility with an older SQLite version.
@Fts4
@Entity(tableName = "users")
data class User(
    /* Specifying a primary key for an FTS-table-backed entity is optional, but
       if you include one, it must use this type and column name. */
    @PrimaryKey @ColumnInfo(name = "rowid") val id: Int,
    @ColumnInfo(name = "first_name") val firstName: String?
)
```

테이블이 여러 언어로 된 콘텐츠를 지원하는 경우, 다음과 같이 languageId 옵션을 사용하여 각 행의 언어 정보를 저장하는 열을 지정한다.

```kotlin
@Fts4(languageId = "lid")
@Entity(tableName = "users")
data class User(
		// ...
		@ColumnInfo(name = "lid") val languageId: Int
)
```

#### 특정 columns 색인(Index)

FTS3 또는 FTS4 테이블 지원 Entity를 사용할 수 없는 SDK 버전을 지원해야 할 때에도, 특정 열의 색인을 생성하여 쿼리 속도를 높일 수 있다. 즉, 데이터베이스 쿼리 속도를 높이기 위한 컬럼들을 인덱스로 지정할 수 있다.

Entity에 색인을 추가하려면 **@Entity 내에 indices 속성을 포함하여 색인 또는 복합 색인에 포함하려는 열의 이름을 나열**한다.

```kotlin
@Entity(indices = arrayOf(Index(value = ["lastName", "address"])))
data class User(
		@PrimaryKey val id: Int,
		val firstName: String?,
		val address: String?,
		@ColumnInfo(name = "lastName") val lastName: String?,
		@Ignore val picture: Bitmap?
)
```

데이터베이스의 특정 필드 또는 필드 그룹이 고유해야 할때, @Index 주석의 unique 속성을 true로 설정하여 unique 속성을 적용할 수 있다.

```kotlin
@Entity(indices = arrayOf(Index(value = ["lastName", "address"],
				unique = true)))
data class User(
    @PrimaryKey val id: Int,
    @ColumnInfo(name = "first_name") val firstName: String?,
    @ColumnInfo(name = "last_name") val lastName: String?,
    @Ignore var picture: Bitmap?
)
```

### Entity Annotataion

#### @Entity

| 메서드(인자)               | Tags                                                  |
| --------------------- | ----------------------------------------------------- |
| tableName()           | 데이터베이스내의 테이블 이름 설정. 설정하지 않을 경우 클래스 이름이 기본값으로 설정.      |
| indices()             | 데이터베이스 쿼리 속도를 높이기 위한 컬럼들을 인덱스로 지정할 수 있음.              |
| inheritSuperIndices() | true로 설정할 경우, 부모 클래스에 선언된 모든 인덱스가 현재의 Entity 클래스로 옮겨짐 |
| primaryKeys()         | 기본키로 지정하고 싶은 컬럼 값들을 한번에 설정할 수 있음.                     |
| foreignKeys()         | 외래키로 지정하고 싶은 컬럼 값들을 한번에 설정할 수 있음.                     |
| ignoredColumns()      | DB에 생성되기를 원하지 않는 컬럼을 한번에 설정할 수 있음.                    |



#### @PrimaryKey

| 메서드(인자)      | 설명                                 |
| ------------ | ---------------------------------- |
| autoGenerate | true로 설정할 경우 유니크한 아이디값을 자동으로 생성해줌. |



#### @ForignKey

| 메서드(인자)       | 설명                                                                                                        |
| ------------- | --------------------------------------------------------------------------------------------------------- |
| entity        | 참조할 부모 Entity를 의미한다. ParentEntity::class                                                                  |
| parentColumns | 참조할 부모의 key 값들                                                                                            |
| childColumns  | 참조할 부모의 key 값을 저장할 현재 Entity의 값들이며 parentColumns의 갯수와 동일해야 한다.                                            |
| onDelete      | 참조하고 있는 부모 Entity가 삭제될 때 이뤄지는 행위를 정의한다. (default: NO\_ACTION. RESTRICT. SET\_NULL. SET\_DEFAULT. CASCADE) |
| onUpdate      | 참조하고 있는 부모 Entity가 업데이트될 때 이뤄지는 행위를 정의한다.(onDelete와 같음)                                                   |
| deferred      | true로 설정할 경우 트랜잭션이 완료될 때까지 외래키의 제약 조건을 연기할 수 있음.                                                          |



#### @ForignKey의 onDelete, onUpdate 옵션

| 메서드(인자)             | 설명                                                         |
| ------------------- | ---------------------------------------------------------- |
| NO\_ACTION(DEFAULT) | 부모 Entity가 삭제되거나 변경되어도 아무런 행위를 하지 않음.                      |
| RESTRICT            | 참조하고 있는 부모의 key 값을 삭제하거나 변경할 수 없게 함.                       |
| SET\_NULL           | 참조하고 있는 key 값이 삭제되거나 변경되면 FK를 NULL로 초기화 시킴.                |
| SET\_DEFAULT        | SET\_NULL과 유사한 속성으로, 참조하고 있는 key 값이 삭제되거나 변경되면 기본값으로 변경한다. |
| CASCADE(삭제시)        | 부모 Entity가 삭제될 경우 자식 Entity를 삭제한다.                         |
| CASCADE(업데이트시)      | 부모 Entity가 업데이트 될 경우에는 자식 Entity의 FK값을 새로운 값으로 변경한다.       |



#### @ColumnInfo

| 메서드(인자)      | 설명                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------- |
| name         | 데이터베이스에서의 컬럼명을 지정할 수 있음. 기본값으로 Entity 내에 있는 필드명이 사용됨.                                       |
| typeAffinity | 컬럼의 타입을 지정할 수 있음. (default: UNDEFINED, TEXT, INTEGER, REAL, BLOB)                           |
| index        | 컬럼들을 인덱싱 할 수 있음.                                                                            |
| collate      | 컬럼값들의 정렬과 정렬 캐스트 작업을 정의함. (DEFAULT: UNSPECIFIED. BINARY. NOCASE. RTRIM. LOCALIZED. UNICODE) |



#### @ColumnInfo의 typeAffinity 타입

| 메서드(인자)            | 설명                                |
| ------------------ | --------------------------------- |
| UNDEFINED(DEFAULT) | 타입을 지정하지 않으며 입력되는 값에 따라 자동으로 저장됨. |
| TEXT               | 문자열로 저장함.                         |
| INTEGER            | int 값으로 저장함.                      |
| REAL               | Double 또는 Float 값으로 저장함.          |
| BLOB               | Binary 값으로 저장함.                   |



#### @ColumnInfo의 collate 옵션

| 메서드(인자)            | 설명                          |
| ------------------ | --------------------------- |
| UNDEFINED(DEFAULT) | 특별이 설정하지 않으나 BINARY 처럼 동작함. |
| BINARY             | 대소문자를 구분하여 정렬함.             |
| NOCASE             | 대소문자를 구분하지 않고 정렬함.          |
| RTRIM              | 앞뒤 공백을 제거하고 대소문자를 구분하여 정렬함. |
| LOCALIZED          | 시스템의 현재 지역을 기반으로 정렬함.       |
| UNICODE            | 유니코드 데이터 정렬 알고리즘을 이용하여 정렬함. |

@Ignore은 별도로 전달받는 인자값이 없으며 해당 Annotation을 설정할 경우 데이터베이스에 생성되지 않음.
