# Wait/Notify: 조건 동기화의 이해

## 1. 들어가며: synchronized만으로는 부족하다

Java에서 `synchronized`를 배웠다면, 이것이 "한 번에 하나의 스레드만 진입할 수 있는 영역"을 만든다는 것을 알 것이다.

```java
synchronized (lock) {
    // 이 안에는 한 번에 하나의 스레드만 들어올 수 있다
}
```

이것을 **상호 배제(Mutual Exclusion)**라고 한다. 하지만 실제 프로그래밍에서는 이것만으로 해결할 수 없는 상황이 있다. 바로 **"특정 조건이 만족될 때까지 기다려야 하는 상황"**이다.

---

## 2. 문제 상황: 식당 주방

이해를 돕기 위해 식당 주방을 예로 들어보자.

### 등장인물
- **요리사**: 음식을 만들어서 카운터에 올려놓는다
- **서빙 직원**: 카운터에서 음식을 가져다가 손님에게 전달한다
- **카운터**: 음식을 올려놓는 공간 (한 번에 한 명만 접근 가능)

### 상황
서빙 직원이 카운터에 갔는데, 아직 음식이 준비되지 않았다면 어떻게 해야 할까?

```java
// 서빙 직원의 코드
synchronized (카운터) {
    if (카운터.비어있음()) {
        // 음식이 없다! 어떻게 하지?
    }
    음식 = 카운터.가져가기();
}
```

---

## 3. 잘못된 해결책들

### 해결책 1: 계속 확인하기 (Busy Waiting)

```java
synchronized (카운터) {
    while (카운터.비어있음()) {
        // 아무것도 안 하고 계속 루프만 돈다
    }
    음식 = 카운터.가져가기();
}
```

**문제점**: CPU가 아무 의미 없는 루프를 계속 돌린다. 컴퓨터 자원을 낭비하는 최악의 방법이다.

### 해결책 2: 락을 잡은 채로 기다리기

```java
synchronized (카운터) {
    while (카운터.비어있음()) {
        // 카운터 앞을 점령한 채로 기다린다
    }
    음식 = 카운터.가져가기();
}
```

**문제점**: 서빙 직원이 카운터를 점령하고 있으니까, 요리사가 음식을 올려놓을 수가 없다. 서빙 직원은 음식을 기다리고, 요리사는 카운터가 비기를 기다린다. **영원히 서로를 기다리는 교착 상태(Deadlock)**가 발생한다.

### 딜레마
- 락을 놓자니 → 다른 서빙 직원이 먼저 가져갈 수 있다
- 락을 잡고 있자니 → 요리사가 음식을 못 올려놓는다

이 딜레마를 어떻게 해결할까?

---

## 4. 해결책: Wait/Notify

Java는 이 문제를 해결하기 위해 `wait()`와 `notify()` 메서드를 제공한다.

### 핵심 아이디어

> "락을 잠시 놓고 대기실에서 기다려. 상황이 바뀌면 불러줄게."

### 동작 방식

**wait() 호출 시:**
1. 현재 가지고 있는 락을 놓는다 (다른 스레드가 쓸 수 있게)
2. 대기실(Wait Set)로 가서 잠든다 (CPU를 사용하지 않음)
3. 누군가 깨워줄 때까지 기다린다

**notify() 호출 시:**
1. 대기실에서 기다리는 스레드 중 하나를 깨운다
2. 깨어난 스레드는 다시 락을 얻기 위해 경쟁한다

### 코드로 보기

```java
// 서빙 직원
synchronized (카운터) {
    while (카운터.비어있음()) {
        카운터.wait();    // 락을 놓고 대기실에서 잠든다
    }
    음식 = 카운터.가져가기();
}

// 요리사
synchronized (카운터) {
    카운터.올려놓기(음식);
    카운터.notify();      // 대기실에 있는 직원을 깨운다
}
```

### 흐름을 따라가보자

1. 서빙 직원이 카운터에 도착 (락 획득)
2. 음식이 없네? → `wait()` 호출
3. 락을 놓고 대기실로 감 (이제 카운터가 비었다)
4. 요리사가 카운터에 도착 (락 획득)
5. 음식을 올려놓고 → `notify()` 호출
6. 서빙 직원이 깨어남
7. 요리사가 락을 놓으면, 서빙 직원이 다시 락을 잡고 음식을 가져감

