# Item 7: 다형성을 가진 기본 클래스에서는 소멸자를 반드시 가상 소멸자로 선언하자

시간을 기록하는 Timekeeper 클래스가 있다고 가정하자.

```c++
class TimeKeeper {
public:
  TimeKeeper();
  ~TimeKeeper();
...
};

class AtomicClock: public TimeKeeper (...};   // 전부 TimerKeeper를 상속받는다
class WaterClock: public TimeKeeper (...};
class WristClock: public TimeKeeper (...};

//Factory function 팩토리 함수
TimeKeeper* getTimeKeeper();   // TimeKeeper에서 파생된 클래스를 통해 동적으로 할당된 객체의 포인터를 반환합니다.

```

Factory function???
클래스의 객체에 대한 포인터를 얻기위한 용도를 위한 함수이다. 

```c++
TimeKeeper *ptk = getTimeKeeper();  // TimeKeeper 클래스 계통으로부터 동적으로 할당된 객체를 얻는다.
                                    // 이러한 객체는 힙에 있게 되므로... 

...                                 // 객체를 사용한다

delete ptk;                         // 최종적으로 자원 누출을 막기위해 삭제한다
```

문제는... 

1. getTimeKeeper 함수라는게 파생 클래스 객체 에 대한 포인터 
2. 이 포인터가 삭제될떄는 기본 클래스 포인터(TimeKeeper*) 를 통해 삭제된다는 점
3. 기본 클래스(TimeKeeper)에 들어있는 소멸자가 비가상 소멸자(non-virtual destructor) 라는 점


$\color{red}{C++}$

기본 클래스 포인터를 통해 파생 클래스 객체가 삭제될 때, 그 기본 클래스에 비가상 소멸자가 들어있으면 프로그램 동작은 **미정의 사항** 이라고 되어있음

## 이렇게 될 경우 파생클래스 쪽은 소멸되지 않는다. 기본 클래스만 쪽만 제대로 삭제된다는 말씀...

자료구조가 오염되는 것은 물론이요. 메모리는 슝슝..

## 문제의 답은 기본 클래스 소멸자 앞에 virtual 만 붙여주면 된다..! 

파생 클래스부분까지 몽땅 제대로 소멸된다!!!

```c++
class TimeKeeper {
public:
  TimeKeeper();
  virtual ~TimeKeeper();
...
};

TimeKeeper *ptk = getTimeKeeper();  

delete ptk;                           // 이제 제대로 지워진다
```

이렇듯... 가상 함수를 하나라도 가진 클래스는 가상 소멸자를 가져야하는 게 대부분 맞다.


```c++
class Point {
  public:
    Point(int xCoord, int yCoord);
    ~Point();
  private:
    int x, y;
};

```

만약 여기에서 Point 클래스의 소멸자가 가상 소멸자로 만들어지는 순간 부터 상황이 달라진다. 
C++에서 가상 함수를 구현하려면 별도의 자료구조가 추가된다. 

바로 실행 중에 객체에 대해 어떤 가상함수를 호출해주어야하는지 결정하는 vptr[] 이다.  (virtual table pointer)

또, vptr의 배열은 vtbl[] 가상함수 테이블(virtual table)이라고 불린다.

가상함수를 하나라도 갖고 있는 클래스는 반드시 관련된 vtbl을 가지고 있다. 

# 아무렇게나 막 소멸자를 virtual로 선언하는 것은 좋지않다.

가상 소멸자를 선언하는 것은 그 클래스에 가상 함수가 하나라도 들어있는 경우에만 한정합시다. 


```c++

class SpecialString : public std::string {    // std::string은 가상 소멸자가 없다요!
  ...
};

SpecailString *pss = new SpecialString("Impending Doom");   // 임박한 종말
std::string *ps;
...
ps = pss;                      // SpecialString *pss => std::string*
...

delete ps;                     // 정의되지 않는 동작이 발생
                               // SpecialString의 소멸자가 호출되지 않습니다.
```

이렇듯 비가상 소멸자가 없는 클래스를 상속받으면 골치가 아파진다. 

# 추상 클래스(abstract class)

순수(pure) 가상 소멸자는 그 자체로는 인스턴스를 못만드는 클래스이다.(객체 x)

추상 클래스는 본래 기본 클래스로 쓰일 목적으로 만들어진 것이고, 기본 클래스로 쓰이려는 클래스는 가상 소멸자를 가져야한다!

```c++

class AWOV {               // AWOV = "abstract w/o virtuals
public:
  virtual ~AWOV() = 0;     // 순수 가상 소멸자 선언
};

AWOC::~AWOV()  {...}         // 순수 가상 소멸자는 정의를 해줘야한다.
```

## 가상 소멸자의 동작하는 순서

상속 계통 구조에서 가장 말단에 있는 파생 클래스의 소멸자가 먼저 호출되는 것을 시작으로 기본 클래스 쪽으로 올라가면서 소멸자들이 하나씩 호출된다. 

가상 소멸자의 경우 다형성(polymorphic)을 가진 기본 클래스에만 적용된다.

(기본 클래스 인터페이스를 통해 파생 클래스 타입의 조작을 허용하도록 설계된 기본 클래스에서만 적용!)


# 이것만은 잊지말자

1. 다형성을 가진 기본 클래스에는 반드시 가상 소멸자를 선언해야합니다. 즉, 어떤 클래스가 가상 함수를 하나라도 갖고 있으면, 이 클래스의 소멸자도 가상 소멸자이어야합니다. 
2. 기본 클래스로 설계되지 않았거나 다형성을 갖도록 설계되지 않은 클래스에는 가상 소멸자를 선언하지 말아야합니다. 

