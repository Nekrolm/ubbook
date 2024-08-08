# Типы «нулевого» размера

В С++ при определении собственных классов и структур никто нам не запрещает не указывать ни одного поля, оставляя структуру пустой:

```C++
struct MyTag {};
```

Конечно же, мы можем не только объявлять пустые структуры, но и создавать объекты этих типов

```C++
struct Tag {};

Tag func(Tag t1) {
    Tag t2;
    return Tag{};
}
```
Возможности несомненно полезные и широко используемые:

- Для определения абстрактного статического или динамического полиморфного интерфейса
- Для введения тегов выбора нужной перегрузки
- Для определения различных предикатов и метафункций над типами

А давайте сыграем в игру? Я буду показывать вам разные определения структур, а вы постараетесь угадать их размеры в байтах (`sizeof`). Начинаем?

```C++
struct StdAllocator {};

struct Vector1 {
    int* data;
    int* size_end;
    int* capacity_end;
    StdAllocator alloc;
};

struct Vector2 {
    StdAllocator alloc;
    int* data;
    int* size_end;
    int* capacity_end;
};

struct Vector3 : StdAllocator {
    int* data;
    int* size_end;
    int* capacity_end;
};
```

[Угадали?](https://godbolt.org/z/MEETs4csc)

`Vector1` и `Vector2` имеют размеры `4*sizeof(int*)`.
Но как же так?! Откуда берутся `3*sizeof(int*)` совершенно очевидно. Но четвертый-то откуда?!

Все очень просто: в C++ не бывает структур нулевого размера. И потому размер пустой структуры: `sizeof(StdAllocator) == 1`

Но `sizeof(int*) != 1`. По крайней мере на x86. А это еще проще: выравнивание и паддинг. `Vector1` добивается байтами в конец, чтобы его размер был кратен выравниванию первого поля. А в `Vector2` добиваются дополнительные байты между `alloc` и `data`, чтобы смещение до `data` было кратным его выравниванию. Все очень просто и очевидно! Если же вам, как и многим другим людям, которые не задаются подобными вопросами каждый день, не очевидно наличие паддинга в той или иной структуре, то советую [использовать](https://godbolt.org/z/a8xof3eqG) флаг компилятора `-Wpadded` для GCC/Clang.

Хорошо, мы разобрались с `Vector1` и `Vector2`. А что там с `Vector3`? Тоже `4*sizeof(int*)`? Ведь мы же знаем, что подобъект базового класса должен быть где-то размещен, а его размер, как мы выяснили, не нулевой...
А вот и нет! Размер `Vector3` равен `3*sizeof(int*)`! Но как же так?! А это называется EBO [(empty base optimization)](https://en.cppreference.com/w/cpp/language/ebo).

Интересный zero-cost! Для сравнения, можно глянуть на аналогичные пустые структуры в Rust. Там их размер [может быть](https://godbolt.org/z/r9YTKrbb3) равен нулю.

Ну ладно, мы выяснили, что, неаккуратно использовав пустые структуры, мы можем получить увеличение потребления памяти. Давайте играть дальше.

```C++
struct StdAllocator {};
struct StdComparator {};

struct Map1 {
    StdAllocator alloc;
    StdComparator comp;
};

struct Map2 {
    StdAllocator alloc;
    [[no_unique_address]] StdComparator comp;
};

struct Map3 {
    [[no_unique_address]] StdAllocator alloc;
    [[no_unique_address]] StdComparator comp;
};

struct MapImpl1 : Map1 {
    int x;
};

struct MapImpl2 : Map2 {
    int x;
};

struct MapImpl3 : Map3 {
    int x;
};
```

Чему равны размеры `Map1`, `Map2`, `Map3`?

Ну, тут все просто!:
- Очевидно, что `sizeof(Map1) == 2`, ведь она состоит из двух пустых структур, каждая из которых имеет размер 1.
- Благодаря атрибуту `[[no_unique_address]]` из стандарта C++20 (Clang поддерживает с C++11), `Map2` и `Map3` должны иметь размер 1. В `Map3` оба поля разделяют общий адрес. В `Map2` то же самое. Да и меньше чем 1 не бывает.

Хорошо. А что же теперь с наследующими структурами?

Все по `2*sizeof(int)`? А вот и нет: у `MapImpl3` [работает EBO](https://godbolt.org/z/exo5zP64E)!

Ну ладно. В этом есть какая-то логика и закономерность. Это еще можно принять. Хотя... На самом деле вы были правы! Ведь если у вас компилятор msvc, то `[[no_unique_address]]` просто [не работает](https://godbolt.org/z/6364qzYe6). И не будет работать. Потому что msvc долгое время просто игнорировал незнакомые ему атрибуты. И если поддержать `[[no_unique_address]]`, то сломается бинарная совместимость. Используйте `[[msvc::no_unique_address]]`! EBO, правда, пока [не работает](https://godbolt.org/z/GTjrsbPPK).


## zero-size array

Язык C (не С++), начиная с версии стандарта 99, позволяет использовать следующую любопытную конструкцию:

```C
struct ImageHeader{
    int h;
    int w;
};

struct Image {
    struct ImageHeader header;
    char data[];
};
```

Поле `data` в структуре `Image` имеет [нулевой размер](https://godbolt.org/z/d3xfdj3Ke). Это FAM (flexible array member). Очень удобная штука, чтобы получать доступ к массиву статически не известной длины, размещенному сразу после некоторого заголовка в бинарном буфере. Длина массива обычно указывается в самом заголовке. FAM может быть только последним полем в структуре.

Стандарт C++ такие фичи не разрешает. Но ведь есть GCC с его нестандартными включенными по умолчанию расширениями.

Что будет если сделать так?
```C++
struct S {
    char data[];
};
```
Чему будет равен размер структуры `S`?

В стандартном C пустые структуры в принципе запрещены. И поведение программы с ними не определено. GCC определяет их размер нулевым при компиляции C программ. А при компиляции C++ — размер, как мы выяснили ранее, единичный. Дело пахнет страшными багами и ночными кошмарами при неосторожном проектировании C++ библиотек с сишным интерфейсом или использованием C-библиотек в C++!

Но вернемся все-таки к нашей структуре с FAM. Поле в ней есть. Стандартный C опять-таки требует, чтобы было еще хотя бы одно поле ненулевой длины перед FAM. GNU C же охотно сделает нам структуру нулевого размера.

А теперь [посмотрим](https://godbolt.org/z/osrzYbcsj) на GCC C++.

```C++
struct S1 {
    char data[];
};

struct S2 {};

static_assert(sizeof(S1) != sizeof(S2));
static_assert(sizeof(S1) == 0);
```

И вот уже внезапно у нас в C++ структуры нулевого размера. Только C++ не стандартный.
Каким образом такие структуры будет взаимодействовать с EBO — нужно читать в спецификации к GCC.

## Не совсем пустые структуры

Кстати об empty base optimization, на cppreference можно обнаружить сноску, что *oбычно* при наследовании от пустой структуры размер структуры-наследника не увеличивается. *Обычно*.

Но вот и пример, когда происходит что-то необычное
```C++
#include <type_traits>

struct EBO {};
struct A : EBO {};
struct B : EBO {};
struct C : EBO {};

static_assert(std::is_empty_v<A>);
static_assert(std::is_empty_v<B>);
static_assert(std::is_empty_v<C>);

struct D : A, B, C {};
static_assert(std::is_empty_v<D>);
static_assert(sizeof(D) == 3);
``` 
[Под MSVC размер структуры `D` равен 1. А под Clang и GCC -- 3](https://godbolt.org/z/jqEPEvaYd).
MSVC при этом, согласно стандарту, не прав, хотя его вариант почти всегда был бы предпочтителен
```
Empty base optimization is prohibited if one of the empty base classes is also the type or the base of the type of the first non-static data member, since the two base subobjects of the same type are required to have different addresses within the object representation of the most derived type.
```

## tag dispatching

Мы видели, что неаккуратное использование пустых структур приводит к увеличению размера других, не пустых структур.
А может еще есть какие-то подводные камни? Например, при использовании пустых структур-тегов для выбора перегрузки?

Есть ли разница между
```C++
struct Mul {};
struct Add {};

int op(Mul, int x, int y) {
    return x * y;
} 

int op(Add, int x, int y) {
    return x + y;
}
```

и

```C++
int mul(int x, int y) {
    return x * y;
} 

int add(int x, int y) {
    return x + y;
}
```
в плане генерируемого кода?


Краткий ответ: да. Есть разница. Зависит от конкретной имплементации. Стандарт не гарантирует оптимизацию пустых аргументов.
От перемены позиций тегов может меняться бинарный интерфейс. Поиграться с наиболее заметными изменениями можно на примере [msvc](https://godbolt.org/z/E68ojMb8f).


## Полезные ссылки
1. https://devblogs.microsoft.com/cppblog/msvc-cpp20-and-the-std-cpp20-switch
2. https://en.cppreference.com/w/cpp/language/attributes/no_unique_address
3. https://en.cppreference.com/w/cpp/language/ebo
