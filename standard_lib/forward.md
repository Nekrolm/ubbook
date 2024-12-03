# Как неправильно использовать std::forward

Однажды я увидел код, который демонстрирует красоту и удобство concepts в C++20/23 и мощь шаблонов. Код их действительно демонстрировал. Давайте я его покажу.

```C++
template<uint32_t Rows, uint32_t Cols, typename F>
struct Matrix {
    F generator;
    auto operator[](uint32_t r, uint32_t c) {
        return generator(r, c);
    }
};

// Make matrix via 2 args generator function
template <uint32_t Rows, uint32_t Cols>
auto MakeMatrix(std::invocable<uint32_t, uint32_t> auto&& fn) {
    using F = std::remove_cvref_t<decltype(fn)>;
    return Matrix<Rows, Cols, F>(std::forward<F>(fn));
}
```

Красиво, неправда ли? Давайте его протестируем

```C++
int main() {
    std::function<int(uint32_t, uint32_t)> generator = [](auto...) {
        return 42;
    };
    auto m1 = MakeMatrix<3, 3>(generator);
    std::cout << "m1[1,1]=" << m1[1,1] << std::endl;
    auto m2 = MakeMatrix<2, 2>(generator);
    std::cout << "m2[1,1]=" << m2[1,1] << std::endl;
}
```

Вы могли бы подумать, что обе операции вывода успешно напечатают число 42...
Но произойдет кое-что [неожиданное](https://godbolt.org/z/6TKEP1caE)!

```
Program returned: 139
terminate called after throwing an instance of 'std::bad_function_call'
  what():  bad_function_call
Program terminated with signal: SIGSEGV
m1[1,1]=42
```

```
==1==ERROR: AddressSanitizer: SEGV on unknown address (pc 0x7a3c0ee28898 bp 0x7a3c0f01be90 sp 0x7ffe7360aa00 T0)
==1==The signal is caused by a READ memory access.
==1==Hint: this fault was caused by a dereference of a high value address (see register values below).  Disassemble the provided pc to learn which register was used.
    #0 0x7a3c0ee28898 in abort (/lib/x86_64-linux-gnu/libc.so.6+0x28898) (BuildId: 490fef8403240c91833978d494d39e537409b92e)
    #1 0x7a3c0f3abbfc  (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0xadbfc) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #2 0x7a3c0f3bd169  (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0xbf169) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #3 0x7a3c0f3ab7a8 in std::terminate() (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0xad7a8) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #4 0x7a3c0f3bd3e6 in __cxa_throw (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0xbf3e6) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #5 0x7a3c0f3ae6c3 in std::__throw_bad_function_call() (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0xb06c3) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #6 0x401256 in std::function<int (unsigned int, unsigned int)>::operator()(unsigned int, unsigned int) const /opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/bits/std_function.h:590
    #7 0x401256 in Matrix<2u, 2u, std::function<int (unsigned int, unsigned int)> >::operator[](unsigned int, unsigned int) /app/example.cpp:12
    #8 0x401256 in main /app/example.cpp:31
```


Все совершенно определенно и соответствует спецификации. `operator()` у `std::function` бросает исключение, если объект-функция оказался пустым... Да, если вы не знали, `std::function` это неявно nullable тип.
Но какого черта объект оказался пустым?!

Посмотрим еще раз на эти две прекрасные строчки

```C++
auto MakeMatrix(std::invocable<uint32_t, uint32_t> auto&& fn) {
    using F = std::remove_cvref_t<decltype(fn)>;
    return Matrix<Rows, Cols, F>(std::forward<F>(fn));
}
```
Шаблон принимает на вход так называемую «универсальную» ссылку. И использует `std::forward` чтобы передать ее дальше «универсальным» способом: передай rvalue как rvalue, lvalue как lvalue.
Вот только программист решил для красоты и читаемости сэкономить на буквах и использовал его неправильно.

Тип-параметр у шаблона `std::forward` — обязателен и его нужно указать правильно. Им должен быть тип ссылки, который мы хотим сохранить и прокинуть далее.

Но разработчик подсунул туда `using F = std::remove_cvref_t<decltype(fn)>;` То есть буквально отбросил ссылку.
И если мы посмотрим на объявление `std::forward`

```C++
template< class T >
constexpr T&& forward( typename std::remove_reference<T>::type& t ) noexcept;
template< class T >
constexpr T&& forward( typename std::remove_reference<T>::type&& t ) noexcept;
```

Станет понятно, что именно пошло не так: бессылочный `T` всегда превращается в `T&&` rvalue-ссылка! Вместо `std::forward` разработчик получил `std::move`. А мы получили use-after-move.

Универсально корректное использование `std::forward` выглядит так:

```C++
std::forward<delctype(value)>(value)
```

Либо, конечно, можно использовать явный параметр шаблона

```C++
template <uint32_t Rows, uint32_t Cols, std::invocable<uint32_t, uint32_t> Gen>
auto MakeMatrix(Gen&& fn) {
    using F = std::remove_cvref_t<Gen>;
    return Matrix<Rows, Cols, F>(std::forward<Gen>(fn));
}
```
Но такой вариант не защищен от ошибок при рефакторинге.

----
Я не большой фанат макросов, но конкретно для этого случая крайне рекомендую завести макрос
```C++
#define FORWARD(x) ::std::forward<decltype(x)>(x)
```
И никогда больше не допускать ошибку.

В C++23 в стандартную библиотеку добавили еще функцию `std::forward_like`, чтобы навешивать на ваш объект ссылку той же категории, как у другого объекта. Соответствущий макрос может выглядеть так

```C++
#define FORWARD_LIKE(target, value) ::std::forward_like<decltype(target)>(value)
```


## Полезные ссылки
1. https://en.cppreference.com/w/cpp/utility/forward
2. https://en.cppreference.com/w/cpp/utility/forward_like

