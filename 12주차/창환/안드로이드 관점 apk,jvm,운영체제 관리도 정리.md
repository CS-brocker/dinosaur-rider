# 안드로이드 관점 APK, JVM, 운영체제 관리도 정리

## 먼저 큰 그림

안드로이드 앱을 이해할 때 헷갈리는 이유는 **APK**, **Android Framework**, **ART/JVM**, **운영체제 커널**이 서로 다른 층에 있기 때문이다.

한 줄로 정리하면 이렇다.

> APK는 앱을 배포하기 위한 압축 패키지이고, 앱이 실행되면 안드로이드 OS가 앱 프로세스를 만들고, 그 프로세스 안의 ART 런타임이 APK 안의 dex 코드를 실행한다.

대략적인 관계는 다음과 같다.

```text
앱 개발자가 작성
├─ Kotlin/Java 코드
├─ res/ 리소스
├─ assets/
├─ AndroidManifest.xml
└─ Gradle 의존성
        ↓ 빌드
APK 파일
├─ classes.dex                  앱 코드 + 포함된 라이브러리 코드
├─ res/                         컴파일된 리소스
├─ resources.arsc               리소스 테이블
├─ AndroidManifest.xml          컴파일된 매니페스트
├─ assets/                      원본에 가까운 파일
├─ lib/{abi}/*.so               네이티브 라이브러리
└─ META-INF/ 또는 서명 관련 정보
        ↓ 설치
기기 저장소에 APK 저장
        ↓ 실행
Zygote가 앱 프로세스 fork
        ↓
앱 프로세스 안에서 ART가 dex 코드 실행
        ↓
Linux 커널이 프로세스, 메모리, 파일, 스케줄링 관리
```

## APK에는 무엇이 들어가나?

APK(Android Package)는 사실상 **zip 형식의 앱 패키지 파일**이다. Windows의 `.exe`처럼 "앱을 실행하기 위한 대표 파일"이라고 비유할 수는 있지만, 정확히는 exe보다 **설치 가능한 압축 패키지**에 더 가깝다.

APK 안에는 보통 다음이 들어간다.

| 구성 요소 | 의미 |
|---|---|
| `classes.dex` | Kotlin/Java 바이트코드가 Android Runtime용 DEX 형식으로 변환된 것 |
| `res/` | XML 레이아웃, drawable, string 등 컴파일된 리소스 |
| `resources.arsc` | 리소스 ID와 실제 리소스를 연결하는 테이블 |
| `AndroidManifest.xml` | 앱 컴포넌트, 권한, 실행 정보 |
| `assets/` | 개발자가 넣은 원본 파일에 가까운 정적 파일 |
| `lib/arm64-v8a/*.so` 등 | C/C++로 작성된 네이티브 라이브러리 |
| 서명 정보 | 이 APK가 누구에 의해 서명되었는지 검증하기 위한 정보 |

즉 APK는 코드만 담는 파일이 아니다. **코드, 리소스, 설정, 네이티브 라이브러리, 서명 정보**가 함께 들어간다.

## DEX 형식이란 무엇인가?

DEX는 **Dalvik Executable**의 줄임말이다. 이름에는 Dalvik이 들어가지만, 현재 안드로이드의 ART도 앱 코드를 기본적으로 DEX 형식으로 받는다.

일반 Java 세계에서는 Java/Kotlin 코드가 보통 `.class` 파일로 컴파일된다. `.class` 파일은 JVM이 읽는 바이트코드 파일이다.

안드로이드는 이 `.class` 파일들을 그대로 APK에 넣어서 실행하지 않고, 여러 `.class` 파일을 모아서 **안드로이드 런타임이 읽기 좋은 하나의 형식**으로 다시 바꾼다. 그 결과물이 `.dex` 파일이다.

```text
Kotlin/Java 소스 코드
        ↓ 컴파일
.class 파일들
        ↓ DEX 변환
classes.dex
        ↓ APK에 포함
앱 실행 시 ART가 읽고 실행
```

