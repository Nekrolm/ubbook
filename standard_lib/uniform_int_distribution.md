# Нельзя так просто взять и сгенерировать случайную последовательность байт

В C++11 стандартная библиотека пополнилась многими отличными штуками, в том числе функциями и классами для работы с генераторами случайных чисел удобным и, как это ни странно, предсказуемым способом:

В наследство от C, С++ досталась функция `rand()`, которой на сегодняшний день не рекомендуется пользоваться нигде, кроме как совершенно игрушечных проектах:
- Если ее `seed` не инициализировать через `srand()`, он будет инициализирован, внезапно, единицей (а могли бы взять 42)
- Естественно она использует некий глобальное, возможно, thread local состояние. Это impementation defined
- В многопоточной среде thread safety тоже implementation defined. 
В общем, очень непредсказуемая штука!

Другое дело функционал из `#include <random>`!

Вот вам и разные генераторы, и преобразования функций распределения, и вы их можете друг от друга отделять и иметь свой собственный, особенно инициализированный генератор для каждой отдельной сущности в вашем проекте, и многопоточный доступ организовывайте как хотите. Красота!

А давайте сгенерируем последовательность случайных, равномерно распределенных байт.

```C++
#include <iostream>
#include <random>
 
int main()
{
    std::random_device rd;  // a seed source for the random number engine
    std::mt19937 gen(rd()); // mersenne_twister_engine seeded with rd()
    std::uniform_int_distribution<uint8_t> distrib{}; // don't try to use std::byte! it won't compile
 
    // Use distrib to generate random byte
    for (int n = 0; n != 10; ++n)
        std::cout << int32_t(distrib(gen)) << ' '; // we need a cast to print numeric values instead of characters
    std::cout << '\n';
}
```
Компилируем, запускаем, все [отлично работает](https://godbolt.org/z/xh1bY7PTq)!..
По крайней мере c GCC и Clang.

А что если нам нужна кроссплатформенная сборка в том числе под Windows с помощью MSVC?... А давайте попробуем просто взять и скомпилировать как есть...

[Ой](https://godbolt.org/z/Ka93s7sW6)

```
example.cpp
C:/data/msvc/14.39.33321-Pre/include\random(2107): error C2338: static_assert failed: 'invalid template argument for uniform_int_distribution: N4950 [rand.req.genl]/1.5 requires one of short, int, long, long long, unsigned short, unsigned int, unsigned long, or unsigned long long'
C:/data/msvc/14.39.33321-Pre/include\random(2107): note: the template instantiation context (the oldest one first) is
<source>(9): note: see reference to class template instantiation 'std::uniform_int_distribution<uint8_t>' being compiled
C:/data/msvc/14.39.33321-Pre/include\random(2107): error C2338: static_assert failed: 'note: char, signed char, unsigned char, char8_t, int8_t, and uint8_t are not allowed'
Compiler returned: 2
```

Ну-у, матерый разработчик, знакомый с причудами старых версий MSVC, который с по умолчанию включенным `\permissive+` мог компилировать совершенно безумные вещи, не сильно удивится и скажет, что опять Microsoft какую-то ерунду придумали просто...

Удивительно, [но нет](https://eel.is/c++draft/rand#req.genl-1.6)!

```
Throughout this subclause [rand], the effect of instantiating a template:
...
that has a template type parameter named UIntType is undefined unless the corresponding template argument is cv-unqualified and is one of unsigned short, unsigned int, unsigned long, or unsigned long long.
```

Подставлять `uint8_t` в `std::uniform_int_distribution` стандартом не разрешается под страхом неопределенных эффектов.

Это, конечно, совершенно нелепо, и в 2013 году даже поднимался вопрос о принятии [defect report](https://cplusplus.github.io/LWG/issue2326), но его отклонили как not a defect и предложили написать proposal в стандарт, чтобы как-то это дело исправить... В общем по состоянию на 2024 год не исправили. Возможно, исправят в C++26. Или в C++29.

Кстати, на всякий случай, `__int128_t` тоже использовать как бы нельзя. [Хотя с GCC работает.](https://godbolt.org/z/9Mxrx6aEr)

# Полезные ссылки
1. [Обсуждение на reddit](https://www.reddit.com/r/cpp/comments/1czwa5h/is_instantiating_stduniform_int_distributionuint8/)
2. [cppreference uniform_int_distribution](https://en.cppreference.com/w/cpp/numeric/random/uniform_int_distribution)

