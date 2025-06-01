---
layout: post
title: "Complex std::string"
subtitle: "std::string의 복잡한 내부 구조와 동작 방식"
date: 2025-01-24 23:45:13 -0400
background: '/img/posts/2025-01-24-01.jpeg'
published: true
categories: [tech]
tags: [C++, string, 표준라이브러리, 템플릿]
---

### C++ 실행 파일에서 본 `std::string`의 복잡성

C++로 빌드된 실행 파일을 디컴파일해본 적이 있으신가요? 특히 `std::string` 클래스를 사용하고 있는 경우, 코드가 유독 복잡하게 나타나는 걸 보실 수 있습니다. "다른 컨테이너들은 간단한데, 왜 `std::string`은 이렇게 복잡할까?"라는 의문을 가진 적이 있었는데요, 이 질문이 C++ 템플릿과 문자열 설계 철학을 이해하는 데 큰 도움이 되었던 기억이 납니다.

![std::string](/img/posts/2025-01-24-basicstr.jpg)

이번 글에서는 그때 알게 된 내용을 정리해 보려고 합니다.

---

## `basic_string`

C++ 라이브러리에서 `std::pair`나 `std::tuple`은 비교적 단순하게 설계된 구조를 가지고 있습니다. 그런데 왜 `std::string`은 `std::basic_string`이라는 복잡한 템플릿 구조를 사용하고 있을까요?

`std::map`, `std::pair`, `std::tuple`은 키-값 구조나 정적인 타입 조합에 초점이 맞춰져 있지만, `std::basic_string`은 범용적인 문자열 컨테이너로 설계되었습니다. 이는 세상의 다양한 문자 타입(`char`, `wchar_t`, `char16_t`, `char32_t`)을 처리해야 하며, 메모리 관리와 성능까지도 고려해야 하기 때문입니다.