여기서 "DEX 형식"이라는 말은, 코드가 다음과 같은 안드로이드 전용 바이트코드 파일 구조로 바뀌었다는 뜻이다.

- JVM용 `.class` 여러 개를 한 파일 안에 모은다.
- 중복되는 문자열, 타입, 메서드 참조 등을 공통 테이블로 정리한다.
- Android Runtime이 검증하고 로딩하기 좋은 명령어 형식으로 바꾼다.
- 모바일 환경에 맞게 메모리를 덜 쓰고 빠르게 읽을 수 있도록 구성한다.

비유하면 이렇다.

```text
.class
→ 일반 JVM용 부품 파일 여러 개

.dex
→ 안드로이드 런타임용으로 다시 포장한 실행 코드 묶음
```

그래서 `classes.dex`는 사람이 작성한 Kotlin/Java 코드 그 자체도 아니고, CPU가 바로 실행하는 기계어도 아니다. **ART가 이해하는 중간 코드**다.

다만 최신 안드로이드는 실행 성능을 위해 설치 시점이나 실행 중에 DEX를 더 최적화해서 OAT, VDEX, ART 내부 캐시 같은 형태로 다룰 수 있다. 그래도 앱 개발자가 APK 안에서 직접 보는 기본 코드 단위는 여전히 `classes.dex`라고 보면 된다.

## APK 안의 `.so` 네이티브 라이브러리는 무엇인가?

APK 안의 `lib/arm64-v8a/*.so` 같은 파일은 보통 **앱이 직접 포함해서 배포하는 C/C++ 네이티브 라이브러리**다.

이건 `View`, `Activity`, `TextView` 같은 Android Framework 제공 코드와는 다르다. Android Framework의 핵심 구현은 기기 OS 쪽에 있고, 앱 APK 안의 `.so`는 보통 앱이나 앱이 쓰는 외부 라이브러리가 필요해서 같이 넣은 네이티브 코드다.

```text
Android Framework 코드
→ 기기 OS 쪽에 있음
→ Activity, View, Context 같은 기본 API 제공

APK 안의 lib/arm64-v8a/*.so
→ 앱이 함께 들고 가는 native 라이브러리
→ C/C++로 작성된 기능을 앱 프로세스 안에서 실행
```

예시는 이런 것들이 있다.

- C/C++로 직접 작성한 성능 민감 코드
- 이미지 처리 라이브러리
- 영상/음성 코덱
- 암호화/보안 라이브러리
- 게임 엔진 코드(Unity, Unreal 등)
- 머신러닝 추론 엔진
- 특정 SDK가 내부적으로 사용하는 native 엔진
- JNI로 Kotlin/Java 코드와 연결되는 C/C++ 코드

예를 들어 Kotlin 코드에서 다음처럼 native 함수를 선언할 수 있다.

```kotlin
external fun processImage(input: ByteArray): ByteArray
```

그리고 C/C++ 쪽에서 실제 구현을 만든 뒤 `.so`로 빌드해서 APK에 넣는다. 앱 실행 중에는 ART가 Kotlin/Java 코드를 실행하다가, 필요한 순간 JNI(Java Native Interface)를 통해 `.so` 안의 native 함수를 호출한다.

```text
Kotlin/Java 코드
        ↓ JNI 호출
APK 안의 .so native 라이브러리
        ↓ 필요하면
Linux 커널 시스템 콜 또는 Android native API 사용
```

여기서 중요한 점은 `.so`가 "시스템콜만 돌리는 코드"는 아니라는 것이다. 그냥 **CPU가 직접 실행할 수 있는 네이티브 기계어 라이브러리**다. 그 안에서 계산만 할 수도 있고, 파일을 열 수도 있고, 네트워크나 그래픽 관련 native API를 호출할 수도 있고, 필요하면 최종적으로 OS 커널의 시스템 콜까지 내려갈 수도 있다.

