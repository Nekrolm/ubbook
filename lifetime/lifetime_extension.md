# Lifetime extension

Продление времени жизни временных объектов тема широкая. И в этой серии заметок она встречалась не раз. Ведь работает эта особенность довольно в ограниченном числе случаев и чаще всего можно получить висячую ссылку. Однако в этой заметке я хочу остановиться на менее очевидном случае, с не совсем ожидаемыми последствиями.

В C++ при **первом** присваивании временного объекта `const lvalue` или `rvalue` ссылке, время жизни этого объекта расширяется до времени жизни ссылки.

```C++
std::string get_string();
void run(const std::string&);

int main() {
    const std::string& s1 = get_string(); 
    run(s1); // ok, ссылка валидна
    std::string&& s2 = get_string();
    run(s2); // ok, ccылка валидна
    // но
    std::string&& s3 = std::move(get_string()); // ссылка уже не валидна!
    // первое присваивание -- ссылка в аргументе std::move, 
    // ее время жизни ограничено телом move
    // аналогично для любой другой функции, принимающей и возвращающей ссылку
    // (std::move тут взят только для примера)
}
```

Чуть менее очевидная особенность: не только лишь ссылка на сам временный объект дает такой эффект, но и [на любой его подобъект!](https://godbolt.org/z/ob15ffvPP)


```C++
#include <iostream>
#include <string>
#include <vector>

struct User {
    std::string name;
    std::vector<int> tokens;
};

User get_user() {
    return {
        "Dmitry",
        {1,2,3,4,5}
    };
}

int main() {
    std::string&& name = get_user().name;
    // some hacky address arithmetics: User is alive, we can access data in it!
    // build with -fsanitize=address to ensure!
    auto& v = *(std::vector<int>*)((char*)(&name) + sizeof(std::string));
    for (int x : v) {
        std::cout << x;
    }
}
```

Код выше выведет содержимое вектора `tokens` из объекта `User`. И в этом даже нет ничего противозаконного: никаких dangling references и use-after-free. Ссылка на поле обеспечивает продление жизни всего объекта. И это может быть ссылка на [сколь угодно вложенное поле](https://godbolt.org/z/d8ejKfhWe):

```C++
struct Name {
    std::string name;
};

struct User {
    Name name;
    std::vector<int> tokens;
};

...

int main() {
    std::string&& name = get_user().name.name;
    ...
}
```

И вложенные поля даже могут [быть массивами](https://godbolt.org/z/jsT1efThW)!
(C arrays, с std::array работать не будет из-за перегрузки operator[]!)

```C++
struct Name {
    std::string name;
};

struct User {
    Name name[2];
    std::vector<int> tokens;
};

User get_user() {
    return {
        { "Dmitry", "Dmitry" },
        {1,2,3,4,5}
    };
}

int main() {
    std::string&& name = get_user().name[1].name;
    ...
}

```

Здорово! Но пытливый читатель уже, наверное, догадался, в чем проблема:
Мы берем ссылку на только одно поле, и, наверное, собираемся работать только с ним одним, а объект остается жить целиком... А что если остальные его поля держат выделенную память? А что если нам **критически важно**, чтоб у них был вызван деструктор?

Для наглядной демонстрации проблемы я приведу пример, внезапно, не на C++, а на Rust, поскольку там необходимый для создания неприятностей тип есть из стандартной коробки, ровно как и изысканно сломанная синтаксическая конструкция:

```Rust
use parking_lot::Mutex;

#[derive(Default, Debug)]
struct State {
    value: u64,
}

impl State {
    fn is_even(&self) -> bool {
        self.value % 2 == 0
    }

    fn increment(&mut self) {
        self.value += 1
    }
}

fn main() {
    let s: Mutex<State> = Default::default();

    match s.lock().is_even() {
        true => {
            s.lock().increment(); // oops, double lock!
        }
        false => {
            println!("wasn't even");
        }
    }
    dbg!(&s.lock());
}
```
Этот [пример](https://play.rust-lang.org/?version=stable&mode=release&edition=2021&gist=31f87adf34e0e6c490a46991e3d81a5d) уходит в дедлок: временный объект `LockGuard` в операторе `match` остается жив по совершенной нелепости! Подробнее можно почитать [тут](https://fasterthanli.me/articles/a-rust-match-made-in-hell). А мы же вернемся к C++

Если мы по какой-то причине решили последовать примеру Rust и сделать `mutex` явно ассоциированный с данными (как и должно быть в 95% случаев), мы получаем такую же [проблему](https://godbolt.org/z/3b4Ms9Yx3) при неаккуратном использовании ссылок:

```C++
template <class T>
struct Mutex {
    T data;
    std::mutex _mutex;
    
    explicit Mutex(T data) : data {data} {}
 
    auto lock() {
        struct LockGuard {
        public:
            LockGuard(T& data, std::unique_lock<std::mutex>&& guard) : data(data), guard(std::move(guard)) {}
            std::reference_wrapper<T> data;
        private: 
            std::unique_lock<std::mutex> guard;
        };

        return LockGuard(this->data, std::unique_lock{_mutex});
    }
};



int main() {
    Mutex<int> m {15};

    // double lock (deadlock, ub) due to LockGuard lifetime extension, remove && and it will be fine
    auto&& data = m.lock().data;
    std::cout << data.get() << "\n";
    auto&& data2 = m.lock().data;
    std::cout << data2.get() << "\n";
}
```

"Ну тут же сам себе злой буратино" -- скажут опытные защитники C++: "Зачем ссылка, если там и так reference_wrapper". И будут, разумеется правы. Но не переживайте. В C++23 (а также С++20, потому что ее бекпортировали) теперь есть такая же сломанная конструкция как и `match` в Rust. И это... [`range-based-for`](for_loop.md)!

Удивительнейшим образом изменения в стандарте, направленные на то чтоб починить висячую ссылку в конструкции

```C++
for (auto item : get_object().get_container()) { ... }
```
Теперь позволяют вляпаться в точно такой же дедлок как в Rust

```C++
template <class T>
struct Mutex {
    T data;
    std::mutex _mutex;
    
    explicit Mutex(T data) : data {data} {}
 
    auto lock() {
        struct LockGuard {
        public:
            LockGuard(T& data, std::unique_lock<std::mutex>&& guard) : data(data), guard(std::move(guard)) {}
            std::reference_wrapper<T> data;

            T& get() const {
                return data.get();
            }
        private: 
            std::unique_lock<std::mutex> guard;
        };

        return LockGuard(this->data, std::unique_lock{_mutex});
    }
};

struct User {
    std::vector<int> _tokens;

    std::vector<int> tokens() const {
        return this->_tokens;
    }
};

int main() {
    Mutex<User> m { { {1,2,3, 4,5} } };

    for (auto token: m.lock().get().tokens()) {
        std::cout << token << "\n";
        m.lock(); // deadlock C++23 (C++20 after backport!)
    }
}

```

Самое замечательно в этом всем то, что в данный момент это "исправленное" поведение еще не реализовано в основных компиляторах. Но скоро, лет через пять, когда вы их обновите... Вас может ждать много удивительных открытий!


## Полезные ссылки
1. https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2718r0.html
2. https://fasterthanli.me/articles/a-rust-match-made-in-hell
3. https://en.cppreference.com/w/cpp/language/lifetime