---

## 5. 모니터 모델: 내부 구조 이해하기

Java의 모든 객체는 내부적으로 **모니터(Monitor)**라는 구조를 가지고 있다. 모니터는 두 개의 대기 공간을 관리한다.

```
┌─────────────────────────────────────────────┐
│                  모니터                       │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │  Entry Set   │    │   Wait Set   │       │
│  │  (입장 대기)  │    │  (조건 대기)  │       │
│  │              │    │              │       │
│  │  스레드 B    │    │  스레드 C     │       │
│  │  스레드 D    │    │  스레드 E     │       │
│  └──────┬───────┘    └───────▲──────┘       │
│         │                    │              │
│         │ 락 획득        wait() │              │
│         ▼                    │              │
│  ┌──────────────────────────┴───────┐      │
│  │        임계 영역 (Critical Section)  │      │
│  │           스레드 A (락 보유 중)       │      │
│  │                                   │      │
│  │  notify() → Wait Set에서 하나 깨움  │      │
│  └───────────────────────────────────┘      │
└─────────────────────────────────────────────┘
```

**Entry Set**: 락을 얻기 위해 기다리는 스레드들
**Wait Set**: 조건이 만족되기를 기다리는 스레드들

`wait()`을 호출하면 Entry Set이 아닌 Wait Set으로 간다. `notify()`가 호출되면 Wait Set에서 Entry Set으로 이동해서 다시 락 경쟁에 참여한다.

---

## 6. 중요한 개념: wait는 "락 하나"만 다룬다

여기서 많은 사람들이 헷갈리는 부분이 있다.

### 잘못된 이해

"락 A를 얻고, 락 B가 필요한데 못 얻으면 wait하고, 누가 락 B를 놓으면서 notify하면 깨어나는 건가요?"

**아니다.**

### 올바른 이해

`wait()`을 호출하면 **그 객체의 락만** 놓는다. 다른 락에 대해서는 아무것도 모른다.

```java
synchronized (lockA) {
    lockA.wait();    // lockA만 놓는다. lockB는 모르는 일이다.
}
```

### 왜 이게 중요한가?

만약 "자원 A도 필요하고 자원 B도 필요한" 상황에서 wait/notify로 여러 락을 직접 조율하려고 하면:

- 교착 상태 (Deadlock)
- 신호 유실 (Lost Wakeup)
- 락 순서 역전 (Lock Order Inversion)

같은 심각한 문제가 발생한다.

---

## 7. 여러 자원이 필요할 때: 올바른 접근법

그렇다면 여러 자원이 동시에 필요한 상황은 어떻게 처리해야 할까?

### 잘못된 방법

```java
// 이렇게 하면 안 된다
synchronized (자원A) {
    synchronized (자원B) {
        // 교착 상태의 위험이 있다
    }
}
```

### 올바른 방법

**여러 자원의 상태를 "하나의 조건"으로 추상화하고, 하나의 모니터 안에서 관리한다.**

```java
class 자원관리자 {
    private boolean 자원A_사용가능 = true;
    private boolean 자원B_사용가능 = true;

    public synchronized void 두자원_획득() throws InterruptedException {
        // 두 자원의 상태를 "하나의 조건"으로 검사
        while (!자원A_사용가능 || !자원B_사용가능) {
            wait();    // 둘 다 사용 가능할 때까지 기다림
        }
        자원A_사용가능 = false;
        자원B_사용가능 = false;
    }

    public synchronized void 두자원_반납() {
        자원A_사용가능 = true;
        자원B_사용가능 = true;
        notifyAll();    // 기다리는 스레드들에게 알림
    }
}
```

**핵심 원칙:**
- wait/notify는 "락 여러 개를 조율하는 도구"가 **아니다**
- 여러 자원이 필요하면 → 그 상태들을 하나로 묶어서 → 단일 모니터 안에서 조건으로 관리한다

---

## 8. 반드시 지켜야 할 규칙

### 규칙 1: synchronized 블록 안에서만 호출

```java
// 잘못된 코드 - IllegalMonitorStateException 발생
object.wait();

// 올바른 코드
synchronized (object) {
    object.wait();
}
```

**이유**: 락을 가지고 있어야 락을 놓을 수 있다. wait()는 락을 놓는 동작이므로, 먼저 락을 가지고 있어야 한다.

