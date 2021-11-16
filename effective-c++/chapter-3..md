---
description: >-
  자원이란 사용을 일단 마치고 난 후엔 시스템에 돌려주어야하는 모든 것을 일컫습니다. 이번 장에서는 메모리 관리에 대해 집중해서 다룰
  것입니다.
---

# Chapter 3. 자원 관리

## Item 13: Use objects to manage resources.

#### 팩토리 패턴

```cpp
class Investment() { … }; // 투자 관련 최상위 클래스
class InvestmentA : Investment() { … }; //파생클래스
class Creator {
	public :
		Investment* Operation() { return factoryMethod(); }
	protected :
		virtual Investment* factoryMethod() = 0;
};

// 투자 파생클래스 객체를 동적할당하고 그 포인터를 반환한다. 따라서 객체의 해제는 호출자 쪽에서 직접 해야합니다.
class CreatorA : Creator{
	private: Investment* factoryMethod() { return new InvestmentA }; 
};

void f() {
	InvestmentA *pInv = CreatorA.factoryMethod(); // 팩토리 함수 호출
	… //??!
	delete pInv; // 객체 해제
}
```

void f()함수는 제대로 된 코드로 보이지만, 객체 삭제는 실패할 가능성이 높다.

* ... 구간에서 return
* 생성과 해제가 같은 루프일 때, continue, goto가 섞여있을 때
* ...에서 예외발생

f()가 객체를 제대로 해제 해줄거라고 믿으면 안된다. 따라서 얻어낸 자원이 항상 해제되도록 만들 방법은, 자원을 객체에 넣고 그 자원 해제를 소멸자가 맡도록 하며, 그 소멸자는 실행 제어가 f를 떠날 때 호출되도록 만드는 것이다. 이 아이디어는 이미 표준라이브러리에 만들어져 있다.

*   auto\_ptr (스마트 포인터) - 가르키고 있는 대상에 대해 소멸자가 자동으로 delete를 불러주도록 설계되어 있음.

    ```cpp
    void f()
    {
     std::auto_ptr<InvestmentA> pInv(CreatorA.fatoryMethod()); //동일
      … //pInv 사용
    } //auto_ptr의 소멸자를 통해 pInv를 삭제한다.
    ```

    여기에는 자원 관리에 객체를 사용하는 방법의 중요한 두가지 특징을 끄집어 낼 수 있다.

    1. 자원을 획득한 후에 자원 관리 객체에게 넘깁니다. Resource Acquisition Is Initailization(RAII)라는 이름으로 불립니다.
    2. 자원 관리 객체는 자신의 소멸자를 사용해서 자원이 확실히 해제되도록 합니다.

    또한 특이한 점으로는, auto\_ptr의 경우 복사 생성자, 복사 대입 연산자를 이용할 경우, 원본을 null로 만듭니다.
* 참조 카운팅 방식 스마트 포인터(reference-counting smart pointer: RCSP) - 다른 대안으로, RCSP가 특정한 어떤 자원을 가리키는 외부 객체의 개수를 유지하고 있다가, 그 개수가 0이 되면 해당 자원을 자동으로 삭제하는 스마트 포인터 입니다. 단, 참조 상태가 고리를 이루는 경우는 없앨 수 없습니다.

auto\_ptr 및 tr1::shared\_ptr은 소멸자 내부에서 delete 연산자를 사용합니다. 따라서 동적으로 할당한 배열에 대해 활용하면 난감합니다. 또한 동적 배열을 써도 컴파일 에러가 나지 않습니다. 이는 vector와 string으로 대체가 가능하기 때문입니다.

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* 자원 누출을 막기 위해, 생성자 안에서 자원을 획득하고 소멸자에서 그것을 해제하는 RAII객체를 쓰자
* 일반적으로 널리 쓰이는 RAII 클래스는 tr1::shared\_ptr과 auto\_ptr이 존재하며, 복사 시에 원본 객체의 삭제 여부의 차이가 존재한다.

