# Lifetime extension в тернарном операторе

У этой истории довольно нестандартное начало для этой книги. Заметить странности и неожиданные ловушки во встроенном операторе C++ внезапно поспособствовала новая возможность в языке Rust. Последние несколько лет я много работаю с кодовыми базами на C++ и Rust, и на стыке между ними. А поскольку вполне естественно переносить ожидания из одного языка на другой, тем более когда они похожи, я решил проверить, а как же работает похожий код на С++.

Rust в версии 1.78 неожиданно расширил возможности по автоматическому продлении жизни временных объектов. Теперь в нем можно, например, написать так

```Rust
let uri: &str = ...;
...
let updated_uri: &str = if !query.is_empty() {
    // Можно вернуть ссылку на временную строку!
    // ее время жизни будет автоматически продлено
    // Ранее этот код не проходил проверку заимствований\
    // и не компилировался
    &format!("{uri}?{query}")
} else {
    uri
}
```
Раньше было сложнее совместить краткость и отсутствие лишней аллокации.
Нужно либо писать довольно уродливо явно
```Rust
let uri: &str = ...;
let updated_uri_tmp: String;
let updated_uri: &str = if !query.is_empty() {
    updated_uri_tmp = format!("{uri}?{query}");
    &updated_uri_tmp
} else {
    uri
};
```
либо прибегать к специальным типам
```Rust
use std::borrow::Cow; // Clone-on-Write smart pointer
let updated_uri: Cow<str> = if !query.is_empty() {
    format!("{uri}?{query}").into() // wrap into Cow::Owned
} else {
    uri.into() // wrap into Cow::Borrowed
};
```

Особенно удобным становилось испольщование динамически полиморфных интерфейсов

```Rust
let output: &mut dyn std::io::Write = match confg {
    StdOut => &mut std::io::stdout(),
    File { path } => &mut std::fs::File::create(path)?,
}
```

Вернемся теперь к C++. В нем, конечно, `switсh` не такой удобный и `if - else` не является выражением — не может возвращать значения. Но в C++ есть тернарный оператор, а вот он уже что-то возвращает.

Посмотрим на довольно распространенный сценарий: у нас есть некоторый key-value контейнер с неисзменяемой конфигурацией. Мы обращаемся к какому-то ключу, если он там есть -- отлично. Иначе -- используем значение по умолчанию. 

```C++
using Map = std::map<int, std::string>;

// Мы сразу рассмотрим наиболее общий случай 
// и наиболее оптимальный случай:
// значение по умолчанию предоставляется функцией. 
// Так мы можем избежать его вычисления, если ключ есть в таблице
void test_default_getter(const Map& m, int key, auto default_getter) {
    std::cout << "try ternary ? const string& : function()\n";
    auto iter = m.find(key);
    
    // используем auto и universal reference, поскольку 
    // 1. Мы не знаем тип default_getter()
    // 2. Это работает
    // 3. Так рекомендуют гайдлайны -- можем еще const добавить
    auto&& value = iter != m.end() ? iter->second : default_getter();
    std::cout << "value=" << value << "\n";
    std::cout << "address=" << uintptr_t(value.data()) << "\n";
}
```
Для отладки, функция выводит адрес начала содержимого строки-значения. Так мы сможем сказать, произошло ли копирование значения из хранилища или же мы успешно взяли ссылку на него — чего бы вполне хотелось бы во всех случаях, когда ключ в таблице присутствиует.

А теперь я предлагаю вам посмотреть на следующие 7 случаев и ответить, что именно произойдет

