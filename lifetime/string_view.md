# string_view -- тот же const& только больнее

С++17 подарил нам тип `std::string_view`, призванный убить сразу двух зайцев:
- Проблемы с перегрузками для функций, которые должны работать хорошо как с C, так и с C++ строками
- А также программистов, периодически забывающих о правилах продления жизни временным объектам.

И так, проблема: функция хочет считать количество вхождений какого-то символа в строку:

```C++
int count_char(const std::string& s, char c) {
    ....
}

count_char("hello world", 'l'); // создастся временный объект std::string, 
// выделится память, скопируется строка, а потом строка умрет и память
// деаллоцируется --- плохо, много лишних операций
```

Так что нам нужна перегрузка для С-строк
```C++
int count_char(const char* s, char c) {
    // мы тут не знаем ничего про длину строки
    // она вообще null-териминированная?
    
    // Можем только написать код, наивно рассчитывающий, что его
    // будут вызывать правильно.
    ...
}
```

И будем либо дублировать код, немножно адапритуя его под C-строки, либо сделаем функцию

```C++
int count_char_impl(const char* s, size_t len, char c) {
   ... 
}
```

В которую поместим весь дублирующийся код и вызовем ее из перегрузок:
```C++
int count_char(const std::string& s, char c) {
    return count_char_impl(s.data(), s.size(), c);
}

int count_char(const char* s, char c) {
    return count_char_impl(s, strlen(s), c);
}
```

И тут на помощь приходит `string_view`, как раз таки являющийся парой: указатель и размер. И убивает обе перегрузки:

```C++
int count_char(std::string_view s, char c) {
    ...
}
```

И все здорово, хорошо и замечательно, кроме одного но:

`std::string_view` по сути является ссылочным типом, как `const&`, и его можно конструировать из временных значений. Но, в отличие от просто `const&`, никакого продления жизни не будет. Вернее будет, но не там, где [ожидается](https://godbolt.org/z/nxxrYb).

```C++
auto GetString = []() -> std::string { return "hello"; };
std::string_view sv = GetString();
std::cout << sv << "\n"; // dangling reference!
```

В этом примере мы, конечно, почти явно стреляем себе в голову. Можно сделать стрельбу [менее явной](https://godbolt.org/z/PPcarE):

```C++
std::string_view common_prefix(std::string_view a, std::string_view b) {
    auto len = std::min(a.size(), b.size());
    auto common_count = [&]{
        for (size_t common_len = 0; common_len < len; ++common_len) {
            if (a[common_len] != b[common_len]) {
                return common_len;
            }
        }
        return len;
    }();
    return a.substr(0, common_count);
}


int main() {
    using namespace std::string_literals;
    {
       auto common = common_prefix("helloW", 
                                   "hello"s + "World111111111111111111111");
       std::cout << common << "\n"; // ok
    }
    {
       auto common = common_prefix("hello"s + "World111111111111111111111111", 
                                   "helloW");
       std::cout << common << "\n"; // dangling ref
    }
}
```

Сутация такая же, как с [ранее рассмотренным](use_after_free_in_general.md) `std::min`.
Только защититься от такой функции `common_prefix`, обернув ее в шаблон с помощью анализа rvalue/lvalue,
намного сложнее: нам нужно разобрать случаи `const char*` и `std::string` для каждого аргумента -- в общем, все то, от чего нас введение `std::string_view` "избавило".


Влететь в `string_view` можно еще изящнее:

```C++
struct Person {
    std::string name;

    std::string_view Initials() const {
        if (name.length() <= 2) {
            return name;
        }
        return name.substr(0, 2); // copy -- dangling reference!
    }
};
```

Причем [видно](https://godbolt.org/z/TPc4zq), что Clang хотя бы выдает предупреждение.
А gcc -- нет.

Все потому что `std::string_view` настолько легендарный, что в clang сделали хоть какой-то lifetime checker сперва для него.


## Полезные ссылки
1. https://quuxplusone.github.io/blog/2018/03/27/string-view-is-a-borrow-type/
2. https://foonathan.net/2017/03/string_view-temporary/
3. https://www.learncpp.com/cpp-tutorial/6-6a-an-introduction-to-stdstring_view/
4. https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetime.pdf