## Item 14: Think carefully about copying behavior in resource-managing classes.

앞에서는 동적 할당한 자원에 대해서 smart ptr을 사용해 자원 관리를 하는 것은 배웠다. 이번에는 힙이 아닌 경우에, 자원 클래스를 직접 스스로 만들어야한다. RAII 방식은 객체의 scope(존재 할 수 있는 범위)를 이용하여, 자원을 관리하는데, 객체를 복사 했을때, 소멸자를 어떻게 정의하느냐에 따라서, **전기적 쇼크를 프로그래머에게 줄지, 램에게 줄지 결정지을수 있기 때문**이다.

**일반적인 사례**

1. **복사를 금지한다** - 유니크한 객체를 관리할 때 사용 한다. Uncopyable을 사용하여 복사를 금한다.
2. **관리하고 있는 자원에 대해 참조 카운팅을 수행한다.** - tr1::shared\_ptr을 사용하여 해당 자원을 가르키는 포인터만을 복사 할때, 카운팅 한다. shared\_ptr은 참조카운팅이 0일때 대상을 삭제해버린다. 하지만 옵션값을 이용하여 참조카운팅이 0일때 삭제대신 특정한 객체 또는 함수(삭제자라고 부른다.)를 호출하도록 설정할 수 있다. 까다롭지만 가장 효율적인 방법이다.
3. **관리하고 있는 자원을 진짜로 복사한다.** - 보통 문자열 객체에 대해 관리할때 사용 한다. 클래스 인터페이스를 무시할 경우가 많이 생기기 때문이다. \[]연산자 때문에.. 표준 string을 보면 힙메모리 공간을 포인터로 참조한다. 이때 복사가 일어날시 포인터도 복사되므로 깊은 복사가 된다.
4. **관리하고 있는 자원의 소유권을 옮긴다.** - 관리되는 자원이 유니크 해야만 할때 사용한다. 대표적으로 std::auto\_ptr 이 있다. auto\_ptr을 이용해서 자원의 소유자를 단 하나로 설정이 가능하다.(auto\_ptr 복사시 이전 auto\_ptr은 삭제된다.)

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* **RAII 방식의 객체의 복사에 대해서 반드시 고려하자**
* 보통은 일반적인 사례 1번과 (2,3 조합형)을 많이 사용 한다

## Item 15: Provide access to raw resources in resource-managing classes.

자원 관리 클래스에서 관리되는 자원은 외부에서 접근가능할 필요가 있다. RAII 방식의 객체의 경우 자원의 관리에 그 중점을 두었기 때문에, 그리고 설계상에 멤버 변수로써의 값으로 많이 쓰이기 때문에, 다 무너진다고 볼순 없다는게 저자의 견해이다.

```cpp
std::tr1::shared_ptr<Investment> pInv(createInvestment());
…
//그리고 이함수를 사용한다고 가정해보자.
int daysHeld(const Investment *pi);

//만약 위 함수를 아래코드와 같이사용한다면 에러!
int days = daysHeld(pInv);//ERROR!
```

daysHeld는 Investment\*의 실제포인터를 원하기 때문이다. 따라서RAII의 객체안에 실제자원을 제공 해줄 방법이 필요

**1. 명시적변환**

get과 같이 함수를 통해 값을 반환

```cpp
int days = daysHeld(pInv.get()); //실제포인터를 반환해준다 get함수를 이용해서
```

**2. 암시적변환**

포인터 역참조 연산자(operator-> / operator\*)를 오버로딩하여 암시적변환을 수행한다. 즉, 외부에서 자원 접근을 가능하게 한다. 따라서 연산자 오버로딩을 통해 daysHeld(pInv) 그대로 사용가능할 수도 있다. 또한, 유연하게 C API를 사용하기 훨씬 쉬워지고, 자연스러워진다. 하지만, 이와 같은 연산자 오버로딩으로 인해 덮어씌워짐 등의 부작용이 생길가능성이 있다.

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* 실제 자원을 직접 접근해야 하는 기존 API들도 많기에, RAII 클래스를 만들 때는 클래스가 관리하는 자원을 얻을 수 있는 방법을 제공해야함.
* 자원 접근은 명시적 변환 혹은 암시적 변환을 통해 가능하다. 안전성은 명시적 변환이, 편의성은 암시적 변환이 좋다.