또 `arm64-v8a`는 CPU 종류를 뜻한다. 스마트폰 CPU 아키텍처에 맞는 native 바이너리를 넣어야 하기 때문에 ABI별 폴더가 나뉜다.

```text
lib/arm64-v8a/*.so      64비트 ARM 기기용
lib/armeabi-v7a/*.so    32비트 ARM 기기용
lib/x86_64/*.so         64비트 x86 에뮬레이터/기기용
```

정리하면, APK 안의 `.so`는 **Android Framework 자체가 아니라 앱에 포함된 native 코드**다. Android 앱은 보통 Kotlin/Java DEX 코드로 동작하지만, 성능이나 기존 C/C++ 라이브러리 재사용이 필요할 때 `.so`를 함께 싣고 간다.

## Android Framework 코드는 어디에 있나?

안드로이드 스튜디오에서 `View`, `Activity`, `Context`, `TextView` 같은 클래스를 쓰면 이런 의문이 생긴다.

> 이 프레임워크 코드도 내 APK 안에 같이 들어가나?

대부분은 **아니다.**

`android.view.View`, `android.app.Activity` 같은 Android Framework 클래스들은 기본적으로 **기기 안의 Android OS 쪽에 이미 들어있다.** 앱은 이 프레임워크 API를 호출해서 OS가 제공하는 기능을 사용하는 것이다.

개발할 때는 Android SDK 안에 있는 `android.jar`를 보고 컴파일한다. 하지만 이 `android.jar`는 보통 실제 구현을 앱에 넣기 위한 라이브러리가 아니라, "이런 API가 있다"는 것을 컴파일러에게 알려주는 **API 껍데기/stub**에 가깝다.

실제 실행 시에는 기기에 설치된 Android Framework 구현을 사용한다.

```text
개발 시점
앱 코드 → Android SDK의 android.jar를 보고 컴파일

실행 시점
앱 프로세스 → 기기 OS에 있는 Android Framework 실제 구현 사용
```

그래서 `Activity`, `View`, `Context` 같은 기본 프레임워크 코드는 앱마다 APK에 한 벌씩 들어가는 구조가 아니다. OS 쪽에 있고, 앱들은 그 공통 프레임워크를 사용한다.

## Gradle 라이브러리는 APK에 포함되나?

Retrofit, OkHttp, Gson, Coil 같은 일반 Gradle 의존성은 보통 **앱 APK 안에 포함된다.**

빌드할 때 Gradle과 Android Gradle Plugin이 의존성을 모으고, Kotlin/Java 바이트코드를 DEX로 변환해서 `classes.dex` 또는 여러 dex 파일에 합친다.

```text
내 앱 코드
+ Retrofit 코드
+ OkHttp 코드
+ Gson 코드
        ↓
DEX 변환
        ↓
APK 안의 classes.dex
```

따라서 Retrofit을 쓰는 앱이 3개 설치되어 있다면, 일반적으로 각 APK 안에 Retrofit 코드가 각각 들어간다. 버전이 같더라도 앱 A의 APK, 앱 B의 APK, 앱 C의 APK에 각각 포함되는 식이다.

물론 예외는 있다.

- Android Framework처럼 OS가 기본 제공하는 API는 APK에 넣지 않는다.
- `compileOnly`처럼 컴파일 때만 쓰고 런타임에는 외부에서 제공된다고 가정하는 의존성은 APK에 포함되지 않을 수 있다.
- Play Feature Delivery, Dynamic Feature, split APK처럼 앱 배포 방식에 따라 APK가 여러 조각으로 나뉠 수 있다.
- 네이티브 `.so` 라이브러리는 `lib/` 아래에 ABI별로 들어갈 수 있다.

하지만 일반적인 앱 개발 관점에서는 **Gradle implementation 의존성은 앱 패키지 안으로 들어간다**고 보면 된다.

## APK는 exe와 같은가?

비유로는 어느 정도 맞지만 완전히 같지는 않다.

