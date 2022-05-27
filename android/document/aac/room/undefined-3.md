# 데이터베이스 마이그레이션

앱에서 기능을 추가 및 변경할 때, Room 엔터티 클래스를 수정하여 이러한 변경사항을 반영해야 한다. 그러나 앱 업데이트가 데이터베이스 스키마를 변경할 때 **이미 기기 내 데이터베이스에 있는 사용자 데이터를 유지**하는 것이 중요하다.

Room 영속성(persistence) 라이브러리는 이러한 요구를 해결하기 위해 Migration 클래스를 사용하여 증분(incremental) 마이그레이션을 지원한다.

각 Migration 서브클래스는 Migration.migrate() 메서드를 재정의하여 startVersion과 endVersion 간 마이그레이션 경로를 정의한다.

앱 업데이트에 데이터베이스 버전 업그레이드가 필요할 때 Room은 하나 이상의 Migration 서브클래스에서 migrate() 메서드를 실행하여 런타임 시 데이터베이스를 최신 버전으로 마이그레이션한다.

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
	override fun migrate(database: SupportSQLiteDatabase) {
			database.execSQL("CREATE TABLE `Fruit` (`id` INTEGER, `name` TEXT, " +
										"PRIMARY KEY(`id`))")
	}
}

val MIGRATION_2_3 = object : Migration(2, 3) {
	override fun migrate(database: SupportSQLiteDatabase) {
			database.execSQL("ALTER TABLE Book ADD COLUMN pub_year INTEGER")
	}
}

Room.databaseBuilder(applicationContext, MyDb::class.java, "database-name")
		.addMigrations(MIGRATION_1_2, MIGRATION_2_3).build()
```

마이그레이션 프로세스가 완료되면 Room은 스키마의 유효성을 검사하여 마이그레이션이 성공했는지 확인한다. Room은 문제를 발견하면 일치하지 않는 정보가 포함된 예외를 발생시킨다.



### 누락된 마이그레이션 경로의 적절한 처리

Room이 기기의 기존 데이터베이스를 현재 버전으로 업그레이드하기 위한 마이그레이션 경로를 찾을 수 없다면 IllegalStateException이 발생한다. 마이그레이션 경로가 누락되었을때 기존 데이터를 잃어도 괜찮다면 데이터베이스 생성 시 다음과 같이 **fallbackToDestructiveMigration()** 빌더 메서드를 호출한다.

```kotlin
Room.databaseBuilder(applicationContext, MyDb::class.java, "database-name")
        **.fallbackToDestructiveMigration()**
        .build()
```

해당 메서드는 정의된 마이그레이션 경로가 없는 증분 마이그레이션을 실행해야 할 때 앱의 데이터베이스에 테이블을 destructive 방식으로 다시 생성하도록 Room에 지시한다.

🗨️ 앱의 데이터베이스 빌더에서 해당 옵션을 설정하면 Room이 정의된 마이그레이션 경로 없이 마이그레이션을 실행하려고 할 때, 데이터베이스의 테이블에서 **모든 데이터를 영구적으로 삭제한**다.



Room이 특정 상황에서만 destructive 재생성으로 대체하도록 하려면 다음과 같이 fallbackToDestructiveMigration()의 대안이 몇 가지 존재한다.

* 스키마 histoty 특정 버전에서 마이그레이션 경로로 해결할 수 없는 오류가 발생하면 fallbackToDestructiveMigrationFrom()을 사용해라. 해당 메서드를 사용하면 특정 버전에서 마이그레이션 할 때만 Room이 destructive 재생성으로 대체한다.
* 데이터베이스 상위 버전에서 하위 버전으로 마이그레이션할 때만 Room이 destructive 재생성으로 대체하도록 하려면 fallbackToDestructiveMigrationOnDowngrade()를 사용해라.

### Room 2.2.0으로 업그레이드 시 열 기본값 처리

Room 2.2.0 이상에서 @ColumnInfo(defaultValue = "...") 애노테이션을 사용하여 열의 기본값을 정의할 수 있다. 2.2.0보다 낮은 버전에서 열의 기본값을 정의하는 유일한 방법은 실행된 SQL 문에서 직접 기본값을 정의하는 방법밖에 없었다. 이는 Room이 인지하지 못하는 기본값을 생성하는 것이다. 즉, 데이터베이스가 원래 2.2.0보다 낮은 Room 버전으로 생성되었다면, Room 2.2.0을 사용하도록 앱을 업그레이드하려고 할 때 Room API를 사용하지 않고 정의한 기존 기본값에 특수한 마이그레이션 경로를 제공해야 할 수 있다.

예를 들어 데이터베이스 버전 1이 Song 엔터티를 정의한다고 가정해보자.

```kotlin
//Song Entity, DB version 1, Room 2.1.0
@Entity
data class Song(
		@PrimaryKey
		val id: Long,
		val title: String
)
```

또한, 동일한 데이터베이스의 버전 2가 새로운 NOT NULL 열을 추가하고, 버전 1에서 버전 2로의 마이그레이션 경로를 정의한다고 가정한다.

```kotlin
//Song Entity, DB version 2, Room 2.1.0
@Entity
data class Song(
		@PrimaryKey
		val id: Long,
		val title: String,
		val tag: String // Added in version 2
)

