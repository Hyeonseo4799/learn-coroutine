# 1_7_공유 객체, Mutex, Actor

### 공유 객체 문제
- `withContext` 는 수행이 완료될 때까지 기다리는 코루틴 빌더이다.
- 아래 코드를 실행하면 항상 10만이 나오진 않는다.
- 이는 여러 스레드 간 서로 다른 정보를 가지고 있기 때문이다.

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100 // 시작할 코루틴의 갯수
    val k = 1000 // 코루틴 내에서 반복할 횟수
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms 동안 ${n * k}개의 액션을 수행했습니다.")
}

var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

### volatile을 적용하기
- `volatile`은 가시성을 제공해주는 어노테이션이다.
- `volatile`로 지정한 값은 어떤 스레드에서 변경을 해도 다른 스레드에 영향을 준다.
- 하지만 아래 코드도 항상 10만이 나오진 않는데, 현재 값을 동시에 읽고 수정해서 생기는 문제는 해결하지 못했기 때문이다.

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100 // 시작할 코루틴의 갯수
    val k = 1000 // 코루틴 내에서 반복할 횟수
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms 동안 ${n * k}개의 액션을 수행했습니다.")
}

@Volatile
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

### 스레드 안전한 자료구조 사용하기
- `AtomicInteger`와 같은 스레드 안전한 자료구조를 사용하는 방법이 있다.
- `incrementAndGet()`은 자료의 값을 읽고 쓸 때 다른 스레드가 값을 변경할 수 없게 한다.
- 모든 문제에서 스레드 안전한 자료구조가 정답은 아니다.

```kotlin
suspend fun massiveRun(ac tion: suspend () -> Unit) {
    val n = 100 // 시작할 코루틴의 갯수
    val k = 1000 // 코루틴 내에서 반복할 횟수
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms 동안 ${n * k}개의 액션을 수행했습니다.")
}

var counter = AtomicInteger()

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
```

### 스레드 한정
- `newSingleThreadContext`를 이용해서 특정한 스레드를 만들고 해당 스레드를 사용한다.
- 아래 코드를 실행시키면 항상 10만이 나오는 것을 확인할 수 있다.
    
```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100 // 시작할 코루틴의 갯수
    val k = 1000 // 코루틴 내에서 반복할 횟수
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms 동안 ${n * k}개의 액션을 수행했습니다.")
}
    
var counter = 0
val counterContext = newSingleThreadContext("CounterContext")
    
fun main() = runBlocking {
    withContext(counterContext) { // 전체 코드를 하나의 스레드에서
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

### 뮤텍스
- 뮤텍스는 상호배제(Mutual exclusion)의 줄임말이다.
- 공유 상태를 수정할 때 임계 영역(critical section)을 이용하게 하며, 임계 영역에 동시에 접근하는 것을 허용하지 않는다.

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100 // 시작할 코루틴의 갯수
    val k = 1000 // 코루틴 내에서 반복할 횟수
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms 동안 ${n * k}개의 액션을 수행했습니다.")
}

val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

### 액터
- 액터가 독점적으로 자료를 가지며 다른 코루틴과 공유하지 않고 액터를 통해서만 접근하게 한다.
- `CounterMsg`는 액터에게 신호를 전달해주기 위해 만들어진 객체이다.
- 채널은 송신 측에서 값을 `send` 할 수 있고 수신 측에서 `receive` 할 수 있는 도구이다.
- 액터는 비교적 최근에 만들어져서 일반적인 형태는 아니다.

```kotlin
sealed class CounterMsg
object IncCounter: CounterMsg()
class GetCounter(val response: CompletableDeferred<Int>): CounterMsg()

fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 

    for (msg in channel) { // suspension point
        when (msg) {
            is IncCounter -> counter++ 
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100 // 시작할 코루틴의 갯수
    val k = 1000 // 코루틴 내에서 반복할 횟수
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms 동안 ${n * k}개의 액션을 수행했습니다.")
}

fun main() = runBlocking<Unit> {
    val counter = counterActor()
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter) // suspension point
        }
    }

    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response)) // suspension point
    println("Counter = ${response.await()}") // suspension point
    counter.close()
}
```