Windows의 `.exe`는 그 자체가 실행 가능한 바이너리 파일이다. 반면 APK는 안드로이드가 설치하고 실행하기 위한 **패키지 파일**이다. APK 안의 `classes.dex`가 앱 코드이고, 이 코드를 Android Runtime이 읽고 실행한다.

그래서 더 정확한 비유는 이렇다.

```text
Windows exe
→ 실행 가능한 프로그램 파일

Android APK
→ 앱 코드, 리소스, 설정, 서명을 담은 설치 패키지
→ 실행은 Android OS + ART 런타임이 담당
```

## JVM은 어디에 있나?

전통적인 Java에서는 JVM(Java Virtual Machine)이 `.class` 바이트코드를 실행한다.

하지만 안드로이드는 일반적인 데스크탑 JVM을 그대로 쓰지 않는다. 안드로이드는 원래 **Dalvik**이라는 런타임을 썼고, 현재는 **ART(Android Runtime)** 를 쓴다.

즉 요즘 안드로이드에서 정확한 표현은 JVM보다 **ART**가 맞다.

여기서 중요한 점은 **Dalvik은 운영체제 자체가 아니라는 것**이다. Dalvik은 Android OS 전체가 아니라, Android OS 안에서 앱 바이트코드를 실행하던 **런타임/가상머신 계층**이었다.

```text
Android OS
├─ Linux Kernel
├─ Android Framework
├─ System Services
├─ Native Libraries
└─ Runtime
   ├─ 예전: Dalvik
   └─ 현재: ART
```

즉 "안드로이드 = 달빅"은 정확하지 않다. 더 정확히는 **예전 안드로이드는 앱 실행 런타임으로 Dalvik을 사용했고, 지금 안드로이드는 ART를 사용한다**이다.

```text
일반 Java
Java/Kotlin 코드 → .class → JVM이 실행

Android
Kotlin/Java 코드 → .class → .dex → ART가 실행
```

그래서 대응 관계를 잡으면 다음처럼 보면 된다.

```text
일반 Java 세계: JVM이 .class 바이트코드 실행
Android 세계: ART가 .dex 바이트코드 실행
```

다만 ART는 JVM과 완전히 같은 구현은 아니다. Android에 맞게 DEX, 앱 샌드박스, Zygote, AOT/JIT 최적화, Android Framework와 엮여 있는 전용 런타임이다.

ART는 기기 OS의 일부로 제공된다. 앱마다 APK 안에 ART가 들어가는 것이 아니다. 앱이 실행되면 앱 프로세스 안에 ART 런타임이 함께 올라와서 dex 코드를 실행한다고 보면 된다.

중요한 점은, ART가 "앱 바깥에 따로 떠 있는 하나의 JVM 서버 프로세스"처럼 모든 앱을 대신 실행하는 구조가 아니라는 것이다. 안드로이드 앱은 보통 **각 앱마다 Linux 프로세스**를 가지고, 그 프로세스 안에서 ART 런타임이 앱 코드를 실행한다.

## 앱 실행 과정

앱 아이콘을 눌렀을 때 아주 단순화하면 다음 일이 벌어진다.

```text
1. 사용자가 앱 실행
2. Android Framework의 ActivityManagerService가 앱 실행 결정
3. 아직 앱 프로세스가 없으면 Zygote에게 fork 요청
4. Zygote가 새 앱 프로세스를 만든다
5. 앱 프로세스 안에서 ART가 초기화된다
6. APK의 dex 코드와 리소스를 로드한다
7. Application, Activity 객체가 생성된다
8. 앱 코드가 실행된다
```

여기서 **Zygote**가 중요하다. Zygote는 부팅 때 미리 만들어져 있는 부모 프로세스다. 안드로이드 프레임워크 클래스와 런타임에 필요한 것들을 어느 정도 미리 로드해 둔다.

새 앱을 실행할 때 매번 처음부터 모든 런타임과 프레임워크를 로드하면 느리다. 그래서 Zygote를 `fork()`해서 앱 프로세스를 빠르게 만든다. 이때 Copy-on-Write 덕분에 공통 프레임워크 메모리는 여러 앱이 상당 부분 공유할 수 있다.

