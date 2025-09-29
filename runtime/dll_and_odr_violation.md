# ODR-violation и разделяемые библиотеки

[Ранее](odr_violation.md) я рассматривал ODR-violation в общих чертах и предупреждал о том, что может произойти, если случайно выбрать не то имя переменной, структуры или функции в C++. В этой же части я бы хотел продемонстрировать более изящный пример, не требующий приложения никаких усилий по написанию кривого кода. Достаточно просто иметь кривой код в ваших third-party зависимостях.

Недавно я имел дело со странным баг-репортом: 

Во внутреннем репозитории с пакетами обновилcя пакет с библиотекой [gtest](https://github.com/google/googletest) -- известная уважаемая библиотека для написания самых разных тестов на C++. И в результате обновления некоторые тесты в конечных приложениях стали внезапно падать.

Падать они стали по-разному. У одних стали валиться проверяющие ассерты. У других же все работало, проверки проходили, но [ctest](https://cmake.org/cmake/help/latest/manual/ctest.1.html) рапортовал что тестирующий процесс вышел с ненулевым кодом возврата.

Если с первыми происходило что-то совершенно невразумительное, то со вторыми можно было работать.

Запускаем тест вручную: 5/5 passed. Segmentation Fault. О!

Запустив тест под отладчиком и выведя бэктрейс, я получил нечто следующего вида

```
Program received signal SIGSEGV, Segmentation fault.
0x00007ffff7e123fe in __GI___libc_free (mem=0x55555556a) at ./malloc/malloc.c:3368
3368    ./malloc/malloc.c: No such file or directory.
(gdb) bt
#0  0x00007ffff7e123fe in __GI___libc_free (mem=0x55555556a) at ./malloc/malloc.c:3368
#1  0x00007ffff7fb75ea in std::vector<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >::~vector() () from ./libgtest.so
#2  0x00007ffff7db2a56 in __cxa_finalize (d=0x7ffff7fb5090) at ./stdlib/cxa_finalize.c:83
#3  0x00007ffff7fb2367 in __do_global_dtors_aux () from ./libgmock.so
```

`__do_global_dtors_aux () from ./libgmock.so` что-то страшное и одновременно прекрасное произошло, осталось лишь понять что именно.

Я прошелся по тестам: они были сгруппированы по отдельным исходникам и каждый из них собирался в свой исполняемый файл. Все тесты собирались одними и теми же параметрами компиляции: конфигурация задавалась в cmake тупо как 

```CMake
file(GLOB files "*_test.cpp")
foreach(file ${files})
  add_test(...., ${file})
endforeach()
```

Я открыл исходники падающего и не падающего теста: падающий тест использовал gMock. А не падающий не использовал. Но при этом оба исполняемых файла были слинкованы с библиотекой `libgmock.so`.

Посмотрим еще раз на фрагмент бэктрейса

```
std::allocator<char> > > >::~vector() () from ./libgtest.so
#2  0x00007ffff7db2a56 in __cxa_finalize (d=0x7ffff7fb5090) at ./stdlib/cxa_finalize.c:83
#3  0x00007ffff7fb2367 in __do_global_dtors_aux () from ./libgmock.so
```

Финализация глобальных объектов в libgmock как-то связана с деструктором глобальной переменной в libgtest.

Я открыл список изменений, что же там такое обновилось во внутреннем пакете с GTest...

```
Commit: ...
Produce shared libraries along with statics
```
И ровно две строчки, добавляющие в его CMakeLists.txt еще и динамические версии библиотек libgmock и libgtest.

Интересно. Я вышел в интернет с этим вопросом и обнаружил что-то очень [похожее](https://github.com/google/googletest/issues/930). Первым делом, глянув на дату issue, а также сверив даты коммитов во внутренней версии, я был глубоко разочаровам темпами обновления зависимостей (ведь на дворе был конец 2023 года, а issue датируется 2016). Но это уже совсем другая проблема C++...

Что же произошло на самом деле?

Фреймворк GoogleTest содержал две библиотеки
- gtest -- core библиотека со всеми причиндалами для автоматической регистрации тестов и легкого их запуска через макрос `RUN_ALL_TESTS`
- gmock -- библиотека специально для mock тестирования 

gmock линкуется с gtest.
Конечный пользовательский исполняемый файл с тестами линкуется с обеими библиотеками.

Все нормально. Ничего криминального в этом нет.

Для поддержки всей красоты автоматической регистрации тестов и фикстур (вам как пользователю не нужно никуда складывать тестовые функции, вы их просто объявляете с помощью макросов) gtest очень активно полагается на глобальные переменные.

Проблемным объектом, чей деструктор приводил к падению, оказался как видно из бэктрейса вектор строк в libgtest.so
В исходниках gtest я обнаружил, что этот вектор -- глобальная переменная куда `InitGoogleTest()` складывает распознанные аргументы командной строки. Просто глобальная перемменная, объявленная в компилируемом файле. Она не была в заголовочном файле. Всё вроде бы должно было быть хорошо... За одним исключением: она не была помечена `static` и не была обернута в анонимный namespace.

И что? Ведь все же работало?
Да, работало. Хитрость в том, как собирается библиотека gmock. Воспроизведем все пошагово.

Заведем свой gtest дома

```C++
// gtest.h
#pragma once
void initGoogleTest(int argc, char* argv[]);
void runTests();

// gtest.cpp
#include "gtest.h"

#include <vector>
#include <string>
#include <iostream>

// глобальная переменная, не static, как было в gtest
std::vector<std::string> g_args;

void runTests() {
    // просто для демонстрации
    std::cout << "run gtest\n";
    for (const auto& arg : g_args) {
        std::cout << arg << " ";
    }
    std::cout << "\n";
}

void initGoogleTest(int argc, char* argv[]) {
    for (int i = 0; i < argc; ++i) {
        g_args.push_back(argv[i]);
    }
}
```

Соберем его в статическую библиотеку. Ведь именно статические библиотеки были изначально в пакете.

```sh
g++ -std=c++17 -fPIC -O2 -c gtest.cpp
ar rcs libgtest.a gtest.o
```

Добавим свой gmock

```C++
// gmock.h
#pragma once
void runMocks();

// gmock.cpp
#include "gmock.h" 
#include "gtest.h" // gmock линкуется с gtest!

#include <iostream>

void runMocks() {
    // для демонстрации
    std::cout << "run Mocks:\n";
    runTests();
}
```

Соберем его также в статическую библиотеку

```sh
g++ -std=c++17 -fPIC -O2 -c gmock.cpp
ar rcs libgmock.a gmock.o gtest.o  # https://github.com/google/googletest/blob/d41f53ae7816863cc52bf3f357dff25597c58864/googlemock/make/Makefile#L89C1-L90C24
```

И начнем пользоваться
```C++
// main.cpp
#include "gtest.h"
#include "gmock.h"

int main(int argc, char* argv[]) {
    initGoogleTest(argc, argv);
    runMocks();
    runTests();
    return 0;
}
```

```sh
g++ -std=c++17 -O2 -o main main.cpp -L . -lgtest -lgmock
./main 1 2 3 4
run Mocks:
run gtest
./main 1 2 3 4 
run gtest
./main 1 2 3 4 
```
Должно быть очевидно, что прилинковав обе библиотеки gtest и gmock, мы уже как бы получили ODR-violation: gmock содержит в себе gtest, а значит у нас две копии глобальной переменной. Но всё работает: линкер выкинул одну из копий.

А теперь **добавим** динамические версии

```sh
g++ -shared -fPIC -o libgtest.so gtest.o
g++ -shared -fPIC -o libgmock.so gtest.o gmock.o
```
И пересоберем клиентский код

```
g++ -std=c++17 -O2 -o main main.cpp -L . -lgtest -lgmock
LD_LIBRARY_PATH=. ./main 
run Mocks:
run gtest
./main 
run gtest
./main 
Segmentation fault (core dumped)
```

Ура! Падает!
Обе библиотеки имеют свою собственную версию глобальной переменной с одним и тем же неявно экспортируемым именем. Использоваться опять-таки будет только **одна**.
После загрузки библиотеки, после конструирования глобальной переменной, стандарт C++ требует зарегистрировать (например через `__cxa_atexit`) функцию для вызова деструктора. У нас две библиотеки -- значит две функции будут вызваны. На одном и том же объекте. Double free. Конструктор, кстати, также вызывается дважды по одному и тому же адресу:

```C++
struct GArgs : std::vector<std::string> {
    GArgs() {
        std::cout << "Construct it: " << uintptr_t(this) << "\n";
    }
};

GArgs g_args;
```

```sh
LD_LIBRARY_PATH=.  ./main 1 2 3
Construct it: 140368928546992
Construct it: 140368928546992
run Mocks:
run gtest
./main 1 2 3 
run gtest
./main 1 2 3 
Segmentation fault (core dumped)
```

И это прекрасно. Хорошо. Теперь проблема ясна. Тесты, которые падали на ассертах, также страдали, но от других глобальных переменных. 
Остался последний вопрос: я упомянул, что все тесты собирались одинаково, но какие-то не падали -- те, что не использовали gmock. Но все равно с ним линковались.

Закомментируем использование нашего gmock
```C++
#include "gtest.h"
#include "gmock.h"

int main(int argc, char* argv[]) {
    initGoogleTest(argc, argv);
    // runMocks();
    runTests();
    return 0;
}
```
```
g++ -std=c++17 -O2 -o main main.cpp -L . -lgtest -lgmock
LD_LIBRARY_PATH=.  ./main 1 2 3
Construct it: 140699707998384
run gtest
./main 1 2 3 
```

О как! Конструктор вызвался только раз. Значит и деструктор будет вызван только раз. Все отлично. Но ведь мы же линковали... Современные линковщики достаточно умны чтоб не тащить то что не используется (что кстати иногда является проблемой, если у конструкторов в библиотеке есть побочные эффекты).

Если мы заставим gcc прилинковать gmock насильно
```sh
g++ -std=c++17 -O2 -o main main.cpp -Wl,--no-as-needed -L . -lgtest -lgmock
LD_LIBRARY_PATH=.  ./main 1 2 3
Construct it: 139687276064944
Construct it: 139687276064944
run gtest
./main 1 2 3 
Segmentation fault (core dumped)
```
Все будет, как и должно, сломано.

-----

Подобный паттерн по созданию проблем оказывается невероятно распространенным! И не только в C++

`LibA` статически влинковывается в `LibB` и обе `LibA` и `LibB` влинковываются в `BinC`. Самый частый кандидат на такую `LibA` -- библиотеки менеджмента памяти.

Например, при сборке динамических библиотек в Rust и подключении их в другие Rust проекты практически всегда люди натыкаются на эту [проблему](https://users.rust-lang.org/t/rust-dynamic-library-dynamically-called-in-rust-randomly-crashed-when-statically-linked-to-std/11074): `std` статически влинковывается и в библиотеку и в исполняемый файл. 

Я также обнаружил подобные проблемы в [AWS SDK](https://clickhouse.com/codebrowser/ClickHouse/contrib/aws/aws-cpp-sdk-core/include/aws/core/utils/memory/AWSMemory.h.html#253):
Глобальный UniquePtr также двумя путями попадает в конечное приложение, потому в его деструктор воткнули зануление, чтоб не вызывать `delete` дважды.


