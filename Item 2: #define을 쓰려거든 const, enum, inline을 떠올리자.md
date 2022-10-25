# item2: #define을 쓰려거든 const, enum, inline을 떠올리자

"가급적 선행 처리자보다 컴파일러를 더 가까이하자..." 라는 제목이 더 맞는데 음?? 

## #define의 문제점
```c++
#define ASPECT_RATIO   1.653
```

사람의 눈에는 위의 코드가 기호식으로 보이지만 컴파일러에게는 보이지 않는 코드다. #define..
컴파일러로 넘어가기전에 전처리기(preprocessor)에서 전부 숫자상수로 치환해버리기 때문이다.
단순 숫자로 치환되어있기 때문에 디버깅시에 갑자기 1.653이 어디서 튀어나온 것인지를 알 수가 없게 된다. 
(기호테이블이 없기 떄문에..)

## 해결법은 매크로 대신 상수를 쓰는 것!!
```c++
const double AspectRatio = 1.653;
```
이렇게 하면 언어차원에서 지원하는 상수 타입의 데이터이기 때문에 컴파일러의 눈에도 보이고, 기호테이블에도 들어간다. 

또한 코드 사이즈도 "약간"이 이득보는 효과가 있다. 
단순 무식하게 1.653을 모두 대입한게 아니라 한 곳의 상수 변수를 참조하기때문이다. 

## 이렇게 사용할때 주의할 점

### 상수포인터 정의(const pointer)
```c++
const char * const authorName = "Scott Meyers"; 
```

포인터는 꼭 const 선언
포인터가 가리키는 대상까지 const 선언
두 번 선언을 해줘야합니다.

### 문자열 상수
char * 보다는 string 객체가 대체적으로 사용하기 좋다.
```c++
const std::string authorName("Scott Meyers");
```

### 클래스 멤버로 상수를 정의하는 경우 

어떤 상수의 유효범위를 클래스로 한정하고자 할때, 그 상수를 클래스의 멤버로 만들어야함!
그 상수의 사본 개수가 한개를 넘지못하게 하려면 정적(static) 멤버로 만들자

```c++
class GamePlayer {
    private:
    static const in NumTurns = 5; // 상수선언
    int scores[NumTurns];
}
```

NumTurns은 '선언(declaration)' 된 것이다. 정의 X

일반적으로는 '정의'가 마련되어야하지만, 정적 멤버로 만들어지는 정수류(char, bool...) 타입의 클래스 내부 상수는 예외입니다. 
정의 없이 선언만 해도 아무 문제가 없습니다. 

```c++
const int GamePlayer::NumTurns; // 값을 넣지 않아도~~ static은 괜찮아!
```

클래스 상수의 정의는 구현파일에 둡니다. 헤더파일에 두지 않음!
#### 클래스 상수의 초기값은 해당 상수가 선언된 시점에서 바로 주어지기 때문입니다. 

다만, 오래된 컴파일러는 위의 문법을 안 받을 가능성 있음. 

#### #define은 클래스 정의를 할 수 없고, 캡슐화 혜택을 받을 수 없다. 
'private'성격의 #define은 없다는 뜻임.


오래된 컴파일러는 정적 클래스 멤버가 선언된 시점에 초기값을 주는 것이 대개 맞지 않다고 판단한다..

따라서 초기값을 상수 '정의' 시점에 주도록 해야한다.

```c++
class CostEstimate {
    private:
    static const double FudgeFactor; // static 클래스 상수 선언! 헤더 파일에 둡니다. 
    ...
};

const double CostEstimate:FudgeFactor = 1.35;   // static 클래스 상수 정의! 구현파일에 둡니다. 
```

### 오래된 컴파일러는 이정도선에는 마무리되지만, 배열 멤버를 선언할때 오류를 뱉을 수 있다. 

이럴때는 "나열자 둔갑술(enum hack)" 을 사용할 수 있다. 
<u>나열자(enumerator) 타입의 값은 int가 놓일 곳에도 쓸 수 있는 C++의 진실(?)을 활용하는 것이다.(???)</u>
(무슨 말이지?)

```c++
class CostEstimate {
    private:
    enum { NumTurns = 5 };    // Numturns를 5에 대한 기호식 이름으로 만든다. 

    int scores[NumTurns];
    ...
};
```

## 나열자 둔갑술(enum hack)을 알아두는 것이 도움이 되는 이유

1. 나열 둔갑술(enum hack)은 동작방식이 const보다 #define에 가깝다. 
다른 사람이(?) 상수를 가지고 주소를 얻는 다던가 참조자를 쓰는 것이 싫다면 enum이 꽤 괜찮은 기법이 된다. 
컴파일러 마다 const 객체에 대한 처리가 다르고, 프로그래머는 어떤 컴파일러 간에 확실하게 const 객체에 대한 메모리를 만들지않는 방법을 쓰고 싶지 않을 것이다. 
또한, enum은 #define 처럼 쓸데없는 메모리 할당을 하지 않는다. 

2. 많이 이 기법이 쓰이므로, 훈련이 필요하다 / 템플릿 메타프로그래밍에서 핵심 기법임.

```c++
#define CALL_WITH_MAX(a,b)   f((a) > (b) ? (a) : (b))     // 인자 마다 괄호를 씌워줘야 합니다. 넘길때 문제 생김..

int a = 5, b = 0;
CALL_WITH_MAX(++a, b);     // a 2번 증가
CALL_WITH_MAX(++a, b+10);  // a 1번 증가
```

코드 보기만해도 내가 의도 했던 것이 아닌 행동을 하게된다!!!

이 문제를 해결하는 함수의 동작방식 및 타입 안정성까지 완벽한 방법이 있다. 
## inline 함수에 대한 template

```c++
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a : b);
}
```

함수 템플릿이기떄문에 위의 코드에서는 동일한 타입의 인자를 받고, 둘 중 큰 것을 넘기는 구조.
인자에 대해서 생각할 필요가 없고, 유효범위라던가 접근 규칙이 그대로 따라가기 때문에 함수 템플릿을 쓰는 것이 더 안전하다.


const, enum, inline을 자주 쓰다보면, #define 쓰는 경우가 줄어들게 됩니다. 

# 이것만은 잊지말자!
1. 단순한 상수를 쓸 때는, #define 보다 const 객체 혹은 enum을 우선 생각하자
2. 함수처럼 쓰이는 매크로를 만들려면, #define보다 inline 함수를 먼저 생각하자