//Migration from 1 to 2, Room 2.1.0
val MIGRATION_1_2 = object: Migration(1, 2) {
	override fun migrate(database: SupportSQLiteDatabase) {
			database.execSQL(
							"ALTER TABLE Song ADD COLUMN tag TEXT NOT NULL DEFAULT ''"
			)
	}
}
```

이로 인해 기본 테이블에 앱 업데이트와 신규 설치 간 불일치가 발생한다. tag 열의 기본값은 버전 1에서 버전 2로의 마이그레이션 경로에만 선언되므로 버전 2부터 앱을 설치한 모든 사용자는 데이터베이스 스키마에 tag의 기본값을 얻을 수 없다.

2.2.0보다 낮은 Room 버전에서는 이러한 불일치 오류와 관계가 없다. 하지만 나중에 앱이 Room 2.2.0 이상을 사용하도록 업그레이드하고 @ColumnInfo 애노테이션을 사용하여 tag의 기본값을 포함하도록 Song 엔터티 클래스를 변경하면 이제 Room에서 이러한 불일치 오류를 확인할 수 있다. 이로 인해서 스키마 유효성 검사가 실패하는 요인이 된다.

초기에 마이그레이션 경로에서 열 기본값이 선언될 때 데이터베이스 스키마가 모든 사용자에게 일관되도록 하려면 Room 2.2.0 이상을 사용하도록 앱을 처음 업그레이드할 때 다음을 실행한다.

1. @ColumnInfo 애노테이션을 사용하여 각 엔터티 클래스에서 열 기본값을 선언한다.
2. 버전 업그레이드 할 때 마이그레이션 시 데이터베이스 버전 번호 1씩 늘린다.
3. 삭제 및 재생성 전략을 구현하는 새 버전 마이그레이션 경로를 정의하여 필요한 기본값을 기존 열에 추가한다.

🗨️ 앱의 데이터베이스가 destructive 마이그레이션으로 fallback 되는 경우, 또는 기본값이 있는 열을 추가하는 마이그레이션 경로가 없는 경우 해당 프로세스는 필요하지 않다.



```kotlin
// Migration from 2 to 3, Room 2.2.0
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("""
                CREATE TABLE new_Song (
                    id INTEGER PRIMARY KEY NOT NULL,
                    name TEXT,
                    tag TEXT NOT NULL DEFAULT ''
                )
                """.trimIndent())
        database.execSQL("""
                INSERT INTO new_Song (id, name, tag)
                SELECT id, name, tag FROM Song
                """.trimIndent())
        database.execSQL("DROP TABLE Song")
        database.execSQL("ALTER TABLE new_Song RENAME TO Song")
    }
}
```
