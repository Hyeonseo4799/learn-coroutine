# 1_6_CEH와 슈퍼바이저 잡

### GlobalScope
- 어디에도 속하지 않지만 원래부터 존재하는 전역적인 스코프.
- 어떤 계층에도 속하지 않고 영원히 동작하게 된다는 문제점이 있다.
- 프로그래밍에서 전역 객체를 잘 사용하지 않는 것처럼 잘 사용되지 않는다.

```kotlin
suspend fun printRandom() {
    delay(500L)
    println(Random.nextInt(0, 500))
}

fun main() {
    val job = GlobalScope.launch(Dispatchers.IO) {
        launch { printRandom() }
    }
    Thread.sleep(1000L)
}
 
```

### CoroutineScope
- `GlobalScope`보다 권장되는 방식은 `CoroutineScope`를 사용하는 것이다.
- `CoroutineScope`는 인자로 `CoroutineContext`를 받는다.

```kotlin
suspend fun printRandom() {
    delay(500L)
    println(Random.nextInt(0, 500))
}

fun main() {
    val scope = CoroutineScope(Dispatchers.Default)
    val job = scope.launch(Dispatchers.IO) {
        launch { printRandom2() }
    }
    Thread.sleep(1000L)
}
```

### CEH (코루틴 익셉션 핸들러)
- 예외를 가장 체계적으로 다루는 방법은 CEH(Coroutine Exception Handler)를 사용하는 것이다.
- 일반적으로 CEH를 만들 때 첫 번째 인자인 `CoroutineContext` 는 잘 사용하지 않으므로 `_`로 처리한다.

```kotlin
suspend fun printRandom1() {
    delay(1000L)
    println(Random.nextInt(0, 500))
}

suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}

val ceh = CoroutineExceptionHandler { _, exception ->
    println("Something happend: $exception")
}

fun main() = runBlocking<Unit> {
    val scope = CoroutineScope(Dispatchers.IO)
    val job = scope.launch(ceh) {
        launch { printRandom1() }
        launch { printRandom2() }
    }
    job.join()
}
```

### runBlocking과 CEH
- `runBlocking`은 자식이 예외로 종료되면 항상 종료되고 CEH를 호출하지 않는다.

```kotlin
suspend fun printRandom1() {
    delay(1000L)
    println(Random.nextInt(0, 500))
}

suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}

val ceh = CoroutineExceptionHandler { _, exception ->
    println("Something happend: $exception")
}

fun main() = runBlocking<Unit> { // 1 최상단 코루틴
    val job = launch(ceh) {// 2
        val a = async { printRandom1() } // 3
        val b = async { printRandom2() } // 3
        println(a.await())
        println(b.await())
    }
    job.join()
}
```

### SupervisorJob
- 슈퍼바이저 잡은 예외에 의한 취소를 아래쪽으로 내려가게 한다.
- `joinAll`은 복수개의 `Job`에 대해 `join`을 수행하여 완전히 종료될 때 까지 기다린다.

```kotlin
suspend fun printRandom1() {
    delay(1000L)
    println(Random.nextInt(0, 500))
}

suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}

val ceh = CoroutineExceptionHandler { _, exception ->
    println("Something happend: $exception")
}

fun main() = runBlocking<Unit> {
    val scope = CoroutineScope(Dispatchers.IO + SupervisorJob() + ceh)
    val job1 = scope.launch { printRandom1() }
    val job2 = scope.launch { printRandom2() }
    joinAll(job1, job2) 
}
```

### SupervisorScope
- 코루틴 스코프와 슈퍼바이저 잡을 합친 것처럼 동작한다.
- 무조건 문제가 발생한 곳에 CEH를 붙이거나 try-catch를 해야하는데, 그렇지 않으면 외부에 에러가 전파된다.
    
```kotlin
suspend fun printRandom1() {
    delay(1000L)
    println(Random.nextInt(0, 500))
}
    
suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}
    
suspend fun supervisoredFunc() = supervisorScope {
    launch { printRandom1() }
    launch(ceh) { printRandom2() }
}
    
val ceh = CoroutineExceptionHandler { _, exception ->
    println("Something happend: $exception")
}
    
fun main() = runBlocking<Unit> {
    val scope = CoroutineScope(Dispatchers.IO)
    val job = scope.launch {
        supervisoredFunc()
    }
    job.join()
}
```