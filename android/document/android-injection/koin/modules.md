# Modules

Koin을 사용하면 모듈의 정의를 설명할 수 있다. 해당 섹션에서는 모듈을 어떻게 정의하고 구성하고 연결하는 방법을 살펴본다.

### What is a module?

Koin 모듈은 Koin의 정의를 작성하는 "space"이다. 작성은 `module` 함수 내에서 정의한다.

```kotlin
val myModule = module {
	// 작성 ...
}
```

### Using several modules

컴포넌트가 반드시 동일한 모듈에 있을 필요는 없다. 모듈은 작성하고자 하는 정의를 구성하는데 도움이 되는 논리적인 공간이며, 다른 모듈의 정의에 따라 달라질 수 있다. 작성하는 것은 lazy하게 흘러가고, 컴포넌트가 이를 요청할 때만 해결된다.

각 모듈에 연결된 컴포넌트를 예로 들어보자.

```kotlin
class ComponentA()
class ComponentB(val componentA : ComponentA)

val moduleA = module {
	//Singleton ComponentA
	single { ComponentA() }
}

val moduleB = module {
	//Singleton ComponentB with linked instance ComponentA
	single { ComponentB(get()) }
}
```

🗨️ Koin은 import 개념이 없다. Koin은 lazy하게 정의된다: Koin은 Koin 컨터이너에서 시작되지만 인스턴스화되진 않는다. 인스턴스 타입에 대한 요청만 생성된다.



Koin 컨테이너를 시작할 때 사용된 모듈의 list를 선언하면 된다.

```kotlin
//Start Koin with moduleA & moduleB
startKoin {
	modules(moduleA, moduleB)
}
```

### Linking modules strategies

모듈 간 선언은 lazy하게 되기 때문에, 다른 계획에 대한 구현을 구현하기 위해 모듈을 사용할 수 있다: 모듈별 구현을 정의한다.

Repository와 Datasource의 예제를 살펴보자. repository는 datasource가 필요하고, datasource를 구현할 수 있는 방법은 2가지가 있다: Local or Remote

```kotlin
class Repository(val datasource : Datasource)
interface Datasource
class LocalDatasource() : Datasource
class RemoteDatasource() : Datasource
```

다음 3개의 모듈에 컴포넌트를 선언할 수 있다: Repository와 Datasource 구현 당 1개

```kotlin
val repositoryModule = module {
	single { Repository(get()) }
}

val localDatasourceModule = module {
	single<Datasource> { LocalDatasource() }
}

val remoteDatasourceModule = module {
	single<Datasource> { RemoteDatasource() }
}
```

그런 다음 올바른 모듈 조합으로 Koin을 출시하면 된다.

```kotlin
//Load Repository + Local Datasource definitions
startKoin {
	modules(repositoryModule, localDatasourceModule)
}

//Load Repository + Remote Datasource definitinos
startKoin {
	modules(repositoryModule, remoteDatasourceModule)
}
```

### Overriding definition or module

Koin은 이미 존재하는 정의를 재정의할 수 없다(type, name, path). 다음과 같이 작성하면 오류를 발생시킨다.

```kotlin
val myModuleA = module {
	single<**Service**> { ServiceImp() }
}

val myModuleB = module {
	single<**Service**> { TestServiceImp() }
}

//Will throw an BeanOverrideException 기존에 있는 타입을 선언해서 에러
startKoin {
	modules(myModuleA, myModuleB)
}
```

중복된 정의를 재정의하려면, `override` 매개변수를 사용해야 한다.

```kotlin
val myModuleA = module {
	single<Service> { ServiceImp() }
}

val myModuleB = module {
	//override for this definition
	single<Service>(**override#true**) { TestServiceImp() }
}
```

```kotlin
val myModuleA = module {
	single<Service> { ServiceImp() }
}

//Allow override for all definitions from module
val myModuleB = module(**override#true**) {
	single<Service> { TestServiceImp() }
}
```

🗨️ 모듈들을 나열 및 정의 override 할때 순서가 중요하다. 모듈 list의 맨 끝에 재정의 정의가 있어야한다.

