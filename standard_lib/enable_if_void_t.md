# std::enable_if_t против std::void_t

Шаблоны C++, начинавшиеся как облагороженная версия копипасты с помощью макроподстановок препроцессора, обросшие правилами SFINAE, породили довольно жуткие, громоздкие, но мощные возможности для метапрограммирования
и вычислений на этапе компиляции. 

Механизм крайне полезный и крайне неудобный. Не удивительно, что в стандартной библиотеке появились различные инструменты упрощения жизни.

Детали работы SFINAE выходят далеко за пределы данной серии заметок. Здесь же будет обсуждаться то, что появилось в C++17, что должно облегчить написание кода, но не работает.

Очень кратко, правило SFINAE (_substitution failure is not an error_) состоит в следующем:
- Если при подстановке аргументов в **заголовок** шаблона происходит ошибка (получается невалидная конструкция), то этот шаблон игнорируется. И берется следующий подходящий
- Если больше подходящих шаблонов нет, происходит ошибка компиляции.

Например:

```C++
#include <type_traits>

template <class T>
decltype(void(std::declval<typename T::Inner>())) fun(T) { // 1
        std::cout << "f1\n";    
}

template <class T>
decltype(void(std::declval<typename T::Outer>())) fun(T) { // 2
        std::cout << "f2\n";    
}

struct X {
    struct Inner {};
};

struct Y {
    struct Outer {};
};

...

fun(X{}); // при подстановке в шаблон 2, конструкция X::Outer невалидна: 
          // в X нет такого типа. Отбрасывается. Подстановка шаблона 1
          // проходит без ошибок — будет выведено «f1» 

fun(Y{}); // аналогично, но наоборот. Y::Inner не существует. Печатает «f2»
```

Конструкция `decltype(void(std::declval<typename T::Outer>))`, используемая для «паттерн матчинга», конечно же, ужасна. Сумрачный гений мастеров C++ порождал и более жуткие вещи. Но для менее искушенного пользователя хотелось бы чего-то более простого, понятного и удобного.

Так у нас есть `std::enable_if_t`, позволяющий триггерить SFINAE не по самописной жуткой конструкции, а по булеву значению.

```C++
template<class T>
std::enable_if_t<sizeof(T) <= 8> process(T) {
    std::cout << "by value";
}

template<class T>
std::enable_if_t<sizeof(T) > 8> process(const T&) {
    std::cout << "by ref";
}
...
process(5); // by value
const std::vector<int> v;
process(v); // by ref
```

Причем, в аргументе `std::enable_if` мы все также можем использовать страшные конструкции, а не только какие-то предикаты.

```C++
template <class T>
std::enable_if_t<std::is_same_v<typename T::Inner, typename T::Inner>> 
fun(T) { // 1
        std::cout << "f1\n";    
}

template <class T>
std::enable_if_t<std::is_same_v<typename T::Outer, typename T::Outer>> 
fun(T) { // 2
        std::cout << "f2\n";    
}

fun(X{}); // несмотря на то что значение std::is_same_v<T, T> всегда истинно, X::Outer не существует. И SFINAE сработает не из-за значения предиката, а из-за его аргументов. 
```

И тут начинатся первая неприятность:
`std::enable_if` против `std::enable_if_t`. 

```C++
// примерно
template <bool cond, T = void>
struct enable_if {};

template <true, T = void>
struct enable_if {
    using type = T;
};

template <bool cond, T = void>
using enable_if_t = typename enable_if<cond, T>::type;
```


Они легко путаются при быстром наборе с автокомплитом. Стоит случайно опустить суффикс `_t` и вместе с ним будут потеряны многие часы отладки всего этого сопоставляющего с образцами добра:

