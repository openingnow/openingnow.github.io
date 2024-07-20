---
permalink: 7f8d5530020006a88c9888cc54b0b672
---
C++23에 optional/expected에 대한 모나딕 함수들이 생겼다.

12월 28일 기준 gcc trunk에서만 작동하는 중.

`and_then`은 하스켈의 `>>=`(bind, flat map) 역할을 하고 `transform`은 하스켈의 `<&>`(fmap) 역할을 한다. 같은 내용을 두 언어로 각각 구현했다. 그냥 체이닝만 하면 별 차이가 없어 보이지만 do notation을 적극 활용하면 가독성이 많이 달라진다.

자세한 문법은 [cppreference expected](https://en.cppreference.com/w/cpp/utility/expected)에서 볼 수 있는데 아직 작성이 안 되었으니 optional의 문서를 보면 된다.

```cpp
#include <charconv>
#include <expected>
#include <string>
#include <system_error>

static_assert(__cpp_lib_constexpr_charconv >= 202207L);
static_assert(__cpp_lib_to_chars >= 202306L);
auto constexpr to_string_constexpr = [](unsigned int i) noexcept {
    std::string s(10, '\0');

    if (auto const res = std::to_chars(s.data(), s.data() + s.size(), i))
        return std::string{s.data(), res.ptr};
    else
        return std::make_error_code(res.ec).message();
};

static_assert(__cpp_lib_expected >= 202202L);
using exp_int_str = std::expected<unsigned int, std::string>;
auto constexpr div_by = [](unsigned int const d) noexcept {
    return [d](unsigned int const a) noexcept -> exp_int_str {
        if (d == 0 || a % d != 0)
            return std::unexpected{"Can't div " + to_string_constexpr(a) +
                                   " by " + to_string_constexpr(d)};
        return a / d;
    };
};

auto constexpr div_by_0 = div_by(0);
auto constexpr div_by_2 = div_by(2);
auto constexpr div_by_3 = div_by(3);
auto constexpr div_by_5 = div_by(5);

static_assert(div_by_0(11) == std::unexpected{"Can't div 11 by 0"});
static_assert(div_by_2(8) == exp_int_str{4});
static_assert(div_by_2(1) == std::unexpected{"Can't div 1 by 2"});

static_assert(__cpp_lib_expected >= 202211L);
static_assert(exp_int_str{12}.and_then(div_by_2).and_then(div_by_3) ==
              exp_int_str{2});
static_assert(exp_int_str{10}.and_then(div_by_2).and_then(div_by_3) ==
              std::unexpected{"Can't div 5 by 3"});
auto constexpr div30mul7 = [](unsigned int const a) noexcept {
    exp_int_str const a2 = div_by_2(a);
    exp_int_str const a2_3 = a2.and_then(div_by_3);
    exp_int_str const a2_3_7 = a2_3.transform([](auto a) { return a * 7; });
    exp_int_str const a2_3_7_5 = a2_3_7.and_then(div_by_5);
    return a2_3_7_5;
};
static_assert(div30mul7(60) == exp_int_str{14});
```

```haskell
{-# LANGUAGE ScopedTypeVariables #-}

module Main where

divBy :: Word -> (Word -> Either String Word)
divBy d = \a -> if d == 0 || mod a d /= 0
  then Left $ "Can't div " ++ show a ++ " by " ++ show d
  else Right $ div a d

divBy0, divBy2, divBy3, divBy5 :: Word -> Either String Word
divBy0 = divBy 0
divBy2 = divBy 2
divBy3 = divBy 3
divBy5 = divBy 5

div30mul7 :: Word -> Either String Word
div30mul7 a = do
  (div2 :: Word) <- divBy2 a
  (div2_3 :: Word) <- divBy3 div2
  let (div2_3_7 :: Word) = div2_3 * 7
  (div2_3_7_5 :: Word) <- divBy5 div2_3_7
  (return :: Word -> Either String Word) div2_3_7_5

main :: IO ()
main = do
  print $ divBy0 11 == Left "Can't div 11 by 0"
  print $ divBy2 1 == Left "Can't div 1 by 2"
  print $ divBy2 8 == Right 4
  print $ (return 12 >>= divBy2 >>= divBy3) == Right 2
  print $ (return 10 >>= divBy2 >>= divBy3) == Left "Can't div 5 by 3"
  print $ div30mul7 60 == Right 14
```