### 규칙 2: 조건 검사는 반드시 while문으로

```java
// 잘못된 코드
synchronized (object) {
    if (조건이_안_맞음) {
        object.wait();
    }
    // 작업 수행
}

// 올바른 코드
synchronized (object) {
    while (조건이_안_맞음) {
        object.wait();
    }
    // 작업 수행
}
```

**이유 (Spurious Wakeup 문제):**
1. OS나 JVM이 notify 없이도 스레드를 깨울 수 있다
2. notify 후 다른 스레드가 먼저 락을 잡아서 조건을 다시 바꿀 수 있다
3. 따라서 깨어난 후에는 항상 조건을 다시 확인해야 한다

### 규칙 3: notify vs notifyAll

| 상황 | 사용할 메서드 |
|------|--------------|
| 대기 중인 스레드들이 모두 같은 조건을 기다림 | `notify()` 사용 가능 |
| 대기 중인 스레드들이 서로 다른 조건을 기다림 | `notifyAll()` 사용 필수 |

확실하지 않다면 `notifyAll()`을 사용하는 것이 안전하다.

---

## 9. 실전 예제 모음

### 예제 1: 생산자-소비자 (Producer-Consumer)

가장 기본적인 패턴이다. 생산자는 데이터를 만들고, 소비자는 데이터를 가져간다.

**상황**: 요리사가 음식을 만들고, 서빙 직원이 가져가는 식당

```java
class SimpleBuffer {
    private int data;
    private boolean empty = true;

    // 생산자가 호출
    public synchronized void produce(int value) throws InterruptedException {
        while (!empty) {
            wait();    // 버퍼가 비워질 때까지 대기
        }
        data = value;
        empty = false;
        System.out.println("생산: " + value);
        notify();
    }

    // 소비자가 호출
    public synchronized int consume() throws InterruptedException {
        while (empty) {
            wait();    // 데이터가 들어올 때까지 대기
        }
        empty = true;
        System.out.println("소비: " + data);
        notify();
        return data;
    }
}
```

---

### 예제 2: 바운디드 버퍼 (Bounded Buffer)

크기가 제한된 버퍼. 버퍼가 가득 차면 생산자가 대기하고, 버퍼가 비면 소비자가 대기한다.

**상황**: 10개의 접시만 올려놓을 수 있는 카운터

```java
class BoundedBuffer {
    private int[] buffer;
    private int count = 0;      // 현재 들어있는 개수
    private int putIndex = 0;   // 다음에 넣을 위치
    private int takeIndex = 0;  // 다음에 뺄 위치

    public BoundedBuffer(int size) {
        buffer = new int[size];
    }

    public synchronized void put(int value) throws InterruptedException {
        // 버퍼가 가득 찼으면 대기
        while (count == buffer.length) {
            System.out.println("버퍼 가득 참. 생산자 대기...");
            wait();
        }

        buffer[putIndex] = value;
        putIndex = (putIndex + 1) % buffer.length;
        count++;

        System.out.println("넣음: " + value + " (현재 " + count + "개)");
        notifyAll();    // 소비자를 깨움
    }

    public synchronized int take() throws InterruptedException {
        // 버퍼가 비었으면 대기
        while (count == 0) {
            System.out.println("버퍼 비어있음. 소비자 대기...");
            wait();
        }

        int value = buffer[takeIndex];
        takeIndex = (takeIndex + 1) % buffer.length;
        count--;

        System.out.println("꺼냄: " + value + " (현재 " + count + "개)");
        notifyAll();    // 생산자를 깨움
        return value;
    }
}
```

**포인트**: `notifyAll()`을 쓰는 이유는 생산자와 소비자가 같은 Wait Set에서 기다리기 때문이다. `notify()`를 쓰면 소비자가 소비자를 깨우는 상황이 생길 수 있다.

---

### 예제 3: 스레드 풀 (Thread Pool)

일정 수의 워커 스레드가 작업 큐에서 작업을 꺼내 처리한다. 작업이 없으면 대기한다.

**상황**: 콜센터 직원들이 전화 대기하다가, 전화가 오면 받는 것

