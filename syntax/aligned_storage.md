# std::aligned_storage

Всех С++ разработчиков можно довольно успешно разделить на две категории:

- Те, у кого код вида
```C++
char buff[sizeof(T)];
...
T* obj = new (buff) T(...);
```
работает и для них нет никаких проблем
- И те, у кого из-за такого кода внезапно прилетает SIGSEGV или SIGBUS, или что-нибудь еще более интересное.

Чаще всего первые собирают свои программы только под x86, каким-нибудь старым компилятором, ничего не знающим про SIMD инструкции, и с, вероятно, выключенными "агрессивными" оптимизациями, чтоб точно ничего не сломалось.

C разработчики тоже не должны уйти обиженными: для них этот код будет выглядеть как
```C
char buff[sizeof(T)];
...
T* obj = (T*)buff;
```
Что в принципе еще страшнее из-за неинициализированной памяти, но это уже другая проблема.


Основная проблема: буфер, в который мы тут собрались что-то записать, может быть выровнен (_alignment_) не так как это требуется для типа `T`.

О том что размер укладываемых в буфер данных должен быть не больше размера самого буфера многие узнают довольно быстро и также быстро вспоминают. Хотя бы из соображений здравого смысла. А вот с выравниванием всё сложно.

Чтобы бездушная машина могла успешно прочитать/записать данные типа `T` по адресу соответствующему значению указателя `T* ptr`, или иногда совершить какую-то хитрую операцию над ними, в общем случае адрес должен быть выровнен — кратен некоторому числу (обычно это степень двойки), которое нам навязали инженеры-разработчики микроархитектуры этой самой машины. А навязали потому что:
- Так должно быть эффективнее
- По-другому не получалось
- Надо было очень сильно сэкономить на длине инструкции (чем больше выравнивание, тем больше младших битов адреса можно не использовать)
- Добавьте свою причину, если вы проектируете набор инструкций и знаете что еще можно придумать.

Если мы вернемся в C++, то обычно мы знаем выравнивание, требуемое для встроенных типов:

| Type | alignment |
|------|-----------|
| `char`  | 1 |
| `int16` | 2 |
| `int32` | 4 |
| `int64` | 8 |
| `__m128i`| 16 |

Для других типов специализированных для работы с SSE/SIMD инструкциями может быть и больше.

Для пользовательских структур и классов — наследуется наибольшее выравнивание из всех полей. А между полями появляются неявные байты-заполнители (_padding_) чтобы удовлетворять требованиям выравнивания каждого поля по отдельности.

```C++
struct S {
    char c;    // 1
    // неявно char _padding[3] 
    int32_t i; // 4
};

static_assert(alignof(S) == 4);
```
Выравнивание массива соответствует выравниванию его элементов.

Поэтому

