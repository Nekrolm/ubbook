# Как передать стандартную функцию и ничего не сломать

Предположим, вам нужно заниматься какими-то вычислениями по долгу службы (или вы просто студент и вам кровь из носу надо выполнить задание по численным методом). 

И вы взяли готовую прекрасную функцию интегрирования:

```C++
template <class F>
concept NumericFunction = std::is_invocable_v<F, float> && requires (float arg, F f) {
    { f(arg) } -> std::convertible_to<float>;
};

float integrate(NumericFunction auto f)  {
    float sum = 0;
    /*Мы не будем вдаваться в подробности точности, сходимости, и шагов разбиения и выбора точки, хотя это тоже очень важно, но в другой книжке*/
    for (float x : std::views::iota(1, 26)) {
        sum += f(x);
    }
    return sum;
}
```

Отлично. Вы начинаете ее тестировать на какой-нибудь стандартной функции

```C++
#include <cmath>
...
int main() {
    return integrate(sqrt); // все ок? (приведение к int считаем нормальным)
}
```
[Вроде да](https://godbolt.org/z/o8oG48z3e).

На самом деле нет. И проблем тут как минимум две

1. Стандартная библиотека C++ содержит в себе стандартную библиотеку C, что делает использование стандартных математических функций особенно болезненным:

```C++
static_assert(std::abs(5.8) > 5.5);
static_assert(abs(5.8) > 5.5);

//---------------------

<source>:26:24: error: static assertion failed
   26 | static_assert(abs(5.8) > 5.5);
      |               ~~~~~~~~~^~~~~
<source>:26:24: note: the comparison reduces to '(5.0e+0 > 5.5e+0)'
Compiler returned: 1
```

Окей. Мы поняли. Надо использовать `std::sqrt`, чтоб не попасть не в ту перегрузку

```C++
int main() {
    return integrate(std::sqrt);
}

// ---------------

<source>:22:21: error: no matching function for call to 'integrate(<unresolved overloaded function type>)'
   22 |     return integrate(std::sqrt);
```

Ага. overloaded function type. И как же нам тогда выбрать нужный?

Вы вышли в гугл с этим вопросом и первая ссылка привела вас на какой-нибудь Qt-форум (о, в Qt это распространенная проблема -- указать перегрузку при соединении сигналов и слотов), и самый релевантный ответ был: соорудить явное приведение типов указателей на функцию

```C++
int main() {
    return integrate(static_cast<float(*)(float)>(&std::sqrt));
}
```

[Ура работает!](https://godbolt.org/z/zqhM98Wf7);

Поздравляю...

2. ...Вы нарушили пункт [16.4.5.2.6](https://eel.is/c++draft/namespace.std#6) ~~и ваша программа будет оштрафована~~

> Let F denote a standard library function ([global.functions]), a standard library static member function, or an instantiation of a standard library function template. Unless F is designated an addressable function, the behavior of a C++ program is unspecified (possibly ill-formed) if it explicitly or implicitly attempts to form a pointer to F.


Вызов `integrate(static_cast<float(*)(float)>(&std::sqrt));` делает именно это. Вы взяли указатель на функцию. Указатели почти на любую функцию стандартной библиотеки брать нельзя.

Изначальный вариант с `return integrate(sqrt)`, использующий `sqrt` из библиотеки C также попадает в эту ловушку, только неявно. 

А с C++20 грозятся, что может перестать компилироваться, но я пока примера тому не видел.

Почему нельзя?

**А кто вам сказал, что это функция?**

Да, почти все функции стандартной библиотеки C++17, после инстанциирования шаблонов, все-таки оказываются нормальными функциями и потому у нас уж сколько лет все работает.

C функциями стандартной библиотеки C все, конечно, хуже -- они могут быть макросами, и черт его знает от чего вы на самом деле взяли адрес в таком случае.

С C++20 (вдохновленные ranges [Эрика Ниблера](https://github.com/ericniebler)) новые (а также потенциально старые после перехода std на модули) функции внезапно могут оказаться *ниблоидами*. Глобальными объектами с определенным `operator()` -- так что они могут выглядеть и крякать как старые добрые функции, но таковыми не быть. 
И если вы использовали `С-style` каст вместо громоздкого `static_cast`, то вас могут ждать интересные результаты:

```C++
// https://godbolt.org/z/98Y6zv6nj
// старая версия
// float f(float a) {
//     return a;
// }

// вы обновились и теперь это ниблоид!
auto f = [](float a) -> float {
    return a;
};

int main() {
    return integrate((float(*)(float))(&f));
    // Segfault
}
```

Положение могло бы спасти отсутствие `&` перед именем функции (для функций и лямбд применяется неявный decay к указателю):

```C++
// https://godbolt.org/z/KnP8q7e3r
// float f(float a) {
//     return a;
// }

auto f = [](float a) -> float {
    return a;
};

int main() {
    return integrate((float(*)(float))(f));
    // компилируется и работает
}

// -----------------------------

// https://godbolt.org/z/fqzdse1Ya

// ниблоиды в std чаще определяются так, а не с помощью лямбд
struct {
    static float operator()(float x) {
        return x;
    } 
} f;

int main() {
    // не компилируется и нам ужасно повезло что это так!
    return integrate((float(*)(float))(f));
}

```

Но, к сожалению, примеров кода с явным взятием адреса функции очень много в мире.



## Что же делать?

Хорошая новость: если когда-нибудь вся замечательная гора стандартных функций станет вызываемыми объектами,
ваш `integrate(std::sqrt)` будет компилироваться и работать правильно из коробки. И все будут счастливы.

Плохая новость: замечательно не будет, поэтому придется писать код

Проблема решится оборачиванием вызова к std функции в вашу функцию или в лямбду.

```C++
int main() {
    return integrate([](float x) {
        return std::sqrt(x);
    });
}
```

или даже можно завести вспомогательный макрос (с C++20 он выглядит чуть менее страшно чем обычно)

```C++
// https://godbolt.org/z/WPM9dx8Yx
#define LAMBDA_WRAP(f) []<class... T>(T&&... args) \
 noexcept(noexcept(f(std::forward<T>(args)...))) -> decltype(auto) \
 { return f(std::forward<T>(args)...); }

int main() {
    return integrate(LAMBDA_WRAP(std::sqrt));
}
```

Причем вариант с лямбдой, а не с функцией будет почти всегда предпочтительнее по соображениям оптимизаций. Смотрите:

Если вызов шаблонной функции `integrate` по какой-либо причине не может быть заинлайнен компилятором и вы передаете указатель на функцию, компилятор не имеет никакого выбора кроме как честно генерировать call по этому указателю:

```C++
// https://godbolt.org/z/7j68n6njq
#define LAMBDA_WRAP(f) []<class... T>(T&&... args) noexcept(noexcept(f(std::forward<T>(args)...))) -> decltype(auto) { return f(std::forward<T>(args)...); }

float my_sqrt(float f) {
    return std::sqrt(f);
}

int main() {
    return integrate(my_sqrt) + integrate(LAMBDA_WRAP(std::sqrt));
}

/*
// C функцией
float integrate<float (*)(float)>(float (*)(float)):
        push    rbp
        mov     rbp, rdi
        ...
.L24:
        pxor    xmm0, xmm0
        cvtsi2ss        xmm0, ebx
        add     ebx, 1
        call    rbp  // ! нет информации о функции -- вызов по указателю
        addss   xmm0, DWORD PTR [rsp+12]
        movss   DWORD PTR [rsp+12], xmm0
        cmp     ebx, 26
        ...
        ret
*/
// C лямбдой
/*
float integrate<main::{lambda<typename... $T0>(($T0&&)...)#1}>(main::{lambda<typename... $T0>(($T0&&)...)#1}) [clone .isra.0]:
        ...
.L16:
        pxor    xmm0, xmm0
        cvtsi2ss        xmm0, ebx
        ucomiss xmm2, xmm0
        ja      .L21
        sqrtss  xmm0, xmm0 // ! sqrt подставлен
        add     ebx, 1
        addss   xmm1, xmm0
        cmp     ebx, 26
        jne     .L16
.L11:
        ...
.L21:
        movss   DWORD PTR [rsp+12], xmm1
        add     ebx, 1
        call    sqrtf  /// ! sqrt подставлен
        ...
*/
```

Почему лямбда почти всегда предпочтительнее, но не всегда?

GCC и Clang, например, генерирует копию кода для каждого вызова с лямбдой, даже если они одинаковые. Ну просто потому что так необходимо: у каждой лямбды должен быть уникальный тип.
```C++
// https://godbolt.org/z/8jazdx5n7
int main() {
    return integrate(my_sqrt) +
           integrate(LAMBDA_WRAP(std::sqrt)) + 
           integrate(LAMBDA_WRAP(std::sqrt)) +
           integrate(LAMBDA_WRAP(std::sqrt));
}
```
Что поделать, раздутие кода -- известный результат [мономорфизации](https://en.wikipedia.org/wiki/Monomorphization) шаблонов/generic функций.

Переиспользуйте лямбду, и будет лучше:

```C++
// https://godbolt.org/z/h14ceaaYc
// сгенерированный код в 2 раза меньше чем для примера выще
int main() {
    auto sqrt_f = LAMBDA_WRAP(std::sqrt);
    return integrate(my_sqrt) +
           integrate(sqrt_f) + 
           integrate(sqrt_f) +
           integrate(sqrt_f);
}
```

# Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/extending_std#Designated_addressable_functions
2. Rust Functions Are Weird (But Be Glad) - https://www.youtube.com/watch?v=SqT5YglW3qU


