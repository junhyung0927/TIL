---
description: 기
---

# Room 데이터베이스에 미리 채우기

때때로, 특정 데이터 셋이 이미 로드된 데이터베이스로 앱을 시작하기를 원할 수 있다. 이를 미리 채우기(prepopulating)라고 부른다. Room 2.2.0 버전 이상에서는 API 메서드를 사용하여 초기화 시 기기 파일 시스템의 사전 패키징된 데이터베이스 파일의 콘텐츠로 Room 데이터베이스를 미리 채울 수 있다.

🗨️ 메모리 내 Room 데이터베이스는 createFromAsset() 또는 createFromFile()을 사용한 데이터베이스 미리 채우기를 지원하지 않는다. 즉, Room을 사용해서 처음에 데이터베이스를 생성하기 전에 작성해야 한다.

### Prepopulate from the file system

앱의 _assets/ 디렉터리를 제외한_ 기기의 파일 시스템 어딘가에 있는 사전 패키징된 데이터베이스 파일에서 Room 데이터베이스를 미리 채우려면 build()를 호출하기 전에 RoomDatabase.Builder 객체에서 createFromFile() 메서드를 호출한다.

```kotlin
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
		**.createFromFile(File("mypath"))**
		.build()
```

createFromFile() 메서드는 사전 패키징된 데이터베이스 파일에 File 인수를 허용한다. Room은 지정된 파일을 직접 열지 않고 파일의 사본을 생성하므로 앱의 **파일 읽기 권한이 있는지 확인**한다.



### 사전 패키징된 데이터베이스가 포함된 migrations 처리

기존에 데이터베이스에 있던(사전 패키징된) 파일은 Room 데이터베이스가 대체(fallback) 마이그레이션을 처리하는 방식을 변경할 수 있다. 일반적으로 _destructive migrations_을 설정하여 사용했으며, Room에서 마이그레이션 경로 없이 마이그레이션을 실행해야 할 때, Room은 데이터베이스의 모든 테이블을 삭제하고 타겟 버전용으로 지정된 스키마로 빈 데이터베이스를 생성한다. 그러나 타겟 버전과 동일한 버전의 사전 패키징된 데이터베이스 파일을 포함하면 Room은 _destructive migrations_을 실행한 후, 새롭게 다시 생성된 데이터베이스를 사전 패키징된 데이터베이스 파일의 콘텐츠로 채우려고 시도한다.&#x20;

#### 사전 패키징된 데이터베이스를 사용한 fallback 마이그레이션

* 앱이 버전 3에서 Room 데이터베이스 정의
* 기기에 이미 설치된 데이터베이스 인스턴스는 버전 2에 존재
* 버전 3에 사전 패키징된 데이터베이스 파일이 존재
* 버전 2에서 버전 3으로 구현된 마이그레이션 경로 존재하지 않음
* _destructive migrations_이 사용 설정되어 있음

```kotlin
//Database class definition declaring version 3
@Database(version = 3)
abstract class AppDatabase : RoomDatabase() {
...
}

//Destructive migrations are enabled and a prepackaged database is provided
Room.databaseBuilder(appContext, AppDatabase.class, "Simple.db")
		.createFromAsset("database/myapp.db")
		.fallbackToDestructiveMigration()
		.build()
```

1. 앱에 정의된 데이터베이스는 버전 3에 존재하고, 이미 설치된 데이터베이스 인스턴스는 버전 2에 있으므로 마이그레이션이 필요하다.
2. 버전 2에서 버전 3으로의 구현된 마이그레이션 계획이 없으므로 fallback 마이그레이션이다.
3. fallbackToDestructiveMigration() 빌더 메서드가 호출되므로 fallback 마이그레이션은 _destructive_하다. Room은 기기에 설치된 데이터베이스 인스턴스를 삭제한다.
4. 버전 3에 사전 패키징된 데이터베이스 파일이 있으므로 Room은 데이터베이스를 다시 생성하고, 사전 패키징된 데이터베이스 파일의 콘텐츠를 사용하여 새로 생성된 데이터베이스를 채운다. 반면에 사전 패키징된 데이터베이스 파일이 버전 2에 있다면, Room은 이 파일이 타겟 버전과 일치하지 않음을 인지하고 해당 파일을 fallback 마이그레이션의 일부로 사용하지 않는다.