```C++
char buff[sizeof(T)]; // alignment == 1
...
T* obj = new (buff) T(...); // (uintptr_t)(obj) должен быть кратен alignof(T)
// но в этом коде гарантируется только то что он кратен 1
```
Для встроенных типов на x86 доступ по невыровненным указателям чаще всего приводит к просто к более медленному исполнению. На других платформах — может быть segfault.
Но с SSE типами и на x86 можно легко получить segfault и [довольно красиво](https://godbolt.org/z/5ja68bnsP)

```C++
#include <memory>
#include <xmmintrin.h>


const size_t head = 0;
struct StaticStorage {
char buffer[256];
} storage;

int main() {
    __m128i* a = new (storage.buffer) __m128i();
    // comment line above & uncomment any line below for segfault
    // __m128i* b = new (storage.buffer + 0) __m128i();
    // __m128i* c = new (storage.buffer + head) __m128i();
}
```

Что же делать?! Как же написать код без такого интересного неопределенного поведения? Не волнуйтесь, С++11 спешит на помощь!

Стандарт предоставляет `alignas` спецификатор. С его помощью можно явно указать требования к выравниванию при описании переменных и структур

```C++
#include <memory>
#include <xmmintrin.h>


const size_t head = 0;
struct StaticStorage {
    alignas(__m128i) char buffer[256];
} storage;

int main() {
    __m128i* a = new (storage.buffer) __m128i();
    __m128i* b = new (storage.buffer + 0) __m128i();
    __m128i* c = new (storage.buffer + head) __m128i();
}
```
И вот уже и [не падает](https://godbolt.org/z/o3G9x1nbv)

Но выглядит как-то громоздко. Неужели для такого важного случая, как создание буфера с подходящим размером и выравниванием, в стандартной библиотеке C++ нет никакой удобной функции?

Конечно есть!

`std::aligned_storage` и его старший брат `std::aligned_union`

```C++
template<std::size_t Len, std::size_t Align = /* default alignment not implemented */>
struct aligned_storage
{
    struct type
    {
        alignas(Align) unsigned char data[Len];
    };
};

template <std::size_t Len, class... Types>
struct aligned_union
{
    static constexpr std::size_t alignment_value = std::max({alignof(Types)...});
 
    struct type
    {
      alignas(alignment_value) char _s[std::max({Len, sizeof(Types)...})];
    };
};
```

Это ж практически то же самое, что было в примере выше! 
Первый совсем низкоуровневый — ему нужно напрямую число-значение выравнивания указать.
А второй более умный — он сам подходящее значение по списку типов выберет. Да еще и размер буфера подстроит, если мы неправильный указали. Какая удобная метафункция!

Давайте же ею воспользуемся.
```C++
#include <memory>
#include <xmmintrin.h>
#include <type_traits>


const size_t head = 0;
std::aligned_union<256, __m128i> storage;

int main() {
    __m128i* a = new (&storage) __m128i();
    __m128i* b = new ((char*)(&storage) + 0) __m128i();
    __m128i* c = new ((char*)&storage + head) __m128i();
}
```
И сразу же все [упало](https://godbolt.org/z/aG18PzsoT).

Но как же так?! Ведь все же верно...

А давайте [проверим](https://godbolt.org/z/dc55976ah)
```C++
static_assert(sizeof(storage) >= 256);
```

```
<source>:9:1: error: static assertion failed due to requirement 'sizeof (storage) >= 256'
static_assert(sizeof(storage) >= 256);
^             ~~~~~~~~~~~~~~~~~~~~~~
<source>:9:31: note: expression evaluates to '1 >= 256'
static_assert(sizeof(storage) >= 256);
```

Дивно. Но если мы еще раз внимательно посмотрим на примеры определения шаблонов `std::aligned_storage` выше, то обнаружим великую подлость и предрасположенность этих шаблонов к ошибке использования.

Нам нужно использовать
`typename std::aligned_union<256, __m128i>::type storage`!

Или, в С++17,
`std::aligned_union_t<256, __m128i> storage`

Разница всего в два символа, а какой [результат!](https://godbolt.org/z/shdjnKdPK)

На момент написания этой заметки, gcc-12 способен из коробки выдать предупреждения
```
<source>:12:23: warning: placement new constructing an object of type '__m128i' and size '16' in a region of type 'std::aligned_union<256, __vector(2) long long int>' and size '1' [-Wplacement-new=]
   12 |     __m128i* a = new (&storage) __m128i();
```
наводящие на мысль об ошибке.

clang-16 по умолчанию такого не сообщает.

`std::aligned_*` признаны невероятно опасными к использованию. Из-за ужасного дизайна, ошибки в использовании которого очень легко прячутся.

В C++23 их пометили как deprecated. Но кто ж знает, когда мы увидим C++23 в больших и старых кодовых базах...

Если вы используете `std::aligned_*` в своем коде — убедитесь дважды, что вы используете его правильно. А лучше замените на свою структуру с явным использованием `alignas`.

## Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/alignas
2. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p1413r3.pdf
3. https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions
