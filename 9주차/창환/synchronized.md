# Synchronized
자바와 코틀린의 Synchronized 문법을 각각 정리하였다.


## 🤼‍♀️기본적인 synchronized 개념(코틀린이나 자바나 마찬가지)
- 객체 인스턴스 자체(혹은 객체 인스턴스)를 물고들어가는 synchronized
- 특정 객체를 물고들어가는 synchronized
- static/companion -> 전역적으로 물고들어가는 synchronized
  
결론적으로 정리하면 이렇게 3개가 있다.  
각각 언어별로 좀더 자세히 봐보자

## 🤮JAVA synchronized🤮

### 1️⃣ 객체 인스턴스 자체(혹은 객체 인스턴스)를 물고들어가는 synchronized

- synchronized 메서드

아예 함수자체에 synchronized를 명시할수 있는데 이는 해당 메서드의 객체 인스턴스의 락을 쥐고 들어가는것이다.
```java
public synchronized void foo() { }
```

- synchronized 블록
  
블록으로도 동일조건을 구성할수 있다.
```java
public void foo() {
    synchronized (this) {
        // ...
    }
}
```

### 2️⃣ 특정 객체를 물고들어가는 synchronized

- synchronized 블록
어떤 인스턴스가 락의 제물이 될건지 정할수 있다.
```java
synchronized (lock) {
    // 임계 구역
}
```

### 3️⃣ static/companion -> 전역적으로 물고들어가는 synchronized

- static synchronized
스태틱 즉 객체 클래스 자체에 걸리는 락을 걸수 있다.
```java
public static synchronized void foo() {}
```

근데 결국 이것도 문법만 다를뿐 동일한 내용이다 
```java
synchronized (MyClass.class) {
}
```


## ❤️Kotlin synchronized❤️

### 1️⃣ 객체 인스턴스 자체(혹은 객체 인스턴스)를 물고들어가는 synchronized

- synchronized 어노테이션

어노테이션으로 synchronized를 명시할수 있는데 이는 해당 메서드를 포함하는 객체의 락을 쥐고 들어가는것이다.
```kotlin
@Synchronized
fun foo() {}
```

-> 자바의 synchronized 메서드와 대치된다고 보면된다.


### 2️⃣ 특정 객체를 물고들어가는 synchronized

- synchronized 블록
자바와 동일하다, 어떤 인스턴스가 락의 제물이 될건지 정할수 있다.
```kotlin
synchronized (lock) {
    // 임계 구역
}
```

### 3️⃣ static/companion -> 전역적으로 물고들어가는 synchronized

- companion object + @Synchronized
자바의 static Synchronized 와 동일하다 보면된다.
```kotlin
companion object {
    @Synchronized
    fun increase() {
        total++
    }
}
```

### 🐗코틀린이라 있는 추가사항: 최상위함수에 걸리는 synchronized 어노테이션

```kotlin
@Synchronized
fun foo() { }
```

>코틀린 최상위 함수에 @Synchronized를 붙이면
“그 함수가 들어있는 파일 클래스(File class)의 Class 객체 락”을 잡습니다.

저거를 자바로 디코딩하면 이렇게 나온다고한다.
```java
public final class UtilsKt {
    public static synchronized void foo() {
        System.out.println("hello");
    }
}
```
즉 파일이 클래스화 되는건 익히아는 사실이고 그 클래스의 스태틱 즉 클래스 단위에 붙는 락이 걸린다.
