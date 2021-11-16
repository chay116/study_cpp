---
description: Accustoming Yourself to C++
---

# Chapter 1

## 1. View C++ as a federation of language

초기 C++은 C언어에서 기능들이 결합된 상태에서 진화해왔다. 이렇게 진화해온 C++은 현재 multiparadigm programming language라 불리며, 절차적(procedural) 프로그래밍을 기본으로 하여 객체 지향, 함수식, 일반화 프로그래밍을 포함하며 메타프로그래밍 개념까지 지원한다. 하지만 이러한 다양한 기능의 추가는 사용법의 혼란을 야기하였다. 그렇다면 어떻게 하면 이를 효과적으로 활용할 수 있을까?

이 책에서는 C++을 단일 언어가 아닌 여러 언어의 연합체로 보는 것을 추천한다. 각각 하위 언어의 특성을 이해함으로서 전체를 이해하도록 이끌고 있다. C++의 하위 언어는 다음과 같이 4가지가 존재한다.

* C : 블록, 문장, 선행 처리자, 데이터 타입 등 많은 부분이 C에서 제공되었다. 역으로 예외, 템플릿, 오버로딩과 같은 기능들은 지원하지 않는다.
* Object-Oriented C++ : 클래스를 쓰는 부분들이 모두 여기에 해당한다. 캡슐화, 상속, 다형성, 가상 함수 등 객체 지향 설계의 규칙 대부분이 포함된다.
* Template C++ : C++ 일반화 프로그래밍 부분입니다.
* STL : 템플릿 라이브러리로서, container, iterator, algorithm 및 function object가 서로 얽혀서 구성되어져 있습니다.

이러한 네가지 하위 언어들의 연합체로 이루어져있고, 각각의 부분에 따라 효과적인 코딩 방식과 규칙이 다르기 때문에, C++의 효과적인 사용을 위해서는 하위 언어들의 이해가 기반이 되어야합니다.&#x20;

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* C++를 사용한 효과적인 프로그래밍 규칙은 C++의 어떤 부분을 사용하느냐에 따라 달라집니다.

## 2. Prefer const, enum, inline to #define

가급적 선행 처리자보다 컴파일러를 더 활용하자. #define보다는 const, enum, inline이 더 깔끔하게 활용될 수 있다.

*   const

    위와 같이 ASPECT\_RATIO를 다음과 같이 #define으로 정의된 경우는, 선행 처리자가 ASPECT\_RATIO와 같은 symbolic name으로 판단하는 것이 아닌 숫자 상수로 바꾼 후에, 컴파일을 한다. 따라서 기호 테이블에 포함되지 않지 않기에 디버깅할 시에 문제가 발생할 수 있다. 또한, ASPECT\_RATIO를 하나씩 1.653으로 바꿔주기 때문에 등장 횟수만큼 사본을 만들어줘야 하기에 컴파일 후의 코드 크기가 커진다.

    또한 const 변수의 경우는 선언과 동시에 초기화가 되어야하는데, 클래스의 정의에는 상수의 초기값이 있으면 안되기 때문에, 클래스 상수의 정의는 구현 파일에 둡니다. 또한 #define의 경우는 유효범위 개념이 없기 때문에 이렇게 활용 불가능하다. 가끔 오래된 컴파일러가 이렇게 작동하지 않는 경우가 존재하는데, const 상수의 선언을 헤더 파일에 두고, 정의는 구현파일에서 하면 작동합니다.

    ```cpp
    # define ASPECT_RATIO 1.653
    const double AspectRatio = 1.653; // const를 사용하여 변경하지 못하도록 함.

    // 문자열을 쓰는 경우
    const char * const authorName = "Scott Meyers"; // char* 기반으로 쓰려면 포인터 및 대상까지 const를 써줘야합니다.
    const std::string authorName("Scott Meyers"); // 따라서 string 객체에 사용하는 것이 더 효율적입니다.

    // 클래스 멤버로 상수를 정의하는 경우
    class GamePlayer {  // 상수의 유효범위를 클래스로 한정하는 경우는 멤버로 만들어야 합니다.
    private:
        static const int NumTurns = 5;    // 클래스 상수는 구현 파일에서 정의를 한다.
        int scores[NumTurns];            // 상수를 사용하는 부분
        ...
    };
    ```
*   enum

    해당 클래스 컴파일하는 도중에 클래스 상수의 값이 필요할 때(배열 멤버를 선언할 때 사용하는 경우), 위에서 말한 방법이 통하지 않을 수도 있습니다. 따라서 enum을 통해서 이를 회피할 수 있습니다. 또한 enum의 경우 #define처럼 쓸데 없는 메모리 할당도 절대 저지르지 않습니다.

    ```cpp
    class GamePlayer {
    private:
        enum { NumTurns = 5 };    // "나열자 둔갑술": NumTurns를
                                // 5에 대한 기호식 이름으로 만듭니다.
        int scores[NumTurns];
        ...
    };
    ```
*   inline

    \#define 함수를 매크로 함수로서 사용이 가능합니다. 이는 호출 오버헤드를 일으키지 않는 매크로인데, 이는 괴현상을 일으킬 수 있습니다. 함수 호출을 없애줄 수 있지만, 좋지 않는 결과를 낼 수 있습니다. 이를 대체할 수 있는 것이 inline입니다. 이 함수를 템플릿이기 때문에 동일 계열 함수군을 만들어냅니다. 따라서 매크로 함수의 효율은 그대로 유지하면서도, 정규 함수의 동작 방식 및 타입 안정성까지 완벽히 취할 수 있습니다.

    ```cpp
    #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
     
    int a = 5, b = 0;
     
    CALL_WITH_MAX(++a, b);        // a가 두 번 증가합니다.
    CALL_WITH_MAX(++a, b + 10);    // a가 한 번 증가합니다.

    template <typename T>
    inline void callWithMax(const T& a, const T& b)
    {
        f(a > b ? a : b);
    }
    ```

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* 단순한 상수를 쓸 때는, #define보다 const 객체 혹은 enum을, macro 함수를 만들 때는 inline함수를 우선시하자.



## 3. Use const whenever possible

const는 제작자의 의도를 컴파일러 및 다른 프로그래머와 나눌 수 있는 의미적인 제약입니다. const는 다양하게 사용될 수 있습니다. 다음과 같이 포인터에 사용할 수 있습니다.

```cpp
char greeting[] = "Hello";
char *p = greeting; // 비상수 포인터, 비상수 데이터
char const *p = greeting; // 비상수 포인터, 상수 데이터
const char *p = greeting; // 상수 포인터, 비상수 데이터
const char const *p = greeting; // 상수 포인터, 상수 데이터
```

const 포인터의 경우는 좌표값이 바뀔 수 없고, const 데이터의 경우는 값이 바뀔 수 없습니다. 또한, operator의 경우 const가 붙는 경우가 많은데 이는 `(a * b) = c` 와 같이 이항 연산자에서 리턴 타입이 상수성을 갖지 않아 위와 같은 대입이 되는 경우를 사용될 수 있습니다.

#### 상수 멤버 함수

멤버 함수에 붙는 const 키워드는 `해당 멤버 함수가 상수 객체에 대해 호출될 함수이다.` 라는 것을 명시합니다. 이는 두가지 이유로 중요합니다.

* 먼저 클래스의 인터페이스를 이해하기 좋게 하기 위해서입니다. const를 붙여 선언하면 컴파일러가 사용상의 에러를 잡아내는 데 도움을 줍니다. const는 어떤 유효범위에 있는 객체에도 붙을 수 있으며, 함수 매개변수 및 반환 타입에도 붙을 수 있으며, 멤버 함수에도 붙을 수 있습니다.
* c++ 성능을 높이는 reference to const를 진행하려면, 상수멤버 함수가 준비되어 있어야 한다. const 키워드 있고 없고의 차이만 있는 멤버 함수들은 오버로딩이 가능함.

#### 비트수준 상수성 vs 논리적 상수성

컴파일러 쪽에서 보면 비트수준 상수성을 지켜야 하지만, 여러분은 개념적인(논리적인) 상수성을 사용해서 프로그래밍 해야 합니다.

* 비트수준 상수성(물리적 상수성): 어떤 객체의 멤버 함수가 객체의 정적 멤버를 제외한 데이터 멤버를 건드리지 않은 경우에만 그 멤버 함수가 'const'임을 인정함. 하지만 이때 아래와 같이 비트적 상수성은 지켜지고 있으나 원하지 않던 방향으로 진행되어 버리는 경우가 존재합니다. 이에 논리적 상수성이 등장합니다.

```cpp
class CTextBlock {
	public : 
		…
		char& operator[](std::size_t position) const {return pText[position]; }
	private :
		char *pText;
};

const CTextBlock cctb("hello");
char *pc = &cctb[0];
*pc = 'J';
```

* 논리적인 상수성: 유연성을 부여하여 상수 멤버 함수도 객체의 일부 몇 비트를 바꿀 수 있되 사용자 측에서 알아채지 못하도록(객체의 상태에 영향을 주지 않도록) 해야 함. 따라서 이를 mutable로 통제함.

```cpp
class CTextBlock {
	public : 
		…
		std::size_t length() const;
	private :
		char *pText;
		mutable std::size_t textlength;
		mutable bool lengthIsValid; //mutable 사용할시, 상수멤버함수에서 데이터멤버를 변경할수있다.
std::size_t Ctextblock::length() const {
	if(!lengthIsValid) {
		TextLength = std::strlen(pText);
		lengthIsValid = true;
	}
	return textlength;
}
```

#### 코드 중복 피하기

* 상수 멤버 및 비상수 멤버 함수가 기능적으로 서로 똑같게 구현되어 있을 경우에는 코드 중복을 피하는 것이 좋은데, 이때 비상수 버전이 상수 버전을 호출하도록 만드세요. 비상수 버전이 const\_cast 및 static\_cast를 활용합니다.

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* const를 붙여 선언하면 컴파일러가 사용 상의 에러를 잡아내는 데 도움을 준다
* 컴파일러 쪽에서 보면 비트수준 상수성을 지켜야 하지만, 사용자는 노릴적인 상수성을 사용하여 코딩해야함.
* 상수 멤버 및 비상수 멤버 함수가 서로 똑같게 구현되어 있을 경우, 코드 중복을 피해야하므로, 비상수 버전이 상수 버전을 호출하도록 만드는 것이 좋음.

## 4. Make sure that objects are initialized before they're used

객체의 값을 초기화하는데, 초기화가 잘 되는 경우가 있고 그렇지 않은 경우가 있다. 잘 초기화가 되지 않은 값을 읽는 경우 잘 작동하는 경우가 존재하는데, 객체 내부가 이상한 값으로 가득차게 된다. 따라서 `사전에 모든 객체를 항상 초기화하는 것이 중요하다`. 특히 객체의 경우는 초기화가 생성자로 귀결되는데, 이를 대입을 통해도 해결 할 수 있지만, 멤버 초기화 리스트를 통하면 좀 더 깔끔하고 생성자만 발동시키기 때문에 더 효율적입니다. 주의할 점으로는 두가지가 있습니다.

* 초기화 리스트에 데이터 멤버를 나열할 때는 클래스에 각 데이터 멤버가 선언된 순서와 똑같이 나열합시다. 이를 깔끔하게 하기 위해서는 초기화가능한 데이터멤버들을 모아서 함수화한다.
* const거나 reference로 되어 있는 데이터 멤버의 경우에는 반드시 초기화 되어야하기 때문에 멤버를 초기화 리스트를 넣는 것이 의무화됩니다.
* 초기화 순서는 기본 클래스→파생클래스 순으로 호출합니다.

```cpp
class score
{
	private:
	    int korean;
	    int math;
	    int english;
	public:
	    babo(int a, int b, int c): korean(a), math(b), english(c){ };
};
```

정적 객체(static object)는 객체 생성 시점부터 프로그램이 끝날 때까지 살아 있는 객체를 일컫습니다. **정적 객체**로는 \*\*`전역** 객체`, \*\*`네임스페이스 유효범위**에서 지정된 객체`, \*\*`클래스나 함수** 내에서 static으로 선언된 객체`, 그리고 \*\*`파일 유효범위**에서 static으로 정의된 객체`가 있습니다. 즉, 이 다섯 종류는 프로그램 종료 시점에 소멸됩니다. 또한 정적 객체는 **함수안에 있으면 지역 정적 객체**라 불리며, 그 외는 **비지역 정적 객체**라 불려진다.

이러한 비지역 정적 객체의 초기화 순서는 개별 번역 단위에서 정해집니다. 여기서 \*\*번역 단위(translation unit)\*\*는 `컴파일을 통해 하나의 object file을 만드는 바탕이 되는 소스 코드`를 일컫습니다. 앞의 말을 풀어서 설명하면, 여러 소스 파일이 있을 때, 한 파일에서 다른 소스 파일의 정적 객체를 참조하는데 이 객체가 초기화가 되어 있지 않을 수도 있다는 것입니다. 따라서 비지역 정적 객체들의 초기화 순서 문제는 피해서 설계해야만 합니다. (비지역 객체는 지역 객체로 설정하면 된다!)

```cpp
class FileSystem { ... };

FileSystem& tfs(void)            // 이 함수는 FileSystem 클래스 안에
{                                // 정적 멤버로 들어가도 됩니다.
 
    static FileSystem fs;        // 지역 정적 객체를 정의하고 초기화 합니다.
    return fs;                    // 이 객체에 대한 참조자를 반환합니다.
}                                // 지역 정적 객체는 함수가 처음 불릴 때
                                // 반드시 초기화 됩니다.
class Directory { ... };
 
Directory::Directory( params )
{
    ...                            // tfs의 참조자였던 것이 tfs()로 바뀌었습니다.
    std::size_t disks = tfs().numDisks();
    ...
}
```

#### <mark style="color:orange;">이것만은 잊지 말자!</mark>

* 기본제공 타입의 객체는 직접 손으로 초기화.
* 생성자에서는 멤버 초기화 리스트를 즐겨 사용할 것
* 비지역 정적 객체들의 초기화 순서 문제를 피하기 위해 지역 정적 객체로 바꿔서 설계.

