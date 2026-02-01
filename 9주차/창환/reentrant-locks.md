# Java Lock 완전 정리

---

## 1. ReentrantLock vs synchronized 비교표

| 항목 | synchronized | ReentrantLock |
|---|---|---|
| 락 해제 | 자동 (블록 끝나면 JVM이 해제) | 수동 (`unlock()` 직접 호출 필수) |
| tryLock (락 시도) | 불가능 | 가능 (실패 시 다른 로직 수행 가능) |
| 타임아웃 | 불가능 | 가능 (`tryLock(time, unit)`) |
| 공정성(Fairness) | 항상 불공정 | 공정/불공정 선택 가능 |
| Condition (대기열) | `wait/notify` 1개만 사용 가능 | 여러 개의 Condition 생성 가능 |
| 인터럽트 대응 | 불가능 (락 대기 중 취소 불가) | `lockInterruptibly()`로 대기 중 취소 가능 |

---

## 2. 비교표 항목별 상세 설명

### 2-1. 락 해제 방식

**synchronized** — 블록이 끝나면 JVM이 알아서 락을 해제해 줍니다. 개발자가 신경 쓸 것이 없습니다.

```java
public void doWork() {
    synchronized (this) {
        // 이 블록이 끝나면 자동으로 락 해제
        // 예외가 발생해도 자동 해제
        process();
    }
}
```

**ReentrantLock** — 반드시 `finally` 블록에서 `unlock()`을 호출해야 합니다. 빠뜨리면 다른 스레드가 영원히 락을 획득하지 못합니다.

```java
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();
    try {
        process();
    } finally {
        lock.unlock(); // 이걸 빠뜨리면 데드락 발생
    }
}
```

> 단순한 동기화라면 synchronized가 실수할 여지가 없어서 더 안전합니다.

#### Kotlin에서는

Kotlin의 `@Synchronized` 어노테이션 또는 `synchronized()` 인라인 함수를 사용합니다. `ReentrantLock`은 `withLock` 확장 함수를 사용하면 try-finally를 직접 작성할 필요가 없습니다.

```kotlin
// synchronized — 방법 1: 어노테이션
@Synchronized
fun doWork() {
    process()
}

// synchronized — 방법 2: 인라인 함수
private val lockObj = Any()

fun doWork() {
    synchronized(lockObj) {
        process()
    }
}

// ReentrantLock — withLock 확장 함수 사용
import java.util.concurrent.locks.ReentrantLock
import kotlin.concurrent.withLock

private val lock = ReentrantLock()

fun doWork() {
    lock.withLock {
        // try-finally를 withLock이 내부적으로 처리해 줌
        process()
    }
}
```

> Kotlin의 `withLock`은 내부적으로 `lock()` → `try { block() } finally { unlock() }` 패턴을 자동 적용하므로 unlock 누락 실수를 방지할 수 있습니다.

---

### 2-2. tryLock (락 획득 시도)

synchronized는 락을 얻을 때까지 무조건 기다립니다. 포기하는 방법이 없습니다.

```java
// synchronized: 다른 스레드가 락을 잡고 있으면 무한 대기
synchronized (lockObj) {
    doSomething(); // 여기까지 오려면 무조건 기다려야 함
}
```

ReentrantLock은 `tryLock()`으로 "시도만 해보고 실패하면 다른 일을 하는" 로직이 가능합니다.

```java
ReentrantLock lock = new ReentrantLock();

public void doWork() {
    if (lock.tryLock()) {
        try {
            process();
        } finally {
            lock.unlock();
        }
    } else {
        // 락을 못 잡았을 때 대안 로직
        System.out.println("다른 스레드가 작업 중이므로 건너뜁니다.");
        doAlternativeWork();
    }
}
```

**실전 예시 — 데드락 방지 (계좌 이체)**

두 계좌에 동시에 락을 걸어야 할 때, synchronized는 순서가 꼬이면 데드락이 발생합니다. tryLock으로 양쪽 다 잡히는 경우에만 진행하면 데드락을 원천 차단할 수 있습니다.

