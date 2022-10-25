# Item 8: 예외가 소멸자를 떠나지 못하도록 붙들어 놓자

```c++

class Widget {
public:
  ...
  ~Widget() { ... }           // 이 함수에서 예외가 발생된다고 가정
};

void doSomething()
{
  std::vector<Widget> v;
  ...
}                            // v는 여기에서 자동 소멸된다.
```

가정
만약 이 Widget이 10개라고 치고, 첫 번째 것을 소멸시키는 도중 예외가 발생되었다. 
예외가 발생되서 나머지 9개에 대해서도 소멸시켜야한다. 그런데 2번째 것을 소멸시키는 중에 또 예외가 발생한다면...?

이렇게 되면.. 프로그램이 종료가 되거나, 미정의 동작을 하게될지도 모른다( c++은 어찌할지 모른다)

따라서 예외가 터지는 순간에 미정의동작을 하게하는 소멸자에게 문제가 있다. 

```c++

class DBConnection {
public:
  ...
  static DBConnection create();    // DBConnection 객체를 반환하는 함수.
  
  void close();                    // 연결 닫기. 연결 실패시 예외를 던진다.
};
```

```c++
class DBConn {                    // DBConnection 객체를 관리하는 클래스
public:
  ...
  ~DBConn()                       // 데이터베이스 연결이 항상 닫히도록 확실히 챙겨주는 함수 
  {
    db.close();
  }
  
private:
  DBConnection db;
};


{
  DBConn dbc(DBConnection::create());     // DBConnection 객체를 생성하고, 바로 DBConn 객체로 넘긴다.
  
  ...                                     // DBConn 인터페이스를 통해, DBConnection 객체를 사용
  
}                                         // 이제 소멸될 시간. DBConnection 객체에 대한 close 함수 호출이 이루어진다. 

```

여기까지 close만 잘 실행되면 끝나는 코드이다. 

만약 close 중에 예외가 발생한다면 어떻게 될까?

# 예외가 발생한다면?

## 1. 프로그램을 바로 끝낸다.

```c++
DBConn::~DBConn()
{
  try { db.close(); }
  catch {...}  {
    close 호출 실패 log
    std::abort();
  }
}

```

## 2. 예외를 삼켜(?) 버린다.

```c++
DBConn::~DBConn()
{
  try { db.close(); }
  catch {...}  {
    close 호출 실패 log.      // 삼킨다는 뜻이.. 그냥 아무것도 안하고 지나가게 한다는 의미같다.
    }
}
```

종료되는 것도 로그를 남기는 것도 삼켜버리는 것도...좋지만, 예외가 발생하더라도 프로그램이 계속 신뢰성있게 돌아가는 것이 중요하다. 

인터페이스를 잘 설계해서 사용자가 대처할 기회를 주는 것도 좋다. 

```c++

class DBConn {
public:
  ...
  void close().               // 사용자 호출을 배려해서 새로 만든 함수 
  {
    db.close();
    closed = true;
  }
  
~DBConn()                     // 소멸자
{
  if(!closed)                 // 안 닫혓냐?
  try {                 
    db.close();               // 닫아봐
  }
  catch (...) {               // 닫다가 실패했네..
    close 호출 실패 log.        // 실패를 알리고, 종료하거나 예외를 삼킨다.
  ...
  }
}

private:
  DBConnection db;
  bool closed;
};

```

# 예외는 소멸자가 아닌 다른 함수에서 비롯된 것이어야 한다.

예외를 일으키는 소멸자는 시한폭탄(불완전 종료 또는 미정의 동작 수행)

사용자에게 에러를 처리할 수 잇도록 기회를 주자.

# 이것만은 잊지말자!

1. 소멸자에서 예외가 빠져나가면 안됩니다. 만약 소멸자 안에서 호출된 함수가 예외를 던질 가능성이 있다면, 어떤 예외이든지 소멸자에서 모두 받아낸 후에 삼켜버리든지, 프로그램을 종료하던가 해야합니다. 
2. 어떤 클래스의 연산이 진행되다가 던진 예외에 대해 사용자가 반응해야 할 필요가 있다면, 해당 연산을 제공하는 함수는 반드시 보통의 함수(즉, 소멸자가 아닌 함수) 이어야 합니다. 




