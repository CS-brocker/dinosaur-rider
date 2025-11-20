# GCD (Grand Central Dispatch)
커널 수준의 스케줄링을 이용(libdispatch 라이브러리)해서 스레드 관리를 자동화한 시스템

개발자가 직접 스레드를 제어하기 위해서 쓰는 API 같은 것

큐에다가 클로저로 작업을 넣는 형태

Swift는 기본 메인스레드에서 돌기 때문에 메인스레드에서 돌릴 필요 없는 경우 개발자가 백그라운드로 지정하도록 유도함

### Queue?

```swift
// 큐의 종류
// 1. 메인 스레드
DispatchQueue.main.async {
  // UI 업데이트, I/O 관련 작업 제어 등
  // ex. 포인트를 획득했는데 그걸 화면에 보여줘야 한다? 포인트 획득같은 api 호출은 백그라운드에서, 완료 후 UI 업데이트는 여기서
}

// 2. 시스템이 관리하는 백그라운드 스레드 풀
DispatchQueue.global().async {
  // 굳이 메인스레드에서 하지 않아도 되는 작업들
  // ex. 이미지 다운로드같은걸 메인스레드에서 하면(메인스레드 점유) 돌아가는 동안 화면이 멈추게 됨.. 
}

// 3. 커스텀 큐
DispatchQueue(label: "myQueue").async {
  // 굳이 메인스레드에서 하지 않아도 되는 작업들
  // ex. 이미지 다운로드같은걸 메인스레드에서 하면(메인스레드 점유) 돌아가는 동안 화면이 멈추게 됨.. 
}
```

### sync? async?

```swift
func asyncFucntion() { // async는 기다리지 않기 때문에 a, b, c의 순차 출력을 기대할 수 없음
  print("a")
    DispatchQueue.global().async {
        print("b")
    }
    print("c")
}

func asyncFucntion() { // sync는 기다리기 때문에 a, b, c의 순차 출력을 기대할 수 있음
  print("a")
    DispatchQueue.global().sync {
        print("b")
    }
    print("c")
}
```

### QoS (Quality Of Service)

![image.png](attachment:9152a3a8-e46b-46ad-947f-26546addb403:image.png)

```swift
DispatchQueue.global(qos: .background).async {
    cleanCache() // OS가 cpu 자원을 천천히 줌
}
```