## Item 16: Use the same form in corresponding uses of new and delete.

new 연산자를 사용해서 표현식(동적 할당)을 꾸미게 되면, 메모리가 할당 → 할당한 메모리에 한개 이상의 생성자 호출이라는 과정을 겪는데, delete 표현식은 우선 기존에 할당된 메모리에 대해 한개 이상의 소멸자 호출 → 메모리 해제의 순서로 작동합니다. 이때, delete 연산자가 적용되는 객체는 소멸자가 호출되는 횟수와 동일합니다.

**만약 맞추지 않으면 어떻게 되는가?**

대다수의 컴파일러는 맨 앞 숫자 (ex. 3)을 보고 배열객체라고 해석하며 해당숫자만큼의 소멸자를 호출한다.

* 한개의 객체 : \[object] - 단일 객체용에는 배열 원소의 개수와 같은 정보가 없음.
* 여러개의 객체 : \[5]\[object]\[object]\[object]\[object]\[object] - 배열용 힙 메모리에는 대게 배열원소의 개수가 박혀 들어간다. 따라서 소멸자가 몇번 호출 해야하는 지를 파악 가능.

만약 new std::string\[5]를 주고, 그냥 delete 를 이용하면 1개의 객체만 풀어주고 나머지 4개는 프로그램 종료되기 까지 잡고 있는 어처구니 없는 사태가 발생한다. 즉, \[]는 배열객체임을 확인하고, 해당 개수만큼 소멸자를 호출하기 위해 필요하다.

* typedef로 배열타입을 정의하지말자. - typedef 로 배열을 정의하면. 가시적으로 확인하기가 힘들다.

```cpp
typedef std::string strArr[4];
std::string *pal1 = new strArr; // new strArr 은 new string[4]이다..
…
delete pal; // 문제 발생
delete[] pal; //정상
```

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* new 표현식의 \[] 여부에 따라, 대응하는 delete 표현식에도 \[]를 사용

## Item 17: Store newed objects in smart pointers in standalone statements.

```cpp
int priority();//우선순위처리 함수
void processWidget(std::tr1::shared_ptr<Widget> pw,int priority); // smart pointer을 사용하도록 만들어짐.
processWidget(new Widget, priority()); // 비정상적 호출
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority()); //정상적 호출
```

다음과 같이 정의된 함수를 3번째 줄처럼 호출하면 에러가 발생한다. tr1::shared\_ptr은 explicit으로 선언되어 있기 대문에 암시적형변환이 불가하다. 4번째 줄 처럼정확하게 타입을 기재해주어야한다. 하지만 여기서도 자원낭비가 발생한다. 그이유는 **호출프로세스에서 자원 leak**가 날 수 있기때문이다.

* processWidget의 호출프로세스를 살펴보자(컴파일러마다 다르다, 2<-> 3 변경가능성도 있다.)

1. new Widget 표현식 실행
2. priority 호출
3. tr1::shared\_ptr 생성자 호출

만약 priority 호출에서 에러가 나면.. new Widget으로 만들어진 포인터가 유실될수 있다. 하지만 그 역도 가능하며, 이는 자원이 생성되는 시점과 그 자원이 자원 관리 객체로 넘어가는 시점 사이에 예외가 끼어들 수 있기 떄문이다. 따라서 다음과 같이 Widget을 별도의 독립적인 한문장으로 만든 후 넘긴다.

```cpp
std::tr1::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* new로 생성한 객체를 스마트 포인터로 넣는 코드는 별도의 한 문장으로 만들자.
