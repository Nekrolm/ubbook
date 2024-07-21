# Пользовательские операторы сравнения в C++20

С++, конечно, развивается медленным циклом в три года. Чтобы ничего не сломать. Большой успех -- большая ответственность...
Но несмотря на всю медлительность и осторожность, новые стандарты C++ все равно умудряются подложить мину там, где никто не ожидает.

С++20 добавил долгожданный "оператором НЛО" `<=>`, *three-way-comapison*, позволяющим существенно сократить однообразный код для определения операций сравнения над пользовательскими типами.

```C++
struct Pair {
    int x;
    int y;

    auto operator<=>(const Pair&) const = default; 
    // все операции <, >, ==, !=, <=, >= -- выведены автоматически в порядке объявления полей!
};

// И вот мы уже можем сравнивать точки!
Pair {1, 2} < Pair { 2, 3 };
```

Ну [почти](https://godbolt.org/z/1n4Mx6j3e).
```
<source>:8:10: error: 'strong_ordering' is not a member of 'std'
    8 |     auto operator <=> (const Pair&) const = default;
      |          ^~~~~~~~
<source>:1:1: note: 'std::strong_ordering' is defined in header '<compare>'; this is probably fixable by adding '#include <compare>'
```

Внезапно, вы обязаны подключить заголовок, если хотите использовать новую синтаксическую конструкцию 
c автоматической реализацией (через `=default`). Причем узнаете вы об этом только, когда попробуете использовать сравнение.

Добавим заголовок и все будет [работать](https://godbolt.org/z/5E4csxK6b). Здорово? Конечно же!
Но есть кое-что еще.

Не все типы можно или осмысленно упорядочивать. Иногда достаточно только равенства.
В C++20 можно определить `operator ==` и `operator !=` будет выведен автоматически.
Более того, реализация через `= defalt` также [работает](https://godbolt.org/z/Mc9YEsGWK).

```C++
struct Pair {
    int x;
    int y;
    
    bool operator==(const Pair&) const = default; 
    
};

// operator != выведен автоматически
Pair {1, 2} != Pair{3, 4};
```

Другие операторы сравнения тоже можно автоматически определять по-умолчанию.

Прекрасно. Но сравнениями для одного и того же типа все не заканчивается. Иногда нам нужно уметь сравнивать
объекты разных типов. Например `std::string` и `std::string_view`. 

Как разработчики справлялись с этой задачей до C++20?

Ну, например, эксплуатируя неявное приведение типов, когда это уместно.
```C++
struct String;

struct StringView {
     // Разрешаем неявное приведение, ведь это удобно
     StringView(const String&) {}
     bool operator==(const StringView &) const { return true; }
};
   
struct String {
    bool operator==(const StringView &sv) const { 
        // а тут меняем порядок местами, ведь у StringView уже есть operator ==
        // overload resolution наедет его по первому аргументу
        // а ко второму (*this: String) можно применить неявное приведенине типа. Все отлично!
        return sv == *this; 
    }
};

String{} == String{};
```
[Работает, компилируется c С++17, все отлично!](https://godbolt.org/z/Ynzo54sYe)

А без трюков 4 перегрузки нужно...

Ну хорошо. [Переходим](https://godbolt.org/z/Yn8M34d7o) в C++20.

```
Program returned: 139
Program terminated with signal: SIGSEGV
```

Шикарно! Надеюсь, если вы компилировали без `-Werror`, у вас были хотя бы тесты.

```
<source>:14:19: warning: in C++20 this comparison calls the current function recursively with reversed arguments
   14 |         return sv == *this;
      |                ~~~^~~~~~~~
<source>: In function 'int main()':
<source>:19:31: warning: C++20 says that these are ambiguous, even though the second is reversed:
   19 |     return String{} == String{};
```

С++20 позаботился о разработчиках. И теперь им не нужно выдумывать странные перестановки аргументов местами чтоб писать поменьше перегрузок операторов сравнения. 
C++20 ввел правила переписывания [всех операторов сравнения](https://en.cppreference.com/w/cpp/language/overload_resolution#Call_to_an_overloaded_operator), так что компиляторы выполнят перестановку за вас. Даже если в ней нет надобности. И разумеется веселые и находчивые разработчики старых кодовых баз получат бесконечную рекурсию. А c ней и неопределенное поведение. И SIGSEGV от переполнения стека, если повезет.

Эти изменения в правила поиска перегрузок для операций сравнения имели и другие побочные эффекты. Их постарались исправить в предложении [P2468R2 The Equality Operator You Are Looking For](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2468r2.html#code-patterns-which-fail-at-runtime). Он принято и реализовано в C++23.

Но внезапную рекурсию все равно не убрали! Полагайтесь на предупреждения компилятора.

В С++20/23, если у вас есть неявное приведение между типами, не нужно больше ничего выдумывать -- определяйте операции для одного и того же типа, а комбинации будут получены автоматическими перестановками.

```C++
struct String {
    bool operator==(const String&) const = default; 
};
struct StringView {
     StringView(const String&) {}
     bool operator==(const StringView &) const = default;
};
   
String{} == StringView{String{}};
StringView{String{}} == String{};
```

Если неявного приведения нет, то достаточно определить только одну дополнительную перегрузку

```C++
struct String {
    bool operator==(const String&) const = default; 
};
struct StringView {
     explicit StringView(const String&) {}
     bool operator==(const StringView &) const = default;
};
   
bool operator == (const String& s, const StringView& sv) {
    return sv == StringView{s};
}

// Обе перестановки работают
String{} == StringView{String{}};
StringView{String{}} == String{};
```
