# Списки захвата лямбда-функций

C++11 подарил нам лямбда-функции и, вместе с ними, еще один способ неявного получения висячих ссылок.

Лямбда-функция, захватывающая что-либо по ссылке, безопасна до тех пор, пока она не возвращается куда-либо за пределы области, в которой ее создали. Как только мы куда-то возвращаем или сохраняем лямбду, начинается веселье:

```C++
auto make_add_n(int n) {
    return [&](int x) {
        return x + n; // n — станет висячей ссылкой!
    };
}

...
auto add5 = make_add_n(5);
std::cout << add5(5); // UB!
```

Ничего принципиально нового — тут все те же проблемы, что и с возвратом ссылки из функции.
clang иногда [способен выдать предупреждение](https://godbolt.org/z/rsq8hM).

Но стоит нам принять аргумент `make_add_n` по ссылке — и [никаких предупреждений не будет](https://godbolt.org/z/1K89z9).

Аналогично проблему [можно наиграть](https://godbolt.org/z/31KdTj) и для методов объектов:
```C++
struct Task {
    int id;

    std::function<void()> GetNotifier() {
        return [this]{
            // this — может стать висячей ссылкой!
            std::cout << "notify " << id << "\n";
        };
    }
};

int main() {
  auto notify = Task { 5 }.GetNotifier();
  notify(); // UB!
}
```

Но в этом примере можно заметить `this` в списке захвата и насторожиться. До C++20 же можно [отстрелить ногу](https://godbolt.org/z/WExKPo) чуть менее явно:
```C++
struct Task {
    int id;

    std::function<void()> GetNotifier() {
        return [=]{
            // this — может стать висячей ссылкой!
            std::cout << "notify " << id << "\n";
        };
    }
};
```
`=` предписывает захватывать все по значению, но захватывается не поле `id`, а сам указатель `this`.

---------

Если видите лямбду, в списке захвата которой есть `this`, `=` (до С++20) или `&`,
обязательно проверьте, как и где эта лямбда используется. Добавьте перегрузки проверки времени жизни захватываемых переменных.
```C++
struct Task {
    int id;

    std::function<void()> GetNotifier() && = delete;

    std::function<void()> GetNotifier() & {
        return [this]{
            // для this теперь намного сложнее стать висячей ссылкой
            std::cout << "notify " << id << "\n";
        };
    }
};
```

Если возможно, вместо захвата по ссылке, лучше использовать захват по значению или захват с инициализацией перемещением.

```C++
auto make_greeting(std::string msg) {
    return [message = std::move(msg)] (const std::string& name) {
        std::cout << message << name << "\n";
    };
}
...
auto greeting = make_greeting("hello, ");
greeting("world");
```

## Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/lambda
2. http://cppatomic.blogspot.com/2018/03/modern-effective-c-avoid-default.html