```java
public void transfer(Account from, Account to, int amount) {
    while (true) {
        boolean gotFrom = from.lock.tryLock();
        boolean gotTo = to.lock.tryLock();

        if (gotFrom && gotTo) {
            try {
                from.withdraw(amount);
                to.deposit(amount);
                return; // 성공하면 종료
            } finally {
                from.lock.unlock();
                to.lock.unlock();
            }
        }

        // 하나라도 못 잡으면 잡은 것도 해제하고 재시도
        if (gotFrom) from.lock.unlock();
        if (gotTo) to.lock.unlock();

        Thread.yield(); // 다른 스레드에게 양보 후 재시도
    }
}
```

#### Kotlin에서는

```kotlin
import java.util.concurrent.locks.ReentrantLock

val lock = ReentrantLock()

fun doWork() {
    if (lock.tryLock()) {
        try {
            process()
        } finally {
            lock.unlock()
        }
    } else {
        println("다른 스레드가 작업 중이므로 건너뜁니다.")
        doAlternativeWork()
    }
}

// 데드락 방지 계좌 이체
fun transfer(from: Account, to: Account, amount: Int) {
    while (true) {
        val gotFrom = from.lock.tryLock()
        val gotTo = to.lock.tryLock()

        if (gotFrom && gotTo) {
            try {
                from.withdraw(amount)
                to.deposit(amount)
                return
            } finally {
                from.lock.unlock()
                to.lock.unlock()
            }
        }

        if (gotFrom) from.lock.unlock()
        if (gotTo) to.lock.unlock()
        Thread.yield()
    }
}
```

> `tryLock()`은 Java API를 그대로 사용합니다. `withLock`은 내부적으로 `lock()`을 호출하므로 tryLock 패턴에서는 사용할 수 없습니다.

---

### 2-3. 타임아웃

synchronized는 타임아웃 개념 자체가 없습니다. 락을 못 잡으면 무한 대기입니다.

ReentrantLock은 일정 시간 내에 락을 못 잡으면 포기할 수 있습니다.

```java
ReentrantLock lock = new ReentrantLock();

public void doWork() throws InterruptedException {
    // 3초 동안 시도하고, 못 잡으면 false 반환
    if (lock.tryLock(3, TimeUnit.SECONDS)) {
        try {
            process();
        } finally {
            lock.unlock();
        }
    } else {
        System.out.println("3초 안에 락을 획득하지 못했습니다.");
        handleTimeout();
    }
}
```

> 외부 리소스 접근처럼 응답이 보장되지 않는 상황에서 유용합니다. 무한 대기로 시스템이 멈추는 것을 방지할 수 있습니다.

#### Kotlin에서는

```kotlin
import java.util.concurrent.TimeUnit
import java.util.concurrent.locks.ReentrantLock

val lock = ReentrantLock()

fun doWork() {
    if (lock.tryLock(3, TimeUnit.SECONDS)) {
        try {
            process()
        } finally {
            lock.unlock()
        }
    } else {
        println("3초 안에 락을 획득하지 못했습니다.")
        handleTimeout()
    }
}
```

> Kotlin 코루틴 환경이라면 `Mutex`를 사용하는 것이 더 자연스럽습니다. `Mutex`는 `withTimeout`과 함께 사용할 수 있습니다.

```kotlin
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlinx.coroutines.withTimeout

val mutex = Mutex()

suspend fun doWork() {
    try {
        withTimeout(3000) {
            mutex.withLock {
                process()
            }
        }
    } catch (e: kotlinx.coroutines.TimeoutCancellationException) {
        println("3초 안에 락을 획득하지 못했습니다.")
        handleTimeout()
    }
}
```

---

### 2-4. 공정성(Fairness)

**불공정 락 (기본값)** — 락이 해제되는 순간 아무 스레드나 가져갈 수 있습니다. 먼저 기다렸는데 나중에 온 스레드가 먼저 잡을 수도 있습니다.

**공정 락** — 먼저 기다린 스레드가 먼저 락을 획득합니다 (FIFO).

