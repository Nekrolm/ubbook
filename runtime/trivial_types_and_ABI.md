# Тривиальные типы и ABI

Допустим, вы написали прекрасную библиотеку для работы с двумерными векторами. 
И там, конечно, же была структура `Point`. Вот такая:

```C++
template <typename T>
struct Point {
    ~Point() {}

    T x;
    T y;
};
```

И была функция

```C++
Point<float> zero_point();
```

имплементацию которой вы, как приличный разработчик, заботящийся о времени компиляции и размерах заголовочных файлов, поместили в компилируемый .cpp файл, а пользователю оставили только объявление.
Ваша библиотека была такой хорошей, что быстро обрела популярность и от нее стали зависеть многие другие приложения.

И все было хорошо. Но однажды вы заметили, что деструктор `Point` вам совершенно не нужен. Пусть его генерит компилятор самостоятельно. И вы его удалили.

```C++
template <typename T>
struct Point {
    T x;
    T y;
};
```

Пересобрали библиотеку и разослали пользователям новые готовые .dll/.so файлы. С новыми заголовками, конечно же.
Но изменение такое незначительное, что пользователи просто подложили готовые бинари себе без перекомпиляции... И все упало со страшным memory corruption.

Почему?

Это измение сломало ABI.

В С++ все типы делятся на тривиальные и нетривиальный. Тривиальные, в свою очередь, бывают еще и в разных аспектах тривиальными. В общем случае тривиальность позволяет не генерировать дополнительный код, чтобы что-то сделать.

- `trivially_constructible` — не надо ничего инициализировать
- `trivially_destructible` — не нужно генерировать код для деструктора
- `trivially_copyable` — не нужно ничего, кроме простого копирования байтов
- `trivially_movable` — аналогично копированию, но при «перемещении»

С объектами тривиальных типов выполнимы дополнительные оптимизации. Например, их можно передавать через регистры. Компилятор способен догадаться соптимизировать memcpy (использованный чтобы избежать неопределенного поведения) в `reinterpret_cast`. И другие подобные вещи. 

Вот, например:

```C++
struct TCopyable {
    int x;
    int y;
};
static_assert(std::is_trivially_copyable_v<TCopyable>);

struct TNCopyable {
    int x;
    int y;

    TNCopyable(const TNCopyable& other) : x{other.x}, y{other.y} {}

    // вынуждены написать конструктор, так как aggregate initialization
    // отключился из-за конструктора копирования
    TNCopyable(int x, int y) : x{x}, y{y} {}
};

static_assert(!std::is_trivially_copyable_v<TNCopyable>);

// Здесь будет возврат через регистр rax. TCopyably в него как раз помещается
extern TCopyable test_tcopy(const TCopyable& c) {
    return {c.x *5, c.y * 6};
} 

// Здесь возврат через указатель, передаваемый через регистр rdi
extern TNCopyable test_tnocopy(const TNCopyable& c) {
    return {c.x *5, c.y * 6};
} 
```

По ассемблерному листингу можно убедиться, что две «одинаковые» функции, возвращающие «одинаково» представленные в памяти структуры, делают это [по-разному](https://godbolt.org/z/Mz8srfdsc)

```asm
test_tcopy(TCopyable const&):             # @test_tcopy(TCopyable const&)
        mov     eax, dword ptr [rdi]
        mov     ecx, dword ptr [rdi + 4]
        lea     eax, [rax + 4*rax]
        add     ecx, ecx
        lea     ecx, [rcx + 2*rcx]
        shl     rcx, 32
        or      rax, rcx  #!
        ret
test_tnocopy(TNCopyable const&):         # @test_tnocopy(TNCopyable const&)
        mov     rax, rdi
        mov     ecx, dword ptr [rsi]
        mov     edx, dword ptr [rsi + 4]
        lea     ecx, [rcx + 4*rcx]
        add     edx, edx
        lea     edx, [rdx + 2*rdx]
        mov     dword ptr [rdi], ecx     #!
        mov     dword ptr [rdi + 4], edx #!
        ret
```

[Аналогично](https://godbolt.org/z/KK1o5E168) и с вашей 2D точкой:

```C++
struct TPoint {
    float x;
    float y;
};
static_assert(std::is_trivially_destructible_v<TPoint>);

struct TNPoint {
    float x;
    float y;
    ~TNPoint() {} // user-provided деструктор делает тип нетривиальным
                  // даже если деструктор ничего не делает
};

static_assert(!std::is_trivially_destructible_v<TNPoint>);

// Возврат через регистр
extern TPoint zero_point() {
    return {0,0};
} 

// Возврат через указатель
extern TNPoint zero_npoint() {
    return {0,0};
} 
```

```
zero_point():                        # @zero_point()
        xorps   xmm0, xmm0
        ret
zero_npoint():                       # @zero_npoint()
        mov     rax, rdi
        mov     qword ptr [rdi], 0
        ret
```

Особенно болезненно все становится с шаблонами, поскольку тривиальность шаблонной структуры может зависеть от тривиальности параметров.

В реализациях стандартной библиотеки C++03, например, шаблона `pair` можно [увидеть](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/stl_pair.h#L624) user-provided, ничего дополнительно не делающие, конструкторы копий. В C++11 и выше их уже [нет](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/stl_pair.h#L210). Так что это еще одна точка бинарной несовместимости библиотек на старом C++ с библиотеками нового C++. 

Проблемы со сломом ABI вокруг тривиальных типов могут застать вас врасплох не только в самом C++, но и в Rust, если вы попытаетесь общаться с плюсовыми библиотеками. Например, есть [баг](https://github.com/rust-lang/rust-bindgen/issues/778) с генерацией биндингов.

Будьте внимательны с тривиальными типами. И если вы намерены предоставлять стабильный ABI своей библиотеки, то выстраивайте его вокруг чистых сишных структур и функций, а не вокруг зоопарка из мира C++.

Также помните, что тривиальность
1. Легко ломается (достаточно [добавить](https://godbolt.org/z/b8T7T3Tbj) инициализатор, и тип уже не trivially_constructible)
2. Позволяет попадаться на [неинициализированные](https://godbolt.org/z/fW7sE9v37) переменные
3. Не всегда влияет на ABI (в основном влияют деструкторы и конструкторы копирования/перемещения)

## Полезные ссылки
1. https://en.cppreference.com/w/cpp/types/is_trivial
2. https://www.youtube.com/watch?v=ZxWjii99yao