```java
class SimpleThreadPool {
    private final List<Runnable> taskQueue = new ArrayList<>();
    private boolean shutdown = false;

    public SimpleThreadPool(int numThreads) {
        // 워커 스레드들 생성
        for (int i = 0; i < numThreads; i++) {
            new Thread(new Worker(), "Worker-" + i).start();
        }
    }

    // 작업 제출
    public synchronized void submit(Runnable task) {
        if (shutdown) {
            throw new IllegalStateException("ThreadPool이 종료됨");
        }
        taskQueue.add(task);
        notify();    // 대기 중인 워커 하나를 깨움
    }

    // 워커 스레드의 동작
    private class Worker implements Runnable {
        @Override
        public void run() {
            while (true) {
                Runnable task;

                synchronized (SimpleThreadPool.this) {
                    // 작업이 없으면 대기
                    while (taskQueue.isEmpty() && !shutdown) {
                        try {
                            System.out.println(Thread.currentThread().getName() + " 대기 중...");
                            SimpleThreadPool.this.wait();
                        } catch (InterruptedException e) {
                            return;
                        }
                    }

                    if (shutdown && taskQueue.isEmpty()) {
                        return;    // 종료
                    }

                    task = taskQueue.remove(0);
                }

                // 락을 놓고 작업 실행 (다른 스레드가 큐에 접근 가능)
                System.out.println(Thread.currentThread().getName() + " 작업 실행");
                task.run();
            }
        }
    }

    public synchronized void shutdown() {
        shutdown = true;
        notifyAll();    // 모든 워커를 깨워서 종료하게 함
    }
}
```

**포인트**: 작업을 실행할 때는 락을 놓는다. 그래야 다른 워커가 큐에서 다음 작업을 가져갈 수 있다.

---

### 예제 4: 게이트/래치 (Gate/Latch)

특정 조건이 만족될 때까지 여러 스레드를 막아두었다가, 조건이 만족되면 한꺼번에 통과시킨다.

**상황**: 경마장에서 모든 말이 준비되면 출발 신호를 보내는 것

```java
class StartingGate {
    private boolean opened = false;

    // 말(스레드)이 게이트 앞에서 대기
    public synchronized void await() throws InterruptedException {
        while (!opened) {
            System.out.println(Thread.currentThread().getName() + " 게이트 앞에서 대기");
            wait();
        }
        System.out.println(Thread.currentThread().getName() + " 출발!");
    }

    // 게이트 열기 (모든 말이 동시에 출발)
    public synchronized void open() {
        System.out.println("=== 게이트 오픈! ===");
        opened = true;
        notifyAll();    // 대기 중인 모든 스레드를 깨움
    }
}

// 사용 예
class Horse implements Runnable {
    private StartingGate gate;
    private String name;

    public Horse(StartingGate gate, String name) {
        this.gate = gate;
        this.name = name;
    }

    @Override
    public void run() {
        try {
            Thread.currentThread().setName(name);
            gate.await();    // 게이트가 열릴 때까지 대기
            // 달리기 시작...
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**포인트**: `notifyAll()`을 써서 모든 스레드를 동시에 깨운다. 한 번 열린 게이트는 계속 열려있으므로, 나중에 오는 스레드도 바로 통과한다.

---

### 예제 5: 순서 보장 실행 (Ordered Execution)

여러 스레드가 특정 순서대로 실행되어야 할 때 사용한다.

**상황**: 세 명의 직원이 A → B → C 순서로 보고해야 하는 것

```java
class OrderedPrinter {
    private int currentTurn = 1;    // 현재 누구 차례인가
    private final int totalThreads = 3;

    public synchronized void print(int threadNumber, String message)
            throws InterruptedException {
        // 내 차례가 아니면 대기
        while (currentTurn != threadNumber) {
            wait();
        }

        // 내 차례다! 출력
        System.out.println("스레드 " + threadNumber + ": " + message);

        // 다음 사람 차례로
        currentTurn = (currentTurn % totalThreads) + 1;
        notifyAll();    // 다음 차례인 스레드가 누군지 모르니까 모두 깨움
    }
}

// 사용 예
class OrderedWorker implements Runnable {
    private OrderedPrinter printer;
    private int myNumber;

