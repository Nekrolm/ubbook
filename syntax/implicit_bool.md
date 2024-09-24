# Неявное приведение к bool

Вы пишете новую восхитительную библиотеку сериализации в JSON. Для этого у вас уже написано много своих версий функции `stringify` для поддерживаемых JSON типов. Их там немного...

И вот у вас есть

```C++
auto stringify(bool b) -> std::string_view {
    return b ? "true" : "false";
} 

auto stringify(std::string_view s) -> std::string_view {
    return s;
}
```

Выглядит хорошо и логично.

Вы тестируете эти функции

```C++
int main() {
    std::cout << stringify(true) << "\n";
    std::cout << stringify("string") << "\n";
}
```
И [получаете](https://gcc.godbolt.org/z/hjE1Kzvxf)
```
true
true
```

Удивлены? Но тут нет ничего удивительного! Просто строковый литерал, который имеет тип `const char[7]` неявно приводится к `const char*`, который неявно приводится к `bool`. А поскольку это все built-in преобразования они имеют приоритет перед user-defined преобразованием к `std::string_view` через его конструктор.


C неявным приведением указателей к `bool` есть еще известный дефект в инициализаторе через фигурные скобки

```C++
bool array[5] = {true, false, true, false, true};
std::vector<bool> vector {array, array + 5};
std::cout << vector.size() << "\n";
```
[Будет выведено](https://gcc.godbolt.org/z/jobeh6) 2, а не 5. Потому что указатели неявно приводятся к `bool`!
Дефект кое-как [исправили](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1957r2.html) в C++20. [Теперь Clang отказывается это компилировать, а GCC просто выдает предупреждение](https://gcc.godbolt.org/z/h3YzzEMz3).

```
<source>:9:32: error: type 'bool[5]' cannot be narrowed to 'bool' in initializer list [-Wc++11-narrowing]
    9 |     std::vector<bool> vector { array, array + 5};
      |                                ^~~~~
<source>:9:32: note: insert an explicit cast to silence this issue
    9 |     std::vector<bool> vector { array, array + 5};
      |                  
```

-------

А вы знаете как определить для вашего типа все возможные арифметические операторы и операторы сравнения разом? Нужно всего лишь определить неявный оператор приведения к `bool`, конечно же!

```C++
struct OptionalPositive {
    int x;

    operator bool() const {
        return x >= 0;
    }
};

int main() {
    std::cout << 5 + OptionalPositive { 5 };
    std::cout << (5 < OptionalPositive { 5 });
    std::cout << (5 == OptionalPositive { 5 });
    std::cout << (5 * OptionalPositive { 5 });
}
```
Это компилируется и выдает результат `6005`. Потому как выполняется user-defined неявное приведенине к `bool`, который далее неявно приводится к `int`. Все правильно.

Последние версии Clang хотя бы вывают частично [предупреждения](https://gcc.godbolt.org/z/fs98G3o3f)
```
<source>:13:21: warning: result of comparison of constant 5 with expression of type 'bool' is always false [-Wtautological-constant-out-of-range-compare]
   13 |     std::cout << (5 < OptionalPositive { 5 });
      |                   ~ ^ ~~~~~~~~~~~~~~~~~~~~~~
<source>:14:21: warning: result of comparison of constant 5 with expression of type 'bool' is always false [-Wtautological-constant-out-of-range-compare]
   14 |     std::cout << (5 == OptionalPositive { 5 });
      |      
```
Правда, если поменять тип константы слева на `double`, в Clang 19. предупреждение исчезнет. Но компилироваться оно [не перестанет](https://gcc.godbolt.org/z/MhafPcTvv).

Никогда, если только у вас не C++98, не определяйте неявный `operator bool`! Он всегда должен быть `explicit`. Если вы боитесь, что это заставит вас делать `static_cast<bool>` там, где этого не хочется делать, то не переживайте! 
С++ определяет несколько контекстов, в которых `explicit operator bool` все равно может быть вызван неявно: в условиях `if`, `for` и `while`, а также в логических операциях. Этого достаточно для большинства использнований `operator bool`.

Если у вас C++98... Я вам очень соболезную. Но и даже в вашем печальном случае есть решение. Чудовищно громоздкое, но решение — можете ознакомиться с устаревшей [Safe Bool Idiom](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Safe_bool) в свободное время в качестве домашнего задания. Если коротко, вместо `operator bool` предгалалось определить

```C++
// Указатель на метод в приватном классе!
typedef void (SomePrivateClass::*bool_type) () const; 
operator bool_type(); // неявное приведение к этому указателю
```
И тогда бы ваш объект в условных операциях неявно приводился бы к указателю, а указатель бы далее неявно приводился к `bool`.