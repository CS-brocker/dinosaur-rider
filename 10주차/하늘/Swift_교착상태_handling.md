# Swift 교착상태 handling

## iOS에서 자주 발생하는 문제

- Deadlock: 아래에서 예시로 설명
- Priority Inversion
    - (단일 코어라고 가정) low, medium, high 스레드가 있을 때
    - low가 락을 잡고 있을 때 high가 락을 요청하면 이미 사용중이니까 block 처리됨
    - 이 때 medium이 요청해서 실행하게 되면 low보다 우선순위가 높으니 바로 실행됨
    - **medium이 실행되는 동안 high는 계속 락이 해제되길 기다림 (highr가 우선순위가 더 높지만 medium을 기다리게 됨)**
    
    → 해결: **Priority Inheritance (low의 우선순위를 높여서 해결)**
    

## 전통 방식

- NSLock
    
    ```json
    let lock = NSLock()
    
    lock.lock()
    shared += 1
    lock.unlock()
    ```
    
    Priority Inversion 가능
    
- OSSpinLock
    - busy waiting 으로 인한 priority inversion 때문에 deprecated 되었음
- os_unfair_lock
    
    ```json
    var lock = os_unfair_lock()
    
    os_unfair_lock_lock(&lock)
    shared += 1
    os_unfair_lock_unlock(&lock)
    ```
    
    애플에서 권장하는 락 (OSSpinLock 제거하면서 이거 사용 권장)
    
    먼저부터 기다린 스레드가 아니라 스케줄러가 깨운 스레드가 먼저 락을 얻을 수 있음
    
    우선순위가 없어서 priority inversion 문제 !일부! 개선할 수 있지만(CPU가 선택하기 때문에 선점은 아니라서) starvation 발생 가능
    
- pthread_mutex
    
    ```json
    var mutex = pthread_mutex_t()
    
    pthread_mutex_init(&mutex, nil)
    
    pthread_mutex_lock(&mutex)
    shared += 1
    pthread_mutex_unlock(&mutex)
    ```
    
    POSIX에서 제공하는 전통적인 mutex
    
    priority inheritance 기능 제공하지만 Swift에서 사용하기 불편
    
- DispatchQueue.sync (GCD)
    
    ```swift
    let queue = DispatchQueue(label: "counter.queue")
    queue.sync { 
      shared += 1
    }
    ```
    
    ```swift
    let queue = DispatchQueue(label: "counter.queue")
    queue.sync { // deadlock🚨
            queue.sync { 
              shared += 1
            }
    }
    ```
    
    - Queue ≠ Thread (worker thread = libdispatch가 작업을 실행하기 위해 사용하는 실제 OS 스레드)
        - *libdispatch: Apple이 만든 작업 스케줄러
    - FIFO으로 실행
    - priority inheritance 할 수 잇음 (DispatchQueue.global(qos: .userInitiated))
    
    1. **queue에 enqueue**
    2. (스케줄링) libdispatch가 탐색하다가 queue 채워졌으면 worker thread 할당하여 실행
    3. (실행) worker thread가 pop → 실행
    

### Swift Concurrency 방식 → 다음주에 파헤치기

- Task
- Actor
- Sendable: 다른 스레드로 안전하게 전달 가능한 타입