우선, `std::basic_string`은 아래와 같은 구조를 가지고 있습니다. [출처](https://github.com/llvm/llvm-project/blob/main/libcxx/include/string)

```cpp
template <class _CharT, class _Traits, class _Allocator>
class basic_string { ... }
```

- **`CharT`**: 문자열의 문자 타입 (예: `char`, `wchar_t`, `char16_t`, `char32_t`).
- **`Traits`**: 문자 연산 방식을 정의하는 정책 클래스. 비교, 복사, 길이 계산 등을 커스터마이징할 수 있습니다.
- **`Allocator`**: 메모리 할당 방식을 정의하는 `std::allocator`. 커스텀 메모리 관리자를 사용할 수 있도록 유연성을 제공합니다.

다행히도, 복잡한 템플릿 구조를 감추기 위해 C++ 라이브러리는 아래와 같은 별칭(alias)을 제공합니다. [출처](https://github.com/llvm/llvm-project/blob/main/libcxx/include/string)

```cpp
typedef basic_string<char>    string;
typedef basic_string<wchar_t> wstring;
typedef basic_string<char8_t> u8string;  // C++20
typedef basic_string<char16_t> u16string;
typedef basic_string<char32_t> u32string;
```

이를 통해 사용자는 템플릿 세부사항에 신경 쓰지 않고, `std::string` 같은 간단한 형태로 문자열을 사용할 수 있습니다.

---

## `std::string` vs `std::wstring`

### 한글을 사용할 때 `wstring` 대신 `string`을 써도 괜찮을까?

입출력만 한다면 대부분의 경우 `std::string`을 사용해도 문제가 없습니다. 그러나 `std::string`과 `std::wstring`의 차이는 단순히 문자 타입(`char` vs `wchar_t`)뿐만 아니라, **문자열을 다루는 방식(`traits`)** 에서 나타납니다. 이를 이해하기 위해 간단한 예제를 살펴보겠습니다.

```cpp
void test_string() {
    std::string s = "한글!";
    std::cout << s << "(length: " << s.length() << ")\n";
    std::cout << s.substr(0, 2) << "(length: " << s.substr(0, 2).length() << ")\n";
}

void test_wstring() {
    std::locale::global(std::locale(""));
    std::wcout.imbue(std::locale());

    std::wstring s = L"한글!";
    std::wcout << s << L"(length: " << s.length() << ")\n";
    std::wcout << s.substr(0, 2) << L"(length: " << s.substr(0, 2).length() << ")\n";
}
```

출력 결과:

```
한글!(length: 7)
�(length: 2)
한글!(length: 3)
한글(length: 2)
```

위 결과를 보면, `std::string`과 `std::wstring`의 **문자 길이 계산**과 **문자열 자르기(substring)** 결과가 다릅니다. 이는 두 클래스가 **문자 타입(`char` vs `wchar_t`)** 뿐만 아니라, **내부적으로 문자를 다루는 방식(`traits`)** 에서도 차이를 보이기 때문입니다.

### 컴파일 결과

```llvm
@.str = private unnamed_addr constant [8 x i8] c"\ED\x95\x9C\xEA\xB8\x80!\00", align 1
@.str.4 = private unnamed_addr constant [4 x i32] [i32 54620, i32 44544, i32 33, i32 0], align 4
```

- `std::string`은 UTF-8로 인코딩된 **8비트 문자 배열**로 저장됩니다. (7바이트)
- `std::wstring`은 UTF-32로 인코딩된 **32비트 코드 포인트 배열**로 저장됩니다. (3개 코드 포인트)

따라서, `std::wstring`에서는 `substr(0, 2)`로 2개의 문자를 올바르게 자를 수 있지만, `std::string`에서는 바이트 단위로 잘리기 때문에 깨진 문자가 출력됩니다.

---

## char_traits 란?

`std::basic_string`의 **`char_traits`** 는 문자열 연산(비교, 복사, 길이 계산)을 정의하는 **정책 클래스**입니다. 문자열 조작 함수는 대부분 이 `char_traits` 클래스의 static 메서드를 호출합니다.

예를 들어, `std::string::compare` 함수는 내부적으로 `Traits::compare`를 호출합니다.

### 예제 코드

```cpp
template <class _CharT, class _Traits, class _Allocator>
inline int basic_string<_CharT, _Traits, _Allocator>::compare(size_type __pos1, size_type __n1, const value_type* __s, size_type __n2) const {
    return traits_type::compare(data() + __pos1, __s, std::min(__rlen, __n2));
}

// char_traits<char> 구현
template <>
struct char_traits<char> {
    static constexpr int compare(const char_type* __s1, const char_type* __s2, size_t __n) noexcept {
        return std::memcmp(__s1, __s2, __n);
    }
};

// char_traits<wchar_t> 구현
template <>
struct char_traits<wchar_t> {
    static int compare(const char_type* __s1, const char_type* __s2, size_t __n) noexcept {
        return std::wmemcmp(__s1, __s2, __n);
    }
}
```

- `char_traits<char>`는 `std::memcmp`를 사용해 **바이트 단위 비교**를 수행합니다.
- `char_traits<wchar_t>`는 `std::wmemcmp`를 사용해 **코드 포인트 단위 비교**를 수행합니다.

---

## 결론

`std::pair`와 `std::tuple`은 단순히 타입의 관계를 정의하는 데 초점을 맞춘 반면, `std::string`은 **다양한 문자 타입을 지원하며, 효율적으로 데이터를 저장하고 조작할 수 있도록 설계된 도구**입니다.

|특성|`std::pair` / `std::tuple`|`std::string`|
|---|---|---|
|**구조**|단순 타입 조합|복잡한 템플릿 기반 구조|
|**템플릿 매개변수**|타입 관계 정의|문자 타입, 연산 정책, 메모리 관리|
|**최적화 필요성**|없음|있음 (SBO, UTF-8/UTF-32 처리)|

디컴파일 및 IR 분석을 통해 `std::string`과 `std::wstring`의 동작 차이를 명확히 이해할 수 있습니다. 단순히 라이브러리를 살펴보는 것 뿐만 아니라, 컴파일된 바이너리를 함께 살펴보시길 권장합니다!