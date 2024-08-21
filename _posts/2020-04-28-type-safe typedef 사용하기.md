---
permalink: bf37bd1fac0ec8a7a13e616a045d2c1c
---
C++에서 typedef 또는 using을 이용해 타입에 새로운 이름을 붙여줄 수 있다.

```cpp
using myint1 = int;
typedef int myint2;
```

이 타입들끼리는 원하지 않더라도 서로 연산이 가능하다.

```cpp
myint1 i1{1};
myint2 i2{2};

i1 = i2;
std::cout << i1 + i2 << "\n";
std::cout << i1 << "\n";
```

여기서부턴 취향에 따라 갈리겠지만, 타입에 더 많은 정보를 담고 싶다면 간단한 트릭을 통해 새 타입들을 구분할 수 있다.

```cpp
template <class T, class>
struct strong_type
{
	T v;
	explicit strong_type(T&& s) : v(std::move(s)) {}
	strong_type& operator=(T&& s)
	{
		v = s;
		return *this;
	}
	explicit operator const T&() const
	{
		return v;
	}
};

using myint3 = strong_type<int, struct myint3_tag>;
using myint4 = strong_type<int, struct myint4_tag>;

myint3 i3{3};
myint4 i4{4};

i3 = i4; // 컴파일 에러!
std::cout << i3 << "\n"; // 컴파일 에러!
std::cout << static_cast<int>(i3) << "\n";
std::cout << i3 + i4 << "\n"; // 컴파일 에러!

```
원래 이럴 때 쓰라고 [BOOST_STRONG_TYPEDEF](http://www.boost.org/doc/libs/release/libs/serialization/doc/strong_typedef.html)가 있는데, 탬플릿 대신 매크로를 사용한 구현이다.

typedef와 string typedef는 하스켈에서 `type`과 `newtype`같은 관계이다.

```hs
type MyInt = Int
newtype MyIntNew = MyIntNew Int

print $ (1:: MyInt) + (2:: MyInt)
print $ MyIntNew 1 + MyIntNew 2 -- 컴파일 에러!
```

------------------------------------

위 방법대로 쓰기 이전에 사용하던 방법인데, compile time string과 non type template paramter(NTTP)를 쓰면 다른 방식으로도 구현이 가능하다. [레딧](https://www.reddit.com/r/cpp/comments/bhxx49/c20_string_literals_as_nontype_template/)에서 찾은 방법인데, 현 시점 최신인 clang10에선 안되고 gcc9에선 되지만 이렇게까지 할 필요는 없을 듯. 자체적으로 구현하는 `FixedString` 말고 `std::string`이 constexpr해지면 그 때는 해볼법도 싶다.

```cpp
#include <iostream>

template <size_t N>
struct FixedString
{
	char buf[N]{};
	constexpr FixedString(char const* s)
	{
		// std::copy_n(s, N - 1, buf); // GCC9.3에선 작동하지 않는다. trunk에선 잘 됨.
		for (size_t i = 0; i < N - 1; ++i)
			buf[i] = s[i];
	}
};

template <size_t N>
FixedString(const char (&)[N]) -> FixedString<N>;

template <class T, FixedString>
struct strong_type
{
	T v;
	explicit strong_type(T&& s) : v(std::move(s)) {}
	strong_type& operator=(T&& s)
	{
		v = s;
		return *this;
	}
	explicit operator const T&() const
	{
		return v;
	}
};

using myint1 = strong_type<int, "myint1">;
using myint2 = strong_type<int, "myint2">;

int main()
{
	myint1 i1{1};
	myint2 i2{2};

	i1 = i2;  // 컴파일 에러!
	std::cout << i1 + i2 << "\n";  // 컴파일 에러!
	std::cout << static_cast<int>(i1) << "\n";
}
```