    public OrderedWorker(OrderedPrinter printer, int myNumber) {
        this.printer = printer;
        this.myNumber = myNumber;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 3; i++) {
                printer.print(myNumber, "보고 #" + (i + 1));
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// 결과:
// 스레드 1: 보고 #1
// 스레드 2: 보고 #1
// 스레드 3: 보고 #1
// 스레드 1: 보고 #2
// 스레드 2: 보고 #2
// ... (항상 1 → 2 → 3 순서 보장)
```

**포인트**: 자기 차례가 아닌 스레드는 계속 대기한다. 순서가 돌아와야만 작업을 수행할 수 있다.

---

### 예제 6: 커넥션 풀 (Connection Pool)

제한된 개수의 자원(DB 커넥션 등)을 여러 스레드가 공유한다. 자원이 없으면 대기한다.

**상황**: 5대의 공용 자전거를 여러 사람이 빌려 쓰는 것

```java
class ConnectionPool {
    private final List<Connection> availableConnections = new ArrayList<>();
    private final int maxConnections;
    private int currentConnections = 0;

    public ConnectionPool(int maxConnections) {
        this.maxConnections = maxConnections;
    }

    // 커넥션 빌리기
    public synchronized Connection borrowConnection() throws InterruptedException {
        // 사용 가능한 커넥션이 없으면 대기
        while (availableConnections.isEmpty()) {
            // 아직 최대치 안 됐으면 새로 만들기
            if (currentConnections < maxConnections) {
                Connection newConn = createNewConnection();
                currentConnections++;
                System.out.println("새 커넥션 생성. 총 " + currentConnections + "개");
                return newConn;
            }

            // 최대치면 누가 반납할 때까지 대기
            System.out.println(Thread.currentThread().getName() + " 커넥션 대기 중...");
            wait();
        }

        Connection conn = availableConnections.remove(0);
        System.out.println(Thread.currentThread().getName() + " 커넥션 획득");
        return conn;
    }

    // 커넥션 반납하기
    public synchronized void returnConnection(Connection conn) {
        availableConnections.add(conn);
        System.out.println(Thread.currentThread().getName() + " 커넥션 반납");
        notify();    // 대기 중인 스레드 하나를 깨움
    }

    private Connection createNewConnection() {
        // 실제로는 DB 연결 생성
        return new Connection();
    }
}

class Connection {
    // 가상의 커넥션 클래스
    public void query(String sql) {
        System.out.println("쿼리 실행: " + sql);
    }
}
```

**포인트**: 자원을 최대치까지 만들고, 그 이후에는 누군가 반납할 때까지 대기한다. 이것이 풀(Pool)의 기본 원리다.

---

## 10. 정리

### synchronized vs wait/notify

| 개념 | 역할 | 비유 |
|------|------|------|
| `synchronized` | 상호 배제 | 화장실 문 잠금 |
| `wait/notify` | 조건 동기화 | 휴지가 없어서 밖에서 기다림 |

### wait/notify의 본질

1. **목적**: 특정 조건이 만족될 때까지 효율적으로 기다리는 것
2. **동작**: 락을 놓고 잠들었다가, 누군가 깨워주면 다시 락을 잡고 진행
3. **범위**: 하나의 모니터(락) 안에서만 동작
4. **주의**: 여러 락을 조율하는 도구가 아님. 여러 자원이 필요하면 하나의 조건으로 추상화해야 함

### 핵심 규칙

1. `synchronized` 블록 안에서만 호출
2. 조건 검사는 반드시 `while`문으로
3. 불확실하면 `notifyAll()` 사용

### 언제 어떤 패턴을 쓰나

| 상황 | 패턴 |
|------|------|
| 데이터 전달 (1:1) | 생산자-소비자 |
| 데이터 전달 (버퍼 있음) | 바운디드 버퍼 |
| 작업 분배 | 스레드 풀 |
| 동시 시작 | 게이트/래치 |
| 순서 제어 | 순서 보장 실행 |
| 자원 공유 | 커넥션 풀 |

---

## 참고: 더 현대적인 대안

Java 5부터는 `java.util.concurrent` 패키지에서 더 강력하고 유연한 도구들을 제공한다.

- `Lock`과 `Condition`: wait/notify의 개선된 버전
- `BlockingQueue`: 생산자-소비자 패턴을 쉽게 구현
- `Semaphore`, `CountDownLatch`, `CyclicBarrier`: 다양한 동기화 패턴
- `ExecutorService`: 스레드 풀의 표준 구현

하지만 wait/notify를 이해하는 것은 이런 고수준 도구들의 동작 원리를 이해하는 데 필수적인 기초가 된다.