```java
// 불공정 (기본값) — synchronized도 이 방식
ReentrantLock unfairLock = new ReentrantLock();        // 또는 new ReentrantLock(false)

// 공정 — 먼저 기다린 순서대로
ReentrantLock fairLock = new ReentrantLock(true);
```

synchronized는 항상 불공정 방식이며, 이 설정을 바꿀 수 없습니다.

> 공정 락은 기아(starvation) 현상을 방지하지만 성능이 떨어집니다. 대부분의 상황에서는 불공정 락이 더 빠르기 때문에, 기아 문제가 실제로 발생할 때만 공정 락을 사용합니다.

#### Kotlin에서는

```kotlin
import java.util.concurrent.locks.ReentrantLock

// 불공정 (기본값)
val unfairLock = ReentrantLock()          // 또는 ReentrantLock(false)

// 공정
val fairLock = ReentrantLock(true)        // 먼저 기다린 순서대로
```

> Java와 동일하게 `ReentrantLock(true)`로 공정성을 설정합니다. Kotlin의 `synchronized`도 Java와 마찬가지로 항상 불공정입니다.

---

### 2-5. Condition (다중 대기열)

synchronized의 `wait()/notify()`는 대기열이 하나뿐입니다. "누구를 깨울지" 구분할 수 없어서 `notifyAll()`로 전부 깨워야 합니다.

```java
// synchronized: 생산자와 소비자를 구분할 수 없음
synchronized (this) {
    while (isFull())
        wait();       // 생산자가 여기서 대기
    add(item);
    notifyAll();      // 생산자/소비자 가리지 않고 전부 깨움 → 불필요한 경쟁
}
```

ReentrantLock의 Condition은 대기열을 여러 개 만들 수 있습니다. 생산자용, 소비자용을 분리해서 정확히 필요한 쪽만 깨울 수 있습니다.

```java
public class BoundedQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition(); // 생산자 전용 대기열
    private final Condition notEmpty = lock.newCondition(); // 소비자 전용 대기열

    public BoundedQueue(int capacity) {
        this.capacity = capacity;
    }

    // 생산자
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity)
                notFull.await();        // 큐가 찰 때까지 생산자만 대기

            queue.add(item);
            notEmpty.signal();          // 소비자만 깨움
        } finally {
            lock.unlock();
        }
    }

    // 소비자
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty())
                notEmpty.await();       // 큐가 빌 때까지 소비자만 대기

            T item = queue.remove();
            notFull.signal();           // 생산자만 깨움
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

> `notEmpty.signal()`은 소비자만 깨우고, `notFull.signal()`은 생산자만 깨웁니다. synchronized의 `notifyAll()`처럼 전부 깨워서 불필요한 경쟁을 시키지 않습니다.

#### Kotlin에서는

```kotlin
import java.util.concurrent.locks.ReentrantLock
import java.util.LinkedList

class BoundedQueue<T>(private val capacity: Int) {
    private val queue = LinkedList<T>()
    private val lock = ReentrantLock()
    private val notFull  = lock.newCondition() // 생산자 전용 대기열
    private val notEmpty = lock.newCondition() // 소비자 전용 대기열

    // 생산자
    @Throws(InterruptedException::class)
    fun put(item: T) {
        lock.lock()
        try {
            while (queue.size == capacity)
                notFull.await()

            queue.add(item)
            notEmpty.signal()
        } finally {
            lock.unlock()
        }
    }

    // 소비자
    @Throws(InterruptedException::class)
    fun take(): T {
        lock.lock()
        try {
            while (queue.isEmpty())
                notEmpty.await()

            val item = queue.remove()
            notFull.signal()
            return item
        } finally {
            lock.unlock()
        }
    }
}
```

> 코루틴 환경이라면 `Channel`이 이 패턴을 이미 내장하고 있어서 직접 구현할 필요가 없습니다.

```kotlin
import kotlinx.coroutines.channels.Channel

val channel = Channel<String>(capacity = 10) // bounded channel

