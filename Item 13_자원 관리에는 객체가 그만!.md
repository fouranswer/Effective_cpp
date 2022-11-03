# Item 13: 자원 관리에는 객체가 그만!

C++에서 가장 흔한 자원이라면 동적할당한 메모리가 있는데..
(메모리를 할당하고서 해제하지 않으면 메모리가 누출된다구!)

3장에서는 객체기반 방식의 자원관리를 보여준다.

```c++

class Investment {...};   // 여러 형태의 투자를 모델링한 클래스 계통의 최상위 클래스

Investment* createInvestments();    // Investment 클래스 계통에 속한 클래스의 객체를 동적할당하고 그 포인터를 반환
                                    // 이 객체의 해제는 호출자 쪽에서 직접 해야합니다. 

void f() {
  Investment *pInv = createInvestments();   // 팩토리 함수를 호출합니다.
  
  ...                                       // 잘 사용합니다.
  
  delete pInv;                              // 객체 해제!  멀쩡한 delete 일까요?
```

멀쩡해보이지만 다음과 같은 이유로 delete가 실패할 수 있어요.
1. ... 어디에서 return 문이 들어가면..?
2. continue, goto 문에 의해서..?
3. 예외를 던지면...(delete 실행이 안된다)

=> f함수가 항상 delete로 갈거라고 믿지 말자!

## C++이 자동으로 호출해주는 소멸자에 의해 자동으로 해제해줄 수 있다!

표준 라이브러리에는 auto_ptr이 있습니다. 
auto_ptr은 포인터와 비슷하게 동작하는 객체 스마트 포인터(Smart Pointer)로서, 가리키는 대상에 대해 소멸자가 자동으로 delete를 불러주도록 설계되어있음!

```c++

void f() {
  std::auto_ptr<Investment> pInv(createInvestment());   // 팩토리 함수를 호출합니다. 

  ...                                                   // pInv 사용
}                                                       // auto_ptr의 소멸자를 통해 pInv를 삭제
```

## 자원 관리에 객체를 사용하는 방법의 2가지 특징
### 1. 자원을 획득한 후에 자원 관리 객체에게 넘깁니다. 

std::auto_ptr<Investment> 이런식으로 자원 관리 객체를 사용하는 방식으로 "자원 획득 즉 초기화(Resource Acquisition Is Initialization: RAII)"  

자원 획득과 자원 관리 객체의 초기화가 바로 한 문장에서 이루어지는 것이 너무나도 일상적이기 때문임.
"자원을 획득하고 나서 바로 자원 관리 객체에 넘겨준다"

### 2. 자원 관리 객체는 자신의 소멸자를 사용해서 자원이 확실히 해제되도록 합니다. 

소멸자는 소멸될때 자동적으로 호출되기 때문에 자원 해제가 제대로 이루어진다. 
예외상황에 빠지면 많이 꼬이지만, 해당 문제는 Item 8을 참조하자. 


-------

어떤 객체를 가리키는 auto_ptr은 2개면 안된다. 
(2번을 지운다구? 절대 안돼..)

auto_ptr의 경우 객체를 복사하면 원본 객체를 null로 만든다. 
복사하는 객체만이 유일한 소유권(ownership)을 가진다고 가정함.

```c++
std::auto_ptr<Investment> pInv(createInvestment);    // pInv1이 가리키는 건  create에서 반환된 객체

std::auto_ptr<Investment> pInv2(pInv1);    // pInv2가 그 객체를 가리키지만 pInv1은 null이다.

pInv1 = pInv2;  // pInv1은 그 객체를 가리키고있으면서, pInv2는 null임.

```

auto_ptr이 관리하는 객체는 2개 이상의 auto_ptr 객체가 잡고 있으면 안된다는 요구사항때문에 auto_ptr은 최선이 아니다.

그 대안으로 

### RCSP(참조 카운팅 방식 스마트 포인터: reference-counting smart pointer)

RCSP는 어떤 자원을 가리키는(참조하는) 외부 객체의 개수를 유지하다가 0이 되면 해당 자원을 자동 삭제하는 스마트 포인터이다.

#### tr1::shared_ptr(item 54 참조)가 대표적인 RCSP

```c++
void f() {
  ...
  std::tr1::shared_ptr<Investment> pInv(createInvestment());    // 팩토리 함수 호출
  ...           // pInv 사용
}               // shared_ptr의 소멸자를 통해 pInv 자동 삭제

void f() {
  ...
  std::tr1::shared_ptr<Investment> pInv1(createInvestment());    // pInv1이 가리키는 건  create에서 반환된 객체

  std::tr1::shared_ptr<Investment> pInv2(pInv1);    // pInv1, pInv2가 동시에 그 객체를 가리킴

  pInv1 = pInv2;                             // 변한게 없다.
}                                            // pInv1, pInv2는 소멸 및 객체도 자동삭제
```

앞서 auto_ptr과 코드상으로 다를바가 없지만, 훨신 자연스럽게 자원 관리가 된다..(고 한다)

##### "와 shared_ptr을 쓰는게 짱이구나!" 하겠지만, 여러 방법 중 하나일 뿐이다.

-------------

### 다만, 동적으로 할당한 배열에 대해서는 위의 방법을 쓸 수 없습니다.

#####auto_ptr , tr1::shared_ptr은 내부에서 delete연산자를 사용합니다.  delete [] 가 아닙니다. 

동적 할당 배열에 대해서는 auto_ptr, tr::shared_ptr은 std (표준 라이브러리)에서 지원하지 않는다. 

```c++
  std::auto_ptr<std::string> aps(new std::string[10]);

  std::tr1::shared_ptr<int> spi(new int[1024]);

  // 둘 다 잘못된 delete가 사용된다. 
```

이러한 문제는 boost 라이브러리에서 지원합니다. boost::scoped_array, boost::shared_array

# 이것만은 잊지 말자!
1. 자원 누출을 막기위해, 생성자 안에서 자원을 획득하고 소멸자에서 그것을 해제하는 RAII객체를 사용합시다. 
2. 일반적으로 널리 쓰이는 RAII 클래스는 tr1::shared_ptr 그리고 auto_ptr 입니다. 이 둘 가운데 tr1::shared_ptr이 복사 시의 동작을 직관적으로 만들기 때문에 대개 더 좋습니다. 반면, auto_ptr은 복사되는 객체(원본 객체)를 null로 만들어 버립니다. 