## APK, RAM, 힙, 스택, 코드 영역 관계

APK는 설치 후 기기의 영구 저장소에 있다. 앱이 실행되면 필요한 코드와 리소스가 메모리로 매핑되거나 로드된다.

여기서 메모리는 크게 두 종류로 나눠볼 수 있다.

### 1. File-backed 메모리

디스크의 어떤 파일과 연결된 메모리다.

예를 들어 APK 안의 dex 코드, 리소스, OS 프레임워크 코드, `.so` 네이티브 라이브러리 같은 것은 원본 파일이 저장소에 있다.

이런 페이지는 메모리가 부족할 때 OS가 비교적 쉽게 버릴 수 있다. 왜냐하면 나중에 필요하면 원본 파일에서 다시 읽으면 되기 때문이다.

```text
APK / OS 파일에 원본 있음
        ↓
RAM에 매핑해서 사용
        ↓
메모리 부족하면 버릴 수 있음
        ↓
다시 필요하면 파일에서 재로딩
```

### 2. Anonymous 메모리

특정 파일에 원본이 없는 메모리다.

예를 들면 다음이 익명 메모리다.

- `malloc()`으로 잡은 native heap
- Kotlin/Java 객체들이 들어가는 managed heap
- 함수 호출에 쓰이는 thread stack
- 런타임 중 계산 결과로 만들어진 데이터

이것들은 APK 안에 원본이 없다. 앱 실행 중에 동적으로 생긴 값들이기 때문이다.

예를 들어 `User("changhwan")` 같은 Kotlin 객체를 만들면, 그 객체는 APK에 미리 들어있던 물건이 아니다. 코드가 실행되면서 힙에 새로 만들어진 데이터다. 그래서 메모리가 부족하다고 그냥 버리면 앱 상태가 깨진다.

```text
앱 코드 실행
        ↓
객체 생성, 함수 호출, malloc
        ↓
힙/스택에 동적 데이터 생성
        ↓
원본 파일이 없음
        ↓
그냥 버릴 수 없음
```

이게 기존 문서에서 말한 **익명 메모리**다.

## 그래서 OS는 메모리 부족 때 다르게 다룬다

OS 입장에서 file-backed 페이지와 anonymous 페이지는 성격이 다르다.

| 종류 | 예시 | 원본 파일 | 메모리 부족 시 |
|---|---|---|---|
| file-backed | dex 코드, 리소스, `.so`, 프레임워크 코드 | 있음 | 버렸다가 다시 읽을 수 있음 |
| anonymous | 힙 객체, 스택, malloc 메모리 | 없음 | 그냥 버리면 안 됨 |

데스크탑 OS라면 anonymous 페이지를 스왑 공간에 써두고 RAM에서 내보낼 수 있다.

하지만 모바일에서는 전통적인 저장소 기반 스왑을 적극적으로 쓰기 어렵다. 그래서 안드로이드는 zRAM 같은 압축 메모리, 캐시 회수, 백그라운드 앱 종료 같은 방식을 조합한다.

즉 기존 문단의 의미는 다음과 같다.

> APK 안의 dex 코드나 리소스는 원본 파일이 있으니 필요하면 다시 읽을 수 있다. 하지만 앱 실행 중 만들어진 Kotlin 객체, 힙, 스택 데이터는 원본 파일이 없으니 OS가 함부로 버릴 수 없다.

## Dalvik, ART, JVM, Android OS의 관계

안드로이드를 "달빅"이라고 기억하고 있다면 절반은 맞고, 최신 기준으로는 업데이트가 필요하다.

- **Dalvik**: 옛날 안드로이드 런타임. Android 4.x 시절까지 중심.
- **ART(Android Runtime)**: 현재 안드로이드 런타임. Android 5.0 이후 기본.
- **JVM**: 일반 Java 세계의 가상 머신 개념. 안드로이드는 표준 JVM 대신 Dalvik/ART 계열 런타임을 사용.
- **Android OS**: Linux 커널, 시스템 서비스, Android Framework, ART, 기본 라이브러리 등을 포함한 전체 운영체제.

