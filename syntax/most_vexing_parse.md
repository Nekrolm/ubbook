# Most Vexing Parse

Помимо неопределенного поведения, в C++ есть неожиданное поведение,
произрастающее из следующих фантастических возможностей языка.

Пользовательские типы и функции можно *объявлять* [где попало и как попало](https://godbolt.org/z/MWszrj).

```C++
template <class T>
struct STagged {};


using S1 = STagged<struct Tag1>; // преобъявление струкруты Tag1
using S2 = STagged<struct Tag2*>; // преобъявление струкруты Tag2

void fun(struct Tag3*); // предобъявление структуры Tag3

void external_fun() {
    int internal_fun(); // предобъявление функции!
    internal_fun();
}

int internal_fun() { // определение предобъявленой функции
    std::cout << "hello internal\n";
    return 0;
}

int main() {
    external_fun();
}
```

При этом *определять* сущности можно не везде.
Типы можно определять локально — внутри функции. А функции определять нельзя.

```C++
void fun() {
    struct LocalS {
        int x, y;
    }; // OK

    void local_f() {
        std::cout << "local_f";
    } // Compilation Error
}
```

И все могло бы быть хорошо, если бы не одно: в C++ есть конструкторы, вызов которых [похож](https://godbolt.org/z/h6zTor) на объявление функции

```C++
struct Timer {
    int val;
    explicit Timer(int v = 0) : val(v) {}
};

struct Worker {
    int time_to_work;

    explicit Worker(Timer t) : time_to_work(t.val) {}

    friend std::ostream& operator << (std::ostream& os, const Worker& w) {
        return os << "Time to work=" << w.time_to_work;
    }
};

int main() {
    // ЭТО НЕ ВЫЗОВ КОНСТРУКТОРА!
    Worker w(Timer()); // предобъявление функции, которая возврщает Worker и принимает функцию, возвращающую Timer и не принимающую ничего!

    std::cout << w; // имя функции неявно преобразуется к указателю, который неявно преобразуется к bool
    // будет выведено 1 (true)
}
```

Подобная ошибка может быть труднообнаружима, если случайно предобъявленная функция используется в контексте приведения к `bool` или если объект, который хотели сконструировать, сам является вызываемым (у него перегружен `operator()`).

Может показаться, что виноват именно конструктор по умолчанию класса `Timer`. На самом деле, виноват C++.
В нем можно объявлять функции вот так:

```C++
void fun(int (val)); // скобки вокруг имени параметра допустимы!
```

И потому получать [более отвратительный](https://godbolt.org/z/dhz6nK) и труднопонимаемый вариант ошибки:

```C++
int main() {
    const int time_to_work = 10;
    Worker w(Timer(time_to_work)); // предобъявление функции, которая возвращает Worker
    // и принимает параметр типа Timer. time_to_work — имя этого параметра!

    std::cout << w;
}
```

Clang способен предепреждать о подобном.

С++11 и новее предлагают *universal initialization* (через `{}`), которая не совсем *universal* и имеет свои проблемы.
C++20 предлагает еще одну *uneversal* инициализацию, но уже снова через `()`...

Избежать проблемы можно, используя *Almost Always Auto* подход с инициализацией вида
`auto w = Worker(Timer())`. Круглые или фигурные скобки здесь — не так важно (хотя, на самом деле, важно, но в другой ситуации).

Возможно, когда-нибудь объявление функций в старом сишном стиле запретят в пользу
*trailing return type* (`auto fun(args) -> ret`). И вляпаться в рассмотренную прелесть станет значительно сложнее (но все равно [можно](https://www.youtube.com/watch?v=tsG95Y-C14k)!).

1. https://www.fluentcpp.com/2018/01/30/most-vexing-parse/
2. http://cginternals.github.io/guidelines/articles/almost-always-auto/