// 생산자
suspend fun produce() {
    channel.send("data") // 가득 차면 자동으로 일시 중단
}

// 소비자
suspend fun consume() {
    val data = channel.receive() // 비어 있으면 자동으로 일시 중단
}
```

---

### 2-6. 인터럽트 대응

synchronized로 락을 대기하는 중에는 `Thread.interrupt()`를 호출해도 반응하지 않습니다. 락을 얻을 때까지 무조건 기다려야 합니다.

```java
// synchronized: 대기 중 인터럽트 불가
Thread t = new Thread(() -> {
    synchronized (lockObj) { // 여기서 대기 중이면 interrupt()가 먹히지 않음
        doWork();
    }
});
t.start();
t.interrupt(); // 효과 없음, 계속 대기
```

ReentrantLock은 `lockInterruptibly()`를 사용하면 대기 중에 인터럽트로 취소할 수 있습니다.

```java
ReentrantLock lock = new ReentrantLock();

Thread t = new Thread(() -> {
    try {
        lock.lockInterruptibly(); // 대기 중에 인터럽트 가능
        try {
            doWork();
        } finally {
            lock.unlock();
        }
    } catch (InterruptedException e) {
        System.out.println("락 대기가 취소되었습니다.");
        // 정리 작업 수행
    }
});
t.start();
t.interrupt(); // InterruptedException 발생 → 대기 탈출
```

> 사용자가 취소 버튼을 누르거나, 서버 셧다운 시 대기 중인 스레드를 빠져나오게 할 때 유용합니다.

#### Kotlin에서는

```kotlin
import java.util.concurrent.locks.ReentrantLock

val lock = ReentrantLock()

val t = Thread {
    try {
        lock.lockInterruptibly()
        try {
            doWork()
        } finally {
            lock.unlock()
        }
    } catch (e: InterruptedException) {
        println("락 대기가 취소되었습니다.")
    }
}
t.start()
t.interrupt()
```

> 코루틴 환경에서는 `Mutex`가 기본적으로 취소(cancellation)를 지원합니다. 별도의 `lockInterruptibly` 같은 메서드가 필요 없습니다.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

val mutex = Mutex()

val job = launch {
    mutex.withLock {
        // 코루틴이 취소되면 자동으로 락 대기에서 빠져나옴
        doWork()
    }
}
job.cancel() // 대기 중이라면 즉시 취소됨
```

---

## 3. ReentrantReadWriteLock 상세

### 3-1. 이것이 왜 필요한가

일반 락(synchronized, ReentrantLock)은 읽기든 쓰기든 무조건 한 스레드만 진입할 수 있습니다.

```
[스레드A: 읽기] → [스레드B: 읽기] → [스레드C: 읽기]   ← 순차 처리 (느림)
```

그런데 읽기는 데이터를 변경하지 않으므로 여러 스레드가 동시에 읽어도 안전합니다. 읽기끼리 서로 블로킹하는 것은 불필요한 낭비입니다.

ReentrantReadWriteLock은 이 문제를 해결합니다.

```
[스레드A: 읽기]
[스레드B: 읽기]  ← 동시 진입 가능 (빠름)
[스레드C: 읽기]
```

### 3-2. 동작 규칙

| 현재 상태 | 읽기 요청 | 쓰기 요청 |
|---|---|---|
| 아무도 없음 | 즉시 진입 | 즉시 진입 |
| 읽기 락 보유 중 | **즉시 진입 (동시 가능)** | 대기 |
| 쓰기 락 보유 중 | 대기 | 대기 |

핵심: **읽기 + 읽기 = 동시 허용**, 그 외(읽기+쓰기, 쓰기+쓰기)는 모두 배타적입니다.

### 3-3. 기본 사용법

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.concurrent.locks.Lock;

