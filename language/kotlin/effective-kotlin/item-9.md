---
description: use를 사용하여 리소스를 닫아라
---

# item 9

리소스가 더 이상 필요하지 않을 때, close 메서드를 사용해서 명시적으로 닫아야 하는 리소스가 있습니다.

* InputStream과 OutputStream
* java.sql.Connection
* java.io.Reader(FileReader, BudderdReader)
* java.new.Socket과 java.util,Scanner

위와 같은 리소스들은 AutoCloseable을 상속받는 Closeable 인터페이스를 구현하고 있습니다. 이러한 모든 리소스는 최종적으로 리소스에 대한 레퍼런스가 없어질 때, 가비지 컬렉터가 처리합니다. 하지만 이는 굉장히 느리고 리소스를 유지하는 비용이 많이 들어갑니다.

따라서 명시적으로 close 메서드를 호출해서 처리합니다. 일반적으로 try-finally 블록을 사용해서 처리했습니다.

하지만 try-finally 블록 코드는 굉장히 복잡하고 좋지 않습니다. 리소스를 닫을 때 예외가 발생할 수 있는데, 이러한 예외를 따로 처리하지 않기 때문입니다. 또한, try 블록과 finally 블록 내부에서 오류가 발생하면, 둘 중 하나만 처리됩니다. 2개 모두 처리하면 좋겠지만, 이를 구현하는 것은 매우 어려운 일입니다.

이를 해결해주는 표준 라이브러리에 use 메서드를 사용하면 됩니다.

```kotlin
//try-finally
fun countCharactersInFile(path: String): Int {
	val reader = BufferdReader(FileReader(path))
	try {
		return reader.lineSequence().sumBy { it.lenght }
	} finally {
		reader.close()
	}
}

//use
fun countCharactersInFile(path: String): Int {
	val reader = BufferdReader(FileReader(path))
	reader.use {
		return reader.lineSequence().sumby { it.lenght }
	}
}

//use(람다 매개변수)
fun countCharactersInFile(path: String): Int {
	BufferdReader(FileReader(path)).use { reader ->
				return reader.lineSequence().sumby { it.lenght }
	}
}

//useLines(파일을 한 줄씩 처리할 때 활용)
fun countCharactersInFile(path: String): Int {
	File(path).useLines { lines ->
		return lines.sumBy { it.length }
	}
}

//useLines(파일을 특정 줄을 두 번이상 반복 처리)
fun countCharactersInFile(path: String): Int {
	File(path).useLines { lines ->
		lines.sumBy { it.length }
	}
}
```

useLines을 사용하면 메모리에 파일의 내용을 한 줄씩만 유지하므로, 대용량 파일도 적절하게 처리할 수 있습니다. 다만 파일의 줄을 한 번만 사용할 수 있다는 단점이 존재합니다.