안드로이드는 "전부 Java라서 JVM 없으면 아무것도 못 하는 OS"라기보다는, **Linux 기반 운영체제 위에 Java/Kotlin 앱 실행 환경과 Android Framework를 얹은 시스템**에 가깝다.

안드로이드 내부에는 C/C++로 작성된 부분도 많다. Linux 커널, 드라이버, Binder, SurfaceFlinger, media stack, ART 런타임 자체 등은 Java/Kotlin만으로 되어 있지 않다.

앱 개발자는 주로 Kotlin/Java와 Android Framework API를 쓰지만, 운영체제 전체는 훨씬 아래 계층까지 포함한다.

```text
앱 계층
Kotlin/Java 앱 코드, Compose/View, Retrofit 등

런타임/프레임워크 계층
ART, Android Framework, System Server, ActivityManagerService 등

네이티브 시스템 계층
Binder, libc, SurfaceFlinger, Media, SQLite 등

커널 계층
Linux kernel, 프로세스, 스케줄링, 가상 메모리, 파일 시스템, 드라이버
```

## 질문별 짧은 답

**Q. Android Framework 코드는 앱별 APK에 들어가나?**  
대부분 아니다. `Activity`, `View`, `Context` 같은 기본 프레임워크 구현은 기기 OS 쪽에 있다. 앱은 SDK의 API를 기준으로 컴파일하고, 실행 시 기기의 프레임워크를 사용한다.

**Q. Retrofit 같은 라이브러리는 앱마다 들어가나?**  
보통 그렇다. `implementation` 의존성은 앱 APK의 dex에 포함된다. Retrofit을 쓰는 앱이 3개면 일반적으로 각 APK가 Retrofit 코드를 가진다.

**Q. APK는 exe인가?**  
비슷하게 "앱 파일"이라고 생각할 수는 있지만, 정확히는 실행 파일이라기보다 설치 패키지다. 실행은 Android OS와 ART가 담당한다.

**Q. JVM은 폰 위에 깔려 있나?**  
일반 JVM이 아니라 ART가 Android OS의 일부로 있다. 앱 프로세스 안에서 ART가 dex 코드를 실행한다.

**Q. APK가 RAM 위의 JVM 프로세스에서 도는 건가?**  
앱은 하나의 Linux 프로세스로 실행된다. 그 프로세스 안에 ART 런타임, 앱 코드, 힙, 스택 등이 있다. APK 자체는 저장소에 있고, 필요한 코드와 리소스가 메모리에 매핑되거나 로드된다.

**Q. 코드 영역은 APK에서 끌고 오고, 객체는 동적으로 생기나?**  
큰 방향은 맞다. dex 코드와 리소스는 APK/설치 파일에서 온 file-backed 메모리로 볼 수 있고, 실행 중 만들어지는 객체, 힙, 스택은 anonymous 메모리다.

## 최종 정리

APK는 앱의 코드와 리소스를 담은 패키지다. Android Framework의 핵심 구현은 앱 APK에 들어가는 것이 아니라 기기 OS 쪽에 있다. Gradle로 추가한 일반 라이브러리는 보통 앱 APK 안에 포함된다.

앱이 실행되면 Android OS가 앱 프로세스를 만들고, 그 안에서 ART가 dex 코드를 실행한다. APK 안의 코드와 리소스는 원본 파일이 있는 file-backed 메모리로 다뤄질 수 있고, 실행 중 생기는 객체, 힙, 스택은 원본 파일이 없는 anonymous 메모리다.

그래서 메모리 부족 상황에서 OS는 이 둘을 다르게 다룬다. file-backed 페이지는 버렸다가 다시 읽을 수 있지만, anonymous 페이지는 그냥 버릴 수 없기 때문이다.
