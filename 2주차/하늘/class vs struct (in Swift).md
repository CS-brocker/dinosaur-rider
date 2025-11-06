> Choosing Between Structures and Classes ([Apple developer article](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes?utm_source=chatgpt.com))


### 구조체를 기본으로 사용하라

구조체는 참조타입이 아닌 값타입이기 때문에 상태 변화를 추적하기 쉽다.

```swift
// class에서의 상속
class Animal {
    func sound() { print("...") }
}

class Dog: Animal {
    override func sound() { print("🐶 Bark!") }
}

// struct에서의 상속
protocol Animal {
    func sound()
}

struct Dog: Animal {
    func sound() { print("🐶 Bark!") }
}
```

상속을 제외한 모든 기능을 구조체에서도 지원한다. → 프로토콜로 구현하면 상속의 부재도 해결할 수 있다. 

class를 상속하는 것은 class만 가능한데, 프로토콜은 enum, class, struct 모두 상속할 수 있다

```swift
// protocol 상속 개념
protocol Animal {
    func sound()
}

protocol Pet: Animal {
    func play()
}

struct Dog: Pet {
    func sound() { print("🐶 Bark!") }
    func play() { print("🐶 play!") }
}

// protocol도 기본형을 줄 수 있고 override도 가능..
extension Animal {
    func sound() {
        print("...") // extension이용하여 override 되지 않았을 때의 기본형 구현
    }
}

struct Cat: Animal {
    func sound() { print("😺 Meow!") } // 
}
```


> iOS에서 화면을 그리려면 UIViewController(Activity)를 상속받는 class를 꼭 구현해야 하는데, 메모리 이슈가 생각보다 현업에서 빈번하다. 예를 들어 A화면은 진입 후 3초 뒤에 콜백받는 것이 있다고 가정할 때, A화면에서 이탈하고 다시 진입한 경우 처음 띄운 A화면에서 메모리릭이 나서 메모리 해제되지 않고 남아있다면 두번 콜백이 오게 된다. (광고 작업인 경우 크리티컬함..) 

> UIViewController(UIKit 라이브러리 내 객체)의 역할을 하는 구현체가 View라는 protocol로 바뀌었고, 이제 뷰는 struct로 만든다. ‘struct HomeView: View’ 이런 식으로 구현!

> 여튼 메모리 관련 이슈가 많으니 꼭 필요한 경우가 아니라면 struct를 사용하는게 좋아보인다. 

