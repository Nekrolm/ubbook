# Direct initialization синтаксис и ссылочные поля

Инициализация в C++ -- одна из самых сложных и запутанных тем среди, наверное, всех языков программирования.
Ей даже отдельная книжка на 400 страниц посвящена! - Bartlomiej Filipek, C++ Initialization Story: A Guide Through All Initialization Options and Related C++ Areas (C++ Stories). Так что я позволю себе не разбирать все тонкости инициализации, а затрону лишь ту прекрасную часть, которая непосредственно связана с нашими любимыми ошибками -- висячими ссылками.

С++11 принес в язык list (uniform) initialization syntax. C фигурными скобочками. Он считается нынче почти всегда предпочтительным. Но вот незадача, в некоторых краевых случаях его недостаточно.

Например
```C++
// У вас есть структура-аггрегат
struct Pair {
    int x, y;
};
// И вы хотите создать ее в куче с помощью std::make_shared/unique
auto p = std::make_unique<Pair>(1, 5);
```
В С++17 такой код не соберется. Компилятор выплюнет длинную ошибку [подстановки](https://godbolt.org/z/TzEGx66nn)
```
/opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/bits/unique_ptr.h: In instantiation of 'std::__detail::__unique_ptr_t<_Tp> std::make_unique(_Args&& ...) [with _Tp = Pair; _Args = {int, int}; __detail::__unique_ptr_t<_Tp> = __detail::__unique_ptr_t<Pair>]':
<source>:9:36:   required from here
    9 |     auto p = std::make_unique<Pair>(1, 5);
      |              ~~~~~~~~~~~~~~~~~~~~~~^~~~~~
/opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/bits/unique_ptr.h:1076:30: error: new initializer expression list treated as compound expression [-fpermissive]
 1076 |     { return unique_ptr<_Tp>(new _Tp(std::forward<_Args>(__args)...)); }
      |                              ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/bits/unique_ptr.h:1076:30: error: no matching function for call to 'Pair::Pair(int)'
<source>:4:8: note: candidate: 'Pair::Pair()'
```

В С++20 решили это недоразумение исправить, так что теперь у нас есть еще один uniform, то есть direct, initialization syntax. C круглыми скобочками.

```C++
// https://godbolt.org/z/fTM4n7nqz
auto p = std::make_unique<Pair>(1, 5); // теперь компилирутся в C++20
```

Ну добавили и добавили... Стало же удобнее? Можно теперь всегда круглые скобки использовать?...
Не спешите. С++ не был бы самим собой, если бы с новой фичей не поставлялся бы новый подвох.

Если в вашей структуре есть ссылочные `const&` или `&&` поля, то с помощью list initialization syntax вы можете спокойно инициализировать их временными значениями. Время жизни временных объектов будет продлено.
А вот в случае direct initialization syntax -- нет. Вы получите висячую ссылку. И никакой диагностики от компилятора стандарт не требует.

```C++
// https://gcc.godbolt.org/z/PW41fx1eE
struct S {
    const int& x;
    const std::string& s;
};

// list initialization 
int main() {
    S s { 1 + 1, 
         "hellooooooooooooooooooooooo0000000" };
    return s.s.length(); // все отлично. Возвращает 34
}

// direct initialization
int main() {
    S s ( 1 + 1, 
          "hellooooooooooooooooooooooo0000000" );
    return s.s.length(); // Бум! Undefined behaviour. 
    // Возвращает что угодно. GCC14 -std=c++23 -O3. Возвращает 98
}
```

Просто не пишите такой код, делов-то!

Но такой код легко может случайно родиться из шаблонов
```C++
template <class T1>
struct Wrapper {
    T1 first;
};

auto make(auto f) {
    using Result = Wrapper<decltype(f())>;
    return std::make_unique<Result>(f() + 10); // Ok, пока f не возвращает ссылки
}
```
Или при неосторожном рефакторинге с оптимизациями

```C++
struct Config { ... }; // Large object
struct Widget {
    // было
    // Config config;
    // Вы обнаружили что копируете один и тот же конфиг сотни раз
    // И решили расшарить его между виджетами по const ссылке
    const Config& config;
};

// Эта строчка после вашей оптимизации продолжает молча компилироваться
// но теперь влечет неопределенное поведенине
// пример: https://gcc.godbolt.org/z/q73erhYWs
auto parent_widget = std::make_unqiue<Widget>(read_config()); 
// И статические анализаторы пока молчат https://gcc.godbolt.org/z/aMsT3afxb
```



Нужно еще отметить, что продление жизни работает только при инициализации объектов аллоцированных на стеке. Если же вы создаете объект на куче/в собственном буфере с помощью `operator new`
```C++
// https://godbolt.org/z/7Y5brzGKv
struct S {
    const std::string& s;
};

int main() {
    auto bad = new S {  "hellooooooooooooooooooooooo0000000" };
    return  bad->s.length();
}
```
То успешно получите висячую ссылку. GCC 14 молчит. Clang 19 выдает предупреждение
```
<source>:10:25: warning: temporary bound to reference member of allocated object will be destroyed at the end of the full-expression [-Wdangling-field]
   10 |     auto bad = new S {  "hellooooooooooooooooooooooo0000000" };
```


## Значения по умолчанию для ссылочных полей

Закончить, пожалуй, нужно еще одним недоразуменинем с ссылочными полями. Им же можно задать значения по умолчанию...

```C++
struct Config {
    // Так можно и оно компилируется
    const std::string& s = "default value"; 
};
```

И теперь если мы создадим такой объект по умолчанию мы обнаружим, что

В случае list initialization:
```C++
Config c1 {}; // Ok c GCC 14. Почти ok с Clang 18
c1.s.length();
/*
 >:8:15: warning: lifetime extension of temporary created by aggregate initialization using a default member initializer is not yet supported; lifetime of temporary will end at the end of the full-expression [-Wdangling]
    8 |     Config c {}; 
*/
// Но при этом работает и санитайзеры не находят проблем
// https://gcc.godbolt.org/z/aWWdh3cs7
```


Но если же мы воспользуемся неявным вызовом конструктора по-умолчанию
```C++
// А вот так уже не все хорошо. 
Config c2;
c2.s.length();

// Clang 18: <source>:3:8: error: reference member 's' binds to a temporary object whose lifetime would be shorter than the lifetime of the constructed object
// GCC 14: компилируется. Санитайзеры обнаруживают use-after-free при обращении к полю s
// https://gcc.godbolt.org/z/vKT1PqvPz
/*
 =1==ERROR: AddressSanitizer: stack-use-after-scope on address 0x7448ea400088 at pc 0x000000401337 bp 0x7fff3bd6f410 sp 0x7fff3bd6f408
READ of size 8 at 0x7448ea400088 thread T0
    #0 0x401336 in std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >::length() const /opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/bits/basic_string.h:1084
*/ 
// GCC с флагом -Wextra выдает предупреждение
// <source>:3:8: warning: a temporary bound to 'Config::s' only persists until the constructor exits [-Wextra]
```

В общем, нельзя так просто взять и проинициализировать объект со ссылочными полями и ничего себе не отстрелить.


## Полезные ссылки
1. [Bartlomiej Filipek, C++ Initialization Story: A Guide Through All Initialization Options and Related C++ Areas (C++ Stories)](https://www.amazon.co.uk/Initialization-Story-Through-Options-Related-ebook/dp/B0BX1LG9LT)