```C++
int main() {
    const Map m {
        {42, "Value in Table"}
    };
    std::cout << "data address in map: " << uintptr_t(m.at(42).data()) << "\n";
    using namespace std::literals;
    test_default_getter(m, 42, []{  return "default"sv; });
    test_default_getter(m, 42, [s = "default"s]() -> const std::string& {  return s; });
    test_default_getter(m, 42, [s = "default"s]() mutable -> std::string& {  return s; });
    test_default_getter(m, 42, [s = "default"s]() mutable -> std::string&& {  return std::move(s); });
    test_default_getter(m, 42, [s = "default"s]() -> const std::string&& {  return std::move(s); });
    test_default_getter(m, 42, []{  return "default"s; });
    test_default_getter(m, 42, []{ return "default"; });
}
```
Несмотря на все разумные ожидания, только в первых трех случаях мы действительно получим ссылку на строку в таблице. В оставшихся четырех мы неявно получим копию!

```
https://godbolt.org/z/dha9b8fY6

data address in map: 7201512
try ternary ? const string& : function()
value=Value in Table
address=7201512
try ternary ? const string& : function()
value=Value in Table
address=7201512
try ternary ? const string& : function()
value=Value in Table
address=7201512
try ternary ? const string& : function()
value=Value in Table
address=140737440180832
try ternary ? const string& : function()
value=Value in Table
address=140737440180832
try ternary ? const string& : function()
value=Value in Table
address=140737440180832
try ternary ? const string& : function()
value=Value in Table
address=140737440180832
```

У тернарного оператора в C++ совершенно удивительные правила вывода типа возвращаемого значения:

```
bool ? T&  : T&  -> T&
bool ? T&& : T&& -> T&&
bool ? T&  : T&& -> T
bool ? T&  : T   -> T
bool ? U   : T   -> U или T, что к чему приведется
```


Посмотрим на наши примеры:

```C++
test_default_getter(m, 42, []{  return "default"sv; });
```
`const string& : string_view` — первый неявно приводим ко второму. Взятие string_view от string происходит без копирования. Все отлично... И вроде безопасно.

А что если наша таблица с конфигурацией отпимизирована хранить string_view на части одной большой json-конфигурационной строки и ключа в ней нет?

```C++
using RefMap = std::map<int, std::string_view>;

void test_refmap_string(const RefMap& m, int key) {
    auto iter = m.find(key);
    auto&& value = iter != m.end() ? iter->second : std::format("value_for_key_{}_generated", key);
    std::cout << value << "\n";
}

int main() {
    const RefMap m1 {
        {42, "Value in Table"}
    };
    test_refmap_string(m1, 43);
}
```

Получаем классический use-after-free cо string-view
```
https://godbolt.org/z/xPhz7rTMb
Dq�kpKi%_generated
```

Неявные приведения типов всегда очень удобны для написания некорректных программ.

Продолжаем дальше с примерами. Второй и третий:

```C++
    test_default_getter(m, 42, [s = "default"s]() -> const std::string& {  return s; });
    test_default_getter(m, 42, [s = "default"s]() mutable -> std::string& {  return s; });
```
Вполне естественно работают как ожидалось -- без копирования.
В обеих ветвях тернарного оператора окажутся lvalue ссылки, const не важен. Результатом будет lvalue ссылка.


Четвертый и пятый

```C++
test_default_getter(m, 42, [s = "default"s]() mutable -> std::string&& {  return std::move(s); });
    test_default_getter(m, 42, [s = "default"s]() -> const std::string&& {  return std::move(s); });
```

Дадут `const std::string& : [const] std::string&&` в тернарном операторе и, согласно его правилам вывода, должны вернуть `std::string`. То есть скопировать из левого или переместить из правого. По-другому быть не может. 

Условно продлять время жизни и условно вызывать деструкторы C++, в отличие от Rust, не умеет. Если мы посмотрим снова на Rust-пример

```Rust
let updated_uri: &str = if !query.is_empty() {
    // чтобы это работало в Rust, на стеке 
    // уже должно быть неявно зарезервировано место под объект
    // а также должен быть runtime-проверяемый drop-флаг
    // означающий что объект был инициализирован во время выполнения этой ветки!
    // подробнее смотрите https://doc.rust-lang.org/nomicon/drop-flags.html
    &format!("{uri}?{query}")
} else {
    uri
}
```
С++ не позволяет себе такой неявности и сопряженных с ней некладных расходов.


