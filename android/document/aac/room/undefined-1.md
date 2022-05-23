# 객체 간 관계 정의



SQLite는 관계형 데이터베이스이므로 엔터티 간 관계를 지정할 수 있다. 대부분의 객체 관계 매핑 라이브러리에서는 엔터티 객체가 서로 참조될 수 있으나, ROOM에서는 이를 명시적으로 금지한다. 금지 이유에 대해서 더 알고 싶다면 [Understand why Room doesn't allow object references](https://developer.android.com/training/data-storage/room/referencing-data#understand-no-object-references)를 참고해라.

### Create embedded objects

때로 개발자는 객체에 여러 필드가 포함되어 있는 경우에도, 데이터베이스 로직에서 엔터티 또는 데이터 객체를 전체적으로 응집되게 표현하려 한다. 이런 경우, @Embedded 어노테이션을 사용하여 테이블 내의 하위 필드로 분해 할 객체를 나타낼 수 있다. 그런 다음, 다른 개별적인 컬럼을 쿼리하듯 삽입된 필드를 쿼리할 수 있다.

예를 들어, User 클래스에는 Address 유형의 필드를 포함할 수 있다(street, city, state, postCode). 구성된 컬럼을 테이블에 별도로 저장하려면 다음 코드와 같이 @Embedded 어노테이션을 사용하여 Address 필드를 User 클래스에 포함한다.

```kotlin
data class Address(
        val street: String?,
        val state: String?,
        val city: String?,
        @ColumnInfo(name = "post_code") val postCode: Int
    )

@Entity
data class User(
        @PrimaryKey val id: Int,
        val firstName: String?,
        @Embedded val address: Address?
)
```

User 객체를 나타내는 테이블에는 id, firstName, street, state, city, post\_code의 컬럼을 포함하고 있게 된다.

\<aside> 🗨️ 삽입된 필드는 삽입된 다른 필드를 포함할 수 있다.

\</aside>

엔터티에 동일한 유형의 삽입된 필드가 여러 개 있으면, prefix 속성을 설정하여 각 컬럼을 고유하게 유지할 수 있다. Room은 제공된 값을 삽입된 객체의 각 컬럼 이름 시작 부분에 추가한다.

### Define one-to-one relationships

두 엔터티 간 1:1 관계는 상위(부모) 엔터티의 각 인스턴스가 정확히 하나의 하위(자식) 엔터티 인스턴스에 상응하는 관계이며, 그 반대의 경우에도 동일하다.

예를 들어 사용자가 소유한 노래 라이브러리가 있는 음악 스트리밍 앱을 생각해보자. 각 사용자에게는 하나의 라이브러리만 있으며 각 라이브러리는 정확히 한 명의 사용자에 상응한다. 따라서 User 엔터티와 Library 엔터티 간 1:1 관계가 있어야 한다.

먼저, 각각의 엔터티 클래스를 만든다. 엔터티 중 하나는 다른 엔터티의 기본 키를 참조하는 변수를 포함해야 한다.

```kotlin
@Entity
data class User(
    **@PrimaryKey val userId: Long,**
    val name: String,
    val age: Int
)

@Entity
data class Library(
    **@PrimaryKey val libraryId: Long,**
    val userOwnerId: Long
)
```

사용자와 상응하는 라이브러리 목록을 쿼리하려면 먼저, 두 엔터티 간 1:1 관계를 모델링해야 한다.

이러한 모델링을 만들려면 각 인스턴스가 상위 엔터티 인스턴스 및 상응하는 하위 엔터티 인스턴스를 보유하는 새 데이터 클래스를 만들어야 한다.

_parentColumn_을 상위 엔터티의 기본 키 컬럼 이름으로 설정하고, _entityColumn_을 상위 엔터티의 기본 키를 참조하는 하위 엔터티의 컬럼 이름으로 설정하여 @Relation 어노테이션을 하위 엔터티 인스턴스에 추가한다.

```kotlin
data class UserAndLibrary(
    @Embedded val user: User,
    @Relation
         parentColumn = "userId",
         entityColumn = "userOwnerId"
    )
    val library: Library
)
```

마지막으로, 상위 엔터티와 하위 엔터티를 쌍으로 하는 데이터 클래스의 모든 인스턴스를 반환하는 메서드를 DAO 클래스에 추가한다. 이 메서드를 사용하려면 Room에서 두 개의 쿼리를 실행해야 하므로, 해당 메서드에 @Transaction 어노테이션을 추가하여 전체 작업이 원자적으로 실행되도록 해야 한다.

```kotlin
@Transaction
@Query("SELECT * FROM User")
fun getUsersAndLibraries(): List<UserAndLibrary>
```

\<aside> 🗨️ atom(원자)라는 단어는 원래 '쪼갤 수 없는 것'이라는 의미를 가진다. 전산에서의 atomicity는 어떤 동작의 묶음이 모두 일어나거나 혹은 어느 것도 일어나지 않도록 보장해주는 것을 말한다. 즉 '쪼개어져서는 안되는, 따로 따로 놀아서는 안되는 동작들'을 묶어주는 DBMS의 기본 기능이다. 예를 들어 계좌 송금을 들 수 있다. 한 쪽 계좌에서 돈이 빠져나가면 다른 쪽 계좌에는 돈이 들어가야 한다. 만약 빠지기만 하고 들어가지 않으면 돈이 증발하고, 빠지지는 않고 들어가기만 하면 없던 돈이 생기는 현상이 발생하면 안되기 때문이다.

\</aside>

### Define one-to-many relationships

두 엔터티 간 1 : 다 관계는 상위 엔터티의 각 인스턴스가 0개 이상의 하위 엔터티 인스턴스에 상응하지만, 하위 엔터티의 각 인스턴스는 정확히 하나의 상위 엔터티 인스턴스에만 상응할 수 있는 관계이다.

전에 예를 들었던 음악 스트리밍 앱에서 사용자가 노래를 재생목록으로 구성할 수 있다고 가정해보자. 각 사용자는 원하는만큼 재생목록을 만들 수 있지만, 각 재생목록은 정확히 한 명의 사용자가 만든다. 따라서 User 엔터티와 Playlist 엔터티 간에 1 : 다 관계가 존재해야 한다.

먼저, 각각의 엔터티 클래스를 만든다. 이전 예에서와 같이 하위 엔터티는 상위 엔터티의 기본 키에 대한 참조 변수를 포함해야 한다.

```kotlin
@Entity
data class User(
		@PrimaryKey val userId: Long,
		val name: String,
		val age: Int
)

@Entity
data class Playlists(
		@PrimaryKey val platlistId: Long,
		val userCreatorId: String,
		val playlistName: Int
)
```

사용자 목록에 상응하는 재생목록을 쿼리하려면 두 엔터티 간 1 : 다 관계를 모델링해야 한다.

이러한 모델링을 만들려면 각 인스턴스가 상위 엔터티 인스턴스 및 상응하는 모든 하위 엔터티 인스턴스 목록을 보유하는 새 데이터 클래스를 만들어야 한다.

_parentColumn_을 상위 엔터티의 기본 키 컬럼 이름으로 설정하고 _entityColumn_을 상위 엔터티의 기본 키를 참조하는 하위 엔터티의 컬럼 이름으로 설정하여 @Relation 어노테이션을 하위 엔터티 인스턴스에 추가해야 한다.

```kotlin
data class UserWithPlaylists(
		@Embedded val user: User,
		@Relation(
				parentColumn = "userId",
				entityColumn = "userCreatorId"**
		)
		val playlists: List<PlayList>
)
```

마지막으로, 상위 엔터티와 하위 엔터티를 쌍으로 하는 데이터 클래스의 모든 인스턴스를 반환하는 메서드를 DAO 클래스에 추가한다. 이 메서드를 사용하려면 Room에서 두 개의 쿼리를 실행해야 하므로, 해당 메서드에 @Transaction 어노테이션을 추가하여 전체 작업이 원자적으로 실행되도록 해야 한다.

```kotlin
@Transaction
@Query("SELECT * FROM User")
fun getUserWithPlaylists(): List<UserWithPlaylists>
```

### Define many-to-many relationships

두 엔터티 간의 다 : 다 관계는 상위 엔터티의 각 인스턴스가 0개 이상의 하위 엔터티 인스턴스에 상응하는 관계이며, 그 반대의 경우도 동일하다.

이전의 예인 음악 스트리밍 앱의 사용자 정의 재생목록을 생각해보자. 각 재생목록에는 많은 노래가 포함될 수 있어야 하며, 각 노래는 여러 다양한 재생목록에 속할 수 있다. 따라서 Playlist 엔터티와 Song 엔터티 간에 다 : 다 관계가 존재해야 한다.

먼저, 각각의 엔터티 클래스를 만든다. 다 : 다 관계는 **일반적으로 하위 엔터티에 상위 엔터티에 대한 참조가 없기 때문에 다른 관계 유형과 구별된다.** 대신 하나의 클래스를 더 만들어 두 엔터티 간의 \*\*\*연결 엔터티(또는 상호 참조 테이블)\*\*\*를 나타낸다.

상호 참조 테이블에는 테이블에 표시된 다 : 다 관계에 있는 각 엔터티의 기본 키 컬럼이 있어야 한다. 해당 예에서는 상호 참조 테이블의 각 행은 _Playlist_ 인스턴스와 _Song_ 인스턴스의 쌍이며, 여기서 참조된 Song은 참조된 Playlist에 포함된다.

```kotlin
@Entity
data class Playlist(
		@PrimaryKey val playlistId: Long,
		val playlistName: String
)

@Entity
data class Song(
		@PrimaryKey val songId: Long,
		val songName: String,
		val artist: String
)

**@Entity(primaryKeys = ["playlistId", "songId"])**
**data class PlaylistSongCrossRef(
		val playlistId: Long,
		val songId: Long
)**
```

다음 단계는 이러한 관련 엔터티를 쿼리하는 방법에 따라 나눠진다.

* **Playlist** 및 **각 Playlist에 상응하는 Song 목록을 쿼리**하려면, 단일 Playlist 객체 및 Playlist에 포함된 모든 Song 객체 목록을 포함하는 새 데이터 클래스를 만든다.
* **Song** 및 **각 Song에 상응하는 Playlist를 쿼리**하려면, 단일 Song 객체 및 Song이 포함된 모든 Playlist 객체 목록을 포함하는 새 데이터 클래스를 만든다.

어느 경우든 이러한 각 클래스의 **@Relation 어노테이션에서 associateBy 속성**을 사용하여 엔터티 간의 관계를 모델링함으로써 Playlist 엔터티와 Song 엔터티 간의 관계를 제공하는 **상호 참조 엔터티를 식별**한다.

```kotlin
//Playlist 및 Playlist에 상응하는 Song 목록을 쿼리
data class PlaylistWithSongs(
		@Embedded val playlistL: Playlist,
		@Relation(
				**parentColumn = "playlistId",
				entityColumn = "songId",**
				**associateBy** = @Juncation(PlaylistSongCrossRef::class)
		)
		val songs: List<Song>
)
//Song 및 Song에 상응하는 Playlist 목록을 쿼리
data class SongWithPlaylists(
		@Embedded val song: Song,
		@Relation(
				**parentColumn = "songId",
				entityColumn = "playlistId",**
				**associateBy** = @Juncation(PlaylistSongCrossRef::class)
		)
		val playlists: List<Playlist>
)
```

마지막으로, DAO 클래스에 메서드를 추가하여 앱에 필요한 쿼리 기능을 만든다.

* _getPlaylistsWithSongs_ : 해당 메서드는 데이터베이스를 쿼리하고, 결과 _PlaylistWithSongs_ 객체를 모두 반환한다.
* _getSongsWithPlaylists_: 해당 메서드는 데이터베이스를 쿼리하고, 결과 _SongWithPlaylists_ 객체를 모두 반환한다.

```kotlin
@Transaction
@Query("SELECT * FROM Playlist")
fun getPlaylistsWithSongs(): List<PlaylistWithSongs>

@Transaction
@Query("SELECT * FROM Song")
fun getSongsWithPlaylists(): List<SongWithPlaylists>
```

🗨️ @Relation 어노테이션이 특정 사용 사례를 충족하지 않으면, SQL 쿼리에서 JOIN 키워드를 사용하여 관계를 수동으로 정의해야 할 수 있다.

### Define nested relationships

때때로, 서로 관련이 있는 세 개 이상의 테이블 집합을 쿼리해야 할 수도 있다. 이러한 경우 테이블 간의 중첩된 관계를 정의한다.

이전의 예인 음악 스트리밍 앱에서 모든 **사용자** & 각 사용자의 모든 **재생목록** & 각 사용자의 각 재생목록에 있는 모든 **노래**를 쿼리하려고 한다고 가정해보자. 사용자는 재생목록과 1 : 다 관계가 존재하며, 재생목록은 노래와 다 : 다 관계가 존재한다. 다음 코드에서는 이러한 엔터티를 나타내는 클래스뿐만 아니라 재생목록과 노래 간의 다 : 다 관계에 관한 상호 참조 테이블을 보여준다.

```kotlin
@Entity
data class User(
		@PrimaryKey val userId: Long,
		val name: String,
		val age: Int
)

@Entity
data class Playlist(
		@PrimaryKey val playlistId: Long,
		val userCreatorId: Long,
		val playlistName: String
)

@Entity
data class Song(
		@PrimaryKey val songId: Long,
		val songName: String,
		val artist: String
)

@Entity(primaryKeys = ["playlistId", "songId"])
data class PlaylistSongCrossRef(
		val playlistId: Long,
		val songId: Long
)
```

먼저 데이터 클래스 및 @Relation 어노테이션을 사용하여 평소처럼 집합 내 두 테이블 간의 관계를 모델링한다. 다음 예에서는 Playlist 엔터티 클래스와 Song 엔터티 클래스 간의 다 : 다 관계를 모델링하는 PlaylistWithSongs 클래스를 보여준다.

```kotlin
data class PlaylistWithSongs(
		@Embedded val playlist: Playlist,
		@Relation(
				parentColumn = "playlistId",
				entityColumn = "songId",
				associateBy = @Junction(PlaylistSongCrossRef::class)
		)
		val songs: List<Song>
)
```

이 관계를 나타내는 데이터 클래스를 정의한 후, 집합의 다른 테이블과 첫 번째 관계 클래스 간의 관계를 모델링하여 새 관계 내에서 기존 관계를 '중첩'하는 또 다른 데이터 클래스를 만든다.

```kotlin
data class UserWithPlaylistsAndSongs {
		@Embedded val user: User
		@Relation(
				entity = Playlist::class,
				parentColumn = "userId",
				entityColumn = "userCreatorId"
		)
		val playlists: List<PlaylistWithSongs>
}
```

UserWithPlaylistsAndSongs 클래스는 세 가지의 모든 엔터티 클래스(User, Playlist, Song) 간의 관계를 간접적으로 모델링한다.\


![](<../../../../.gitbook/assets/Untitled (2) (1).png>)

집합에 테이블이 더 많이 있다면, 나머지의 각 테이블 간 관계를 모델링하는 클래스 및 이전의 모든 테이블 간의 관계를 모델링하는 관계 클래스를 만든다. 이렇게 하면 쿼리하려는 모든 테이블 간에 중첩된 관계 체인이 생성된다.

마지막으로, DAO 클래스에 메서드를 추가하여 앱에 필요한 쿼리 기능을 만든다. 이 메서드를 사용하려면 Room에서 여러 쿼리를 실행해야 하므로, @Transaction 어노테이션을 추가하여 전체 작업이 원자적으로 실행되도록 해야 한다.

```kotlin
@Transaction
@Query("SELECT * FROM User")
fun getUsersWithPlaylistsAndSongs(): List<UserWithPlaylistsAndSongs>
```