#### 사전 패키징된 데이터베이스로 구현된 마이그레이션

_앱이 버전 2에서 버전 3으로의 fallback 경로를 구현한다고 가정_

```kotlin
//Database class definition declaring version 3
@Database(version = 3)
abstract class AppDatabase : RoomDatabase() {
...
}

//Migration path definition from version 2 to version 3
val MIGRATION_2_3 = object : Migration(2,3) {
	override fun migrate(database: SupportSQLiteDatabase) {
			 ...
	}
}

//A prepackaged database is provided
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
		.createFromAsset("dataabase/myapp.db")
		.addMigrations(MIGRATION_2_3)
		.build()
```

1. 앱에 정의된 데이터베이스는 버전 3에 있고, 기기에 이미 설치된 데이터베이스는 버전 2에 있으므로 마이그레이션이 필요하다.
2. 버전 2에서 버전 3으로의 구현된 마이그레이션 경로가 있으므로 Room은 정의된 migrate() 메서드를 실행하여 기기의 데이터베이스 인스턴스를 버전 3으로 업데이트한다. 이미 데이터베이스에 있는 데이터는 보존한다. Room은 fallback 마이그레이션의 경우에만 사전 패키징된 데이터베이스 파일을 사용하기 때문에, Room은 사전 패키징된 데이터베이스 파일을 사용하지 않는다.

#### 사전 패키징된 데이터베이스를 사용한 Multi-step 마이그레이션

사전 패키징된 데이터베이스 파일은 여러 단계로 구성된 마이그레이션에 영향을 줄 수 있다.

* 앱이 버전 4에서 Room 데이터베이스를 정의한다.
* 기기에 이미 설치된 데이터베이스 인스턴스는 버전 2에 있다.
* 버전 3에 사전 패키징된 데이터베이스 파일이 존재한다.
* 버전 3에서 버전 4로의 구현된 마이그레이션 경로가 있지만, 버전 2에서 버전 3으로의 마이그레이션 경로는 없다.
* _destructive_ 마이그레이션이 사용 설정 되었다.

```kotlin
//Database class definition declaring version 4
@Database(version = 4)
abstract class AppDatabase : RoomDatabase() {
...
}

//Migration path definition from version 3 to version 4
val MIGRATION_3_4 = object : Migration(3, 4) {
	override fun migrate(database: SupportSQLiteDatabase) {
			 ...
	}
}

Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
		.createFromAsset("database/myapp.db")
		.addMigration(MIGRATION_3_4)
		.fallbackToDestructiveMigration()
		.build()
```

1. 앱에 정의된 데이터베이스는 버전 4에 있고, 기기에 이미 설치된 데이터베이스 인스턴스는 버전 2에 있으므로 마이그레이션이 필요하다.
2. 버전 2에서 버전 3으로의 구현된 마이그레이션 경로가 없으므로 fallback 마이그레이션이다.
3. fallbackToDestructiveMigration() 빌더 메서드가 호출되므로 fallback 마이그레이션은 _destructive_하다.
4. 버전 3에 사전 패키징된 데이터베이스 파일이 있으므로 Room은 데이터베이스를 다시 생성하고 사전 패키징된 데이터베이스 파일의 콘텐츠를 사용하여 새로 생성된 해당 데이터베이스를 채운다.
5. 기기에 설치된 데이터베이스는 이제 버전 3에 존재한다. 이는 여전히 앱에 정의된 버전보다 낮으므로 또 다른 마이그레이션이 필요하다.
6. 버전 3에서 버전 4로의 구현된 마이그레이션 경로가 있으므로 Room은 정의된 migrate() 메서드를 실행하여 기기의 데이터베이스 인스턴스를 버전 4로 업데이트하며, 버전 3의 사전 패키징된 데이터베이스 파일에서 복사된 데이터는 보존한다.