Также копированине в тернарном операторе — это в некотором роде безопасное поведение по умолчанию: если результатом станет копия, то это точно не висячая ссылка!

Последние примеры. Шестой.

```C++
  test_default_getter(m, 42, []{  return "default"s; });
```
То же самое что и с четвертым и пятым, только возвращается чистое временное значенине, а не ссылка

`const std::string& : std::string` даст в результате `std::string`. И левый аргумент всегда будет скопирован.

Последний, седьмой пример

```C++
    test_default_getter(m, 42, []{ return "default"; });
```

Здесь же мы получаем
`const std::string& : const char*`. Указатель неявно приводится к `std::string`. Значит, для правого нужно будет создавать временный объект. Условного создания временных объектов в C++ нет — копируй левый аргумент!

-----
Ознакомившись с этим поведением я вспомнил все те десятки и сотни раз, когда я видел или сам писал

```C++
const auto& value = config.hasValue(key) ? config.GetValue(key) : "default";
```
не подозревая что я всегда делаю копию... Зато работало!

-----
Ну хорошо, со ссылками и временными значениями понятно. Выбран некоторый условно безопасный вариант и мы должны быть за это благодарны.

Посмотрим, как еще мы можем себе что-нибудь отстрелить. Я упоминал, что автоматическое продление времени жизни в Rust еще облегчает работу с полиморфными объектами.

```C++
class Base {
public:
    virtual void foo() const {
        std::cout << "base\n";
    };
};

class Derived: public Base {
public:
    void foo() const override {
        std::cout << "derived\n";
    };
};

void test_ternary_inheritance(bool cond, const Base& a) {
    const auto& x = cond ? Derived() : a;
    x.foo(); 
}

int main() {
    const Derived d;

    test_ternary_inheritance(true,  d);
    test_ternary_inheritance(false, d);
}
```

Поскольку мы теперь знаем, как работает тернарный оператор и что в райнтайме он по условию время жизни не продляет, можно относительно легко понять, что в обоих случаях произойдет копирование `a`. А вместе с ним и слайсинг — только подобъект базового класса будет скопирован. И дважды будет [выведено](https://godbolt.org/z/5j5hx4PWf) `base`.

А что если попробовать наоборот?

```C++
void test_ternary_inheritance_derived(bool cond, const Derived& a) {
    const auto& x = cond ? Base() : a;
    const auto& y = cond ? a : Base();
    x.foo(); 
    y.foo();
}
```

В свете всего того что мы уже увидели, результат окажется весьма неожиданным...
Ошибка компиляции! 

```C++
https://godbolt.org/z/dYEbG8fn3

<source>: In function 'void test_ternary_inheritance_derived(bool, const Derived&)':
<source>:28:26: error: operands to '?:' have different types 'Base' and 'const Derived'
   28 |     const auto& x = cond ? Base() : a;
      |                     ~~~~~^~~~~~~~~~~~
<source>:29:26: error: operands to '?:' have different types 'const Derived' and 'Base'
   29 |     const auto& y = cond ? a : Base();
      |   
```

И это же замечательно. Если код ошибочный, а слайсинг это чаще всего ошибка, то он не должен компилироваться. По крайней мере так происходит в GCC 14.2 и Clang 18.1. C MSVC 19 же все молча компилируется.

Если же мы просто уберем `const` из параметра

```C++
void test_ternary_inheritance_derived(bool cond, Derived& a) {
    const auto& x = cond ? Base() : a;
    const auto& y = cond ? a : Base();
    x.foo(); 
    y.foo();
}
```

И вот оно снова и под Clang и под GCC [компилируется](https://godbolt.org/z/1e9ejK4PK), как ожидается, со слайсингом.