```C++
// SFINAE тригерилось от значения предиката и больше не работает.
// std::enable_if<false> -- валидный тип
// Получаем CE из-за переопределения одной и той же сущности по-разному
template<class T>
std::enable_if<sizeof(T) <= 8> process(T);
template<class T>
std::enable_if<sizeof(T) > 8> process(const T&);


// SFINAE тригерилось от аргументов предиката и продолжает работать.
// Если ожидали void в качестве типа возврата, может быть UB из-за отсутствующего return;
template <class T>
std::enable_if<std::is_same_v<typename T::Inner, typename T::Inner>> 
fun(T);
template <class T>
std::enable_if<std::is_same_v<typename T::Outer, typename T::Outer>> 
fun(T); 
```

Эта же неприятность касается всех остальных зверей из заголовка `<type_traits>`. Любой `std::trait_X` и `std::trait_X_t` — оба являются типами, будучи перепутанными далеко не всегда проявляют себя.

Лучше взять за правило: с помощью `std::enable_if` триггерить SFINAE только по предикату. Так проблем будет меньше.

Если предиката нет, его можно написать:

```C++
template <class T, 
          class = void> // костыль-placeholder для проверяемого «паттерна»
struct has_inner_impl : std::false_type {};

template <class T> 
struct has_inner_impl<T,
   decltype(void(std::declval<typename T::Inner>()))> // сам «паттерн», тип-результат должен совпадать с тем, что указан в заглушке выше
   : std::true_type {};

template <class T>
constexpr bool has_inner_v = has_inner_impl<T>::value;

static_assert(has_inner_v<X>);
static_assert(!has_inner_v<Y>);
```

Это один из наиболее распространенных и «простых» подходов к написанию подобных предикатов. И `void` чаще всего используется в качестве костыльной заглушки. И чтобы не писать постоянно этот страшный `decltype(void(std::declval<X>()))` каждый раз, когда нам нужно всего лишь проверить тип `X` на валидность, придумали, а потом и втащили в C++17, шаблон `std::void_t`.

Вот такой
```C++
template <class...>
using void_t = void;
```

И с ним все должно стать короче и красивее:
```C++
template <class T> 
struct has_inner_impl<T,
   std::void_t<typename T::Inner>> 
   : std::true_type {};
```

Но, увы, это чаще не работает, чем работает. В стандарте C++11 был обнаружен [дефект](https://wg21.cmeerw.net/cwg/issue1558), позволяющий этой конструкции не работать. И под многими не самыми-самыми новыми версиями компиляторов такой предикат всегда будет возвращать истину.

Но и под новыми версиями `std::void_t` также сломан.
Если мы попробуем использовать его, чтобы переписать самый-самый первый пример в этой заметке:

```C++
template <class T>
std::void_t<typename T::Inner> fun(T) {
    std::cout << "f1\n";
}

template <class T>
std::void_t<typename T::Outer> fun(T) {
    std::cout << "f2\n";
}
```
Ни один из трех основных компиляторов [не будет](https://godbolt.org/z/sW96vf) этот код компилировать. Несмотря на то, что первая версия с уродливым `decltype` [собиралась](https://godbolt.org/z/Wfe8eT).

Потому что у нас есть понятия «эквивалентности» и «функциональной эквивалентности» объявлений. Компилятор проверяет первое. А вот второе имеет отношение к SFINAE. [Страшная вещь](http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#1980).

Общий совет: не пользоваться `std::void_t`. А также не пытаться строить SFINAE на параметрах шаблонов-алиасов, если от этих параметров ничего не зависит справа от `unsing =`.

```C++
template <class T>
struct my_void {
    using type = void;
}

template <class T>
using my_void_t = void; // не работает

template <class T>
using my_void_t = typename my_void<T>::type; // ok
```

А вообще лучше переходить на C++20 и не заниматься всей это ерундой. Там специально для всех этих страшных конструкций читаемый синтаксический сахар придумали. Конечно, не без затаившихся граблей, но об этом в другой раз.

## Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/function_template
2. https://en.cppreference.com/w/cpp/types/void_t
