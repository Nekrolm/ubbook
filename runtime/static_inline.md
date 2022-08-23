# static inline

C++ славен тем, что почти все его конструкции невероятно сильно зависят от контекста, и просто взглянув на случайный участок кода крайне сложно быть уверенным в понимании того, что же он делает. Перегруженные операторы, контекстно-зависимые значения ключевых слов, ADL, `auto`, `auto`, `auto`!

Одно из самых перегруженных значениями ключевых слов в C++ -- `static`.

- `static` это и модификатор видимости, влияющий на линковку,
- `static` это и storage-модификатор, влияющий на то, где и как долго переменная будет храниться,
- `static` это еще и модификатор, влияющий на то, как переменная или метод, ассоциированные с классом или структурой, будут взаимодействовать с объектами этих типов.

В C++23 [будут](https://wg21.link/P1169R4) еще и static перегрузки для `operator()`! Это будет что-то новое, восхитительное и прекрасное.

Главное -- не путать со static модификатором при перегрузке других операторов вне класса. Ведь это уже модификатор видимости!
И если написать в разных единицах трансляции что-нибудь вот такое
```C++
/// TU1.cpp
static Monoid operator + (Monoid a, Monoid b) {
    return {
        a.value + b.value
    };
}

Monoid sum(Monoid a, Monoid b) {
    return a + b;
}

/// TU2.cpp
static Monoid operator + (Monoid a, Monoid b) {
    return {
        a.value * b.value
    };
}

Monoid mult(Monoid a, Monoid b) {
    return a + b;
}

/// main.cpp
int main(int argc, char **argv) {
    auto v1 = sum({5}, {6}).value;
    auto v2 = mult({5}, {6}).value;
    std::cout << v1 << " " << v2 << "\n";
}
```
То оно даже будет [работать](https://godbolt.org/z/G5747YGK8) ожидаемым образом. Ведь никакой проблемы нет -- определения локальны в единицах трансляции.

В C++17 дополнительными значениями обросло еще и ключевое слово `inline`.

Когда-то оно было лишь подсказкой компилятору, что тело функции надо "встраивать" в место вызова -- то есть не делать относительно дорогой `call` с сохранением точки возврата, регистров, еще чего-то, прямо в место вызова вотнкуть инструкции... Посдказка эта, правда, не всегда работает. По разным причинам. Но в основном потому что программисты писали и пишут ее налево и направо даже туда, куда этого делать не стоит, чтобы не раздувать чрезмерно получаемый код. Но это не наша история. Наша история о другом.

В современном C++ `inline` используется чаще всего только для того чтоб поместить определение функции в заголовочный файл. 
В C это тоже работает, но [совсем не так](https://godbolt.org/z/PeaqGr7r3) -- вместо ошибки multiple definition, к которой приводит помещение не-inline функций в заголовочный файл, и которую мы хотели избежать, мы вовсе получили undefined reference. 

В C inline определения из заголовка нужно сопрячь модификатором `static`. И, возможно, получить code bloating, потому что получите копию функции в каждой единице трансляции и все они будут считаться разными, если линковщик окажется не достаточно умным.

Либо все-таки [предоставить](https://godbolt.org/z/YPvGv16fG) одно не-inline определение где-нибудь. Например, вот таким мерзким трюком
```C
// square.h
#ifdef DEFINE_STUB
#define INLINE 
#else 
#define INLINE inline
#endif

INLINE int square(int num) {
    return num * num;
}

// square.c
#define DEFINE_STUB 
#include "foo.h"

// main.c
#include "foo.h"

int main() {
    return square(5);
}
```

Или же [упомянуть](https://godbolt.org/z/n6q5jeqG4) где-нибудь объявление этой функции со спецификатором `extern` (или даже без него [может работать](https://godbolt.org/z/5sq7Y6rx5))
```C
// square.h
inline int square(int num) {
    return num * num;
}

// square.c
#include "foo.h"
extern int square(int num);

// main.c
#include "foo.h"

int main() {
    return square(5);
}
```

Либо, пользуясь GCC, никогда [не собирать](https://godbolt.org/z/EG7Kxbfov) сишный код без оптимизаций. Я таких разработчиков тоже видел. Но работает это решение [не всегда](https://godbolt.org/z/hc9MrbdY6)
```C
// square.h
inline int square(int num) {
    return num * num;
}

inline int cube(int num) {
    return num * num * num;
}

// main.c
#include "foo.h"
#include <stdlib.h>

typedef int (*fn) (int);

int main() {
    fn f;
    if (rand() % 2) {
        f = square;
    } else {
        f = cube;
    }
    // адреса inline-функции не известны -> undefined reference
    return f(5);
}
```

Но вернемся к C++. Помимо функций в заголовках иногда очень хочется определять еще и переменные. В приличных проектах, конечно, в основном константы. Но разработка сложна, туманна и полна ужасов. А также нестандартных креативных решений, которые пришлось принять здесь и сейчас. Поэтому встречаются не только константы.

К сожалению, в C++ до 17 стандарта просто так взять и поместить в заголовочный файл определение какой-то константы было не всегда возможно. А если и возможно, то с интересными [спецэффектами](https://godbolt.org/z/W8c1YbqdE).

```C++
// my_class.hpp
struct MyClass {
    static const int max_limit = 5000;
};

// main.cpp
#include "my_class.hpp"

#include <algorithm>

int main() {
    int limit = MyClass::max_limit; // OK
    return std::min(5, MyClass::max_limit); // Compilation error! std::min хочет принять ссылку, но линкер не знает адрес этой константы!
}
```

Можно написать 
```C++
// my_class.hpp
struct MyClass {
    static constexpr int max_limit = 5000;
};
```
И оно [заработает](https://godbolt.org/z/93cbWb7eY)

Но `constexpr` возможен не всегда и тогда все-таки придется взять и отнести определение в отдельную единицу транслсяции...

Пришел C++17 и нашим мучениям настал конец! Теперь можно написать `inline` у переменной и компилятор это съест, сгенерирует подобающую аннотацию для символа в объектном файле, чтоб линковщик более не кричал на multiple definition. Пусть берет любое, мы гарантируем что все определения одинаковые, а иначе undefined behavior.

```C++
// my_class.hpp
#include <unordered_map>
#include <string>

struct MyClass {
    static const inline
    std::unordered_map<std::string, int> supported_types_versions = {
        {"int", 5},
        {"string", 10}
    };
};

inline const std::unordered_map<std::string, int> another_useful_map = {
    {"int", 5},
    {"string", 6}
};

void test();

// my_class.cpp
#include "my_class.hpp"
#include <iostream>

void test() {
    std::cout << another_useful_map.size() << "\n";
}

// main.cpp
#include "my_class.hpp"
#include <algorithm>
#include <iostream>

int main() {
    std::cout << MyClass::supported_types_versions.size() << "\n";
    test();
}
```
Все прекрасно [работает](https://godbolt.org/z/W79dbxGz3) -- никаких multiple definitions и никаких undefined references! Невероятно похорошел C++ при 17-ом стандарте!


Внимательный читатель уже должен был почувствовать и даже заметить подвох.

Вот перед вами блок кода 

```C++
DEFINE_NAMESPACE(details)
{
   class Impl { ... };

   static int process(Impl);

   static inline const std::vector<std::string> type_list = { ... };
}; 
```
Может ли что-то пойти не так?

Конечно же может! Это же C++!

`DEFINE_NAMESPACE(name)` может быть определен как

```C++
#define DEFINE_NAMESPACE(name) namespace name
```

А может быть как
```C++
#define DEFINE_NAMESPACE(name) struct name
```
Что?! Да! Что если из благих побуждений, чтоб спрятать доступ к перегрузке функции `process` от вездесущего ADL, однажды сумрачному гению автора библиотеки пришло в голову именно такое решение, которое включается и выключается всего одним макросом!

В таких случаях вообще-то `type_list` это разные вещи. 

В случае `namespace` это `static inline` глобальная переменная. `inline` тут как бы бесполезен, потому что `static` и глобальной переменной модифицирует видимость (linkage). B в каждой единице трансляции, в которой окажется такой заголовок подключенным, будет своя копия переменной `type_list`.

В случае же `class` или `struct` этот `static inline` поле, ассоциированное с классом. И оно будет одно на всех.

Ну ладно, какая разница! Они же константы и объявлены одинаково! Никто ничего не заметит на практике... Разумеется.

А теперь мы вспоминаем, что иногда нам нужны не константы. Например, если мы опять таки делаем эту избитую систему с автоматической регистрацией плагинов при загрузке библиотек или иную систему авторегистрации типов.

Вот так [все работает](https://godbolt.org/z/s5aca7hce). Красиво и ожидаемо.
```C++
// plugin_storage.h
#include <vector>
#include <string>
using PluginName = std::string;
struct PluginStorage {
    static inline std::vector<PluginName> registered_plugins;
};

// plugin.cpp
#include "plugin_storage.h"

namespace {
struct Registrator {
    Registrator() {
        PluginStorage::registered_plugins.push_back("plugin");
    }
} static registrator_;
}

// main.cpp
#include "plugin_storage.h"
#include <iostream>
int main() {
    // печатает ровно один элемент
    for (auto&& p : PluginStorage::registered_plugins) {
        std::cout << p << "\n";
    }
}
```
Но меняем `struct PluginStorage` на `namespace PluginStorage` -- все компилируется, но [уже не работает](https://godbolt.org/z/7P7aEzxjT). Переменная `PluginStorage` своя в каждой единицу трансляции, поэтому в `main` мы видим пустой список.
Нужно удалить `static` перед `inline` и мы [получим](https://godbolt.org/z/hnn5YM1Wf) желаемое поведение снова.

### Итого

Изменяемые глобальные статические переменные это сложно везде. В Rust, например, обращение к ним обязательно требует `unsafe`. 
C++ ничего не требует. Вам нужно самим помнить о множественных синтаксических ритаулах, которые нужно произвести.
- Спрятать в функцию, чтоб избежать static initialization order fiasco
- Не написать лишних `static`
- Не запихнуть по неосторожности в заголовочный файл
- Максимально ограничить доступ

И еще не забыть про многопоточный доступ.

С++17 породил `static inline` переменные. Они удобные. Но только когда неизменяемые. Хотя и не безпроблемные. 
Средства просмотра изменений на ревью могут не показывать весь файл, а только лишь часть с добавлением. Если видите `static inline` не забудьте посмотреть, в каком он контексте. Если это проигнорировать в лучшем случае ваши исполняемые файлы будут тяжелыми. В худшем -- можно уйти во многие часы безнадежной отладки после какого-нибудь минималистичного изменения: кто-то объявление переменной с глобальным состоянием в заголовок вынес или наоборот внес, логически же ничего не поменялось...

Изменяемые статики -- страшное зло. С ними не только у рядовых разработчиков проблемы. 
На момент написания этой статьи в clang имеется [баг](https://github.com/llvm/llvm-project/issues/55804) с порядком инициализации статиков внутри одной единицы трансляции. Из-за неправильной сортировки `static` глобальных переменных и `static inline` полей классов.

### Полезные ссылки
1. https://en.cppreference.com/w/cpp/keyword/static
2. https://en.cppreference.com/w/cpp/language/inline