public class ThreadSafeCache {
    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // 읽기 — 여러 스레드가 동시에 호출 가능
    public String get(String key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // 쓰기 — 한 스레드만 진입, 읽기도 전부 블로킹
    public void put(String key, String value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    // 삭제 — 쓰기 락 사용
    public void remove(String key) {
        writeLock.lock();
        try {
            cache.remove(key);
        } finally {
            writeLock.unlock();
        }
    }

    // 전체 조회 — 읽기 락 사용
    public List<String> getAllKeys() {
        readLock.lock();
        try {
            return new ArrayList<>(cache.keySet());
        } finally {
            readLock.unlock();
        }
    }
}
```

#### Kotlin에서는

```kotlin
import java.util.concurrent.locks.ReentrantReadWriteLock
import kotlin.concurrent.read
import kotlin.concurrent.write

class ThreadSafeCache {
    private val cache = mutableMapOf<String, String>()
    private val rwLock = ReentrantReadWriteLock()

    // 읽기 — kotlin.concurrent.read 확장 함수 사용
    fun get(key: String): String? = rwLock.read {
        cache[key]
    }

    // 쓰기 — kotlin.concurrent.write 확장 함수 사용
    fun put(key: String, value: String) = rwLock.write {
        cache[key] = value
    }

    fun remove(key: String) = rwLock.write {
        cache.remove(key)
    }

    fun getAllKeys(): List<String> = rwLock.read {
        cache.keys.toList()
    }
}
```

> Kotlin 표준 라이브러리의 `kotlin.concurrent.read`와 `kotlin.concurrent.write` 확장 함수를 사용하면 try-finally 없이 간결하게 작성할 수 있습니다.

---

### 3-4. 실전 예시 — 설정 관리자

애플리케이션 설정을 런타임에 변경할 수 있는 상황입니다. 설정 읽기는 매 요청마다 발생하지만, 설정 변경은 관리자가 가끔 합니다.

```java
public class ConfigManager {
    private final Map<String, String> config = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    // 모든 HTTP 요청마다 호출됨 (초당 수천 번) → 읽기 락
    public String getConfig(String key) {
        rwLock.readLock().lock();
        try {
            return config.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    // 관리자가 설정 변경할 때만 호출됨 (하루 몇 번) → 쓰기 락
    public void updateConfig(String key, String value) {
        rwLock.writeLock().lock();
        try {
            config.put(key, value);
            System.out.println("설정 변경: " + key + " = " + value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    // 설정 전체 리로드 → 쓰기 락
    public void reloadAll(Map<String, String> newConfig) {
        rwLock.writeLock().lock();
        try {
            config.clear();
            config.putAll(newConfig);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

만약 여기서 synchronized를 쓰면 읽기 요청(초당 수천 번)이 서로 블로킹되어 심각한 병목이 됩니다. ReadWriteLock을 쓰면 읽기끼리는 동시에 처리되므로 성능이 크게 향상됩니다.

#### Kotlin에서는

```kotlin
import java.util.concurrent.locks.ReentrantReadWriteLock
import kotlin.concurrent.read
import kotlin.concurrent.write

class ConfigManager {
    private val config = mutableMapOf<String, String>()
    private val rwLock = ReentrantReadWriteLock()

    // 모든 HTTP 요청마다 호출됨 (초당 수천 번)
    fun getConfig(key: String): String? = rwLock.read {
        config[key]
    }

    // 관리자가 설정 변경할 때만 호출됨 (하루 몇 번)
    fun updateConfig(key: String, value: String) = rwLock.write {
        config[key] = value
        println("설정 변경: $key = $value")
    }

    // 설정 전체 리로드
    fun reloadAll(newConfig: Map<String, String>) = rwLock.write {
        config.clear()
        config.putAll(newConfig)
    }
}
```

---

### 3-5. 락 다운그레이드 (쓰기 → 읽기 전환)

쓰기 락을 잡은 상태에서 읽기 락으로 전환할 수 있습니다. 반대(읽기 → 쓰기)는 불가능합니다.

```java
public String getOrCompute(String key) {
    rwLock.readLock().lock();
    try {
        String value = cache.get(key);
        if (value != null) {
            return value; // 캐시 히트 → 읽기 락만으로 충분
        }
    } finally {
        rwLock.readLock().unlock();
    }

    // 캐시 미스 → 쓰기 락 획득
    rwLock.writeLock().lock();
    try {
        // 다른 스레드가 먼저 넣었을 수 있으므로 다시 확인
        String value = cache.get(key);
        if (value != null) {
            return value;
        }

        value = computeExpensiveValue(key);
        cache.put(key, value);

        // 쓰기 락을 잡은 상태에서 읽기 락 획득 (다운그레이드 준비)
        rwLock.readLock().lock();
        return value;
    } finally {
        rwLock.writeLock().unlock(); // 쓰기 락 해제 → 이제 읽기 락만 보유
        rwLock.readLock().unlock();  // 읽기 락도 해제
    }
}
```

#### Kotlin에서는

```kotlin
fun getOrCompute(key: String): String {
    // 먼저 읽기 락으로 시도
    rwLock.read {
        cache[key]?.let { return it }
    }

    // 캐시 미스 → 쓰기 락
    rwLock.write {
        // 다른 스레드가 먼저 넣었을 수 있으므로 다시 확인
        cache[key]?.let { return it }

        val value = computeExpensiveValue(key)
        cache[key] = value

        // 다운그레이드: 쓰기 락 안에서 읽기 락 획득
        rwLock.readLock().lock()
        // write 블록이 끝나면 쓰기 락 해제 → 읽기 락만 보유
    }

    try {
        return cache[key]!!
    } finally {
        rwLock.readLock().unlock()
    }
}
```

> 락 다운그레이드는 복잡하므로 꼭 필요한 경우가 아니면 단순히 쓰기 락 안에서 모든 작업을 마치는 것이 안전합니다.

---

### 3-6. 용처 정리

| 상황 | 적합 여부 |
|---|---|
| 읽기 90% + 쓰기 10% (설정 조회, 캐시) | **매우 적합** |
| 읽기 50% + 쓰기 50% | 효과 적음 (일반 ReentrantLock과 비슷) |
| 읽기 100% + 쓰기 0% | 락 자체가 필요 없음 (불변 객체 사용) |
| 쓰기가 대부분 | 오히려 오버헤드만 증가, ReentrantLock이 나음 |

### 3-7. 장단점

**장점**
- 읽기끼리 동시 진입이 가능하여 읽기 비율이 높을수록 성능 향상이 큽니다
- ReentrantLock의 기능(재진입, 공정성 설정)을 그대로 사용할 수 있습니다
- 락 다운그레이드(쓰기 → 읽기)를 지원합니다

**단점**
- 코드가 복잡합니다. 읽기 락과 쓰기 락을 구분해서 관리해야 합니다
- 쓰기 비율이 높으면 일반 락 대비 오버헤드만 증가합니다 (내부적으로 읽기/쓰기 상태를 관리하는 비용)
- 락 업그레이드(읽기 → 쓰기)가 불가능합니다. 읽기 락을 먼저 해제하고 쓰기 락을 다시 잡아야 합니다
- 쓰기 기아(writer starvation) 문제가 발생할 수 있습니다. 읽기 요청이 끊임없이 들어오면 쓰기 락이 계속 대기하게 됩니다. 이 경우 공정 모드(`new ReentrantReadWriteLock(true)`)로 완화할 수 있습니다

---

## 4. 전체 요약 — 언제 무엇을 쓸 것인가

| 상황 | 추천 |
|---|---|
| 단순한 임계영역 보호 | `synchronized` (Kotlin: `synchronized {}` 또는 `@Synchronized`) |
| tryLock, 타임아웃, 인터럽트, 다중 Condition 필요 | `ReentrantLock` (Kotlin: `lock.withLock {}`) |
| 읽기가 많고 쓰기가 가끔 발생 | `ReentrantReadWriteLock` (Kotlin: `rwLock.read {}` / `rwLock.write {}`) |
| 읽기만 존재, 쓰기 없음 | 락 불필요 (불변 객체, volatile 등) |
| 코루틴 환경 | `Mutex` 또는 `Channel` 사용 |
