# std::ranges::views (как-то лениво)

2024 год. C++20 уже как 4 года готов (не совсем) к использованию в серьезной production разработке. По крайней мере мне недавно сообщили, что компиляторы наконец-то обновлены и мы уже можем...

С++20 принес 4 крупные фичи. 2 из них готовы к употреблению в вашем коде сразу, а еще 2 -- не очень. Здесь мы будем говорить о первых двух.

## std::ranges

Революция в методе работы с последовательностями элементов в C++! Последний раз такое было, когда в C++11 range-based-for сделали. И вот теперь опять.

Забудьте о паре итераторов begin/end и мучениях с тем, чтоб лихо и красиво выбросить все нечетные числа, а все четные возвести в квадрат, как это можно сделать в других высокоуровневых языках:

```Rust
let v : Vec<_> = ints.iter()
                     .filter(|x| x % 2 == 0)
                     .map(|x| x * x)
                     .collect();
```

```Java
List<int> v = Stream.of(ints)
      .filter(x -> x % 2 == 0)
      .map(x -> x * x)
      .collect(Collectors.toList());
```

```C#
var v = ints.Where(x => x % 2 == 0)
            .Select(x => x * 2)
            .ToList();
```

```C++
// до C++20
std::vector<int> v;
std::copy_if(ints.begin(), ints.end(), std::back_inserter(v), [](int x) { return x % 2 == 0;});
std::transform(v.begin(), v.end(), v.begin(), [](int x){return x * x;});

// после С++20
std::vector<int> v;
std::ranges::copy(
    ints | std::views::filter([](int x){ return x % 2 == 0;})
         | std::views::transform([](int x) { return x * x;})
    std::back_inserter(v)
);

// после C++23
auto v = 
    ints | std::views::filter([](int x){ return x % 2 == 0;})
         | std::views::transform([](int x) { return x * x;})
         | std::ranges::to<std::vector>();
```

Красота! Только компилируется оно долго, омтимизируется не всегда хорошо, но ничего страшного...

## Concepts

Концепты были нужны и как самостоятельная фича: уж дюже SFINAE в современном C++ необходим то здесь, то там (особенно библиотекописателям), а техника эта тяжелая и для чтения и для написания. И ошибки компиляции чудовищные... Концепты как именованные ограничения должны были улучшить ситуацию.

Так что теперь мы, например, можем написать максимально правильную и, вероятно, понятную generic функцию для сложения целых чисел. И только целых. И это будет явно видно в ее сигнатуре без всякой магии enable_if

```C++
std::integral auto sum(std::integral auto a
                       std::integral auto b) {
    return a + b;
}
```

И для ranges они были страшно необходимы: вся библиотека ranges это невероятная гора шаблонов с ограничениями, без синтаксического сахара концептов прочитать сигнатуры функций было бы невероятно сложно, а в попытках разрешить ошибки компиляции можно было бы провести вечность.


## А теперь стреляем с двух рук!

Один разработчик написал в общий C++ чат, что он работает над добавлением тестов к какой-то новой фиче в библиотеке, которая так здорово реализована с помощью C++20 и ranges, да вот только что-то странное происходит: тесты падают, valgrind что-то совершенно невразумительное говорит...

Код фичи был таков

```C++
struct MetricsRecord {
    std::string tag;
    // ...
};

struct Metrics {
    std::vector<MetricsRecord> records;

    std::ranges::range auto by_tag(const std::string& tag) const;
    // ...
};

// ... много-много кода

std::ranges::range auto Metrics::by_tag(const std::string& tag) const {
    return records | std::ranges::views::filter([&](auto&& r) { return r.tag == tag; });
}
```

Ничего страшного, вроде все нормально. Никаких проблем не видно.

[Но посмотрим тесты](https://godbolt.org/z/Wjqvoc9e5):

```C++
int main() {
    auto m = Metrics {
        {
            {"hello"}, {"world"}, {"loooooooooooooooongtag"}
        }
    };

    {
        // печатает found
        auto found = m.by_tag("hello");
        for (const auto& f: found) {
            std::cout << std::format("found {}\n", f.tag);
        }
    }

    {
        // не печатает... странно
        auto found = m.by_tag("loooooooooooooooongtag");
        for (const auto& f: found) {
            std::cout << std::format("found {}\n", f.tag);
        }
    }

    {
        // а так работает
        std::string tag = "loooooooooooooooongtag";
        auto found = m.by_tag(tag);
        for (const auto& f: found) {
            std::cout << std::format("found {}\n", f.tag);
        }
    }
}
```

Искушенный читатель уже понял, что проблема во временной переменной
```C++
auto found = m.by_tag("loooooooooooooooongtag"); // неявное создание временной переменной std::string в аргументе!
```

и в том что предикат фильтрации захватывет переменную по ссылке
```C++
std::ranges::range auto Metrics::by_tag(const std::string& tag) const {
    return records | std::ranges::views::filter([&](auto&& r) { return r.tag == tag; });
}
```

а еще в том, что разработчик долгое время писал на JavaScript, а там `Array.prototype.filter` сразу же создает новый массив. 
```JavaScript
const words = ['spray', 'elite', 'exuberant', 'destruction', 'present'];

const result = words.filter((word) => {
     console.log(word);
     return word.length > 6
  });
// сразу же будут напечатаны все элементы
```

И разработчик просто не понял сразу, что метод ленивый (и что все std::ranges ленивые).

Всё так просто! Вам нужно просто правильно использовать C++ и проблем не будет! Или будут?..

```C++
std::ranges::range auto Metrics::by_tag(const std::string& tag) const 
```
Можно ли по сигнатуре метода догадаться, что он ленивый? Вряд ли. `std::ranges::range auto` не дает об этом никакой информации.

Это должно быть написано в документирующем комментарии. Но его не было. 
Либо должен был быть использован концет `std::ranges::view`.
Ах, как обычно, вот если бы все было сделано правильно... 

Но зато valgrind поймал ошибку! Да. В тестах к библиотеке. Кто знает, если тесты у пользователей этой библиотеки...

Пусть пишут тесты! Пусть используют статический анализ! Ага. Возможно, они помогут.
**На момент написания этого текста (апрель 2024 года)**: ни clang-tidy, ни PVS-studio не могут диагностировать эту ошибку.

-----

Ну хорошо. Все теперь будут заранее читать документацию и будут знать, что `std::viewes` ленивые. А значит захватывать ссылки с ними нужно осторожно. Закрываем вопрос...

_Секция ниже во многом вдохновлена выступлением [Nicolai Josuttis](https://twitter.com/NicoJosuttis) на [Keynote Meeting C++ 2022](https://www.youtube.com/watch?v=O8HndvYNvQ4)_

Подождите. `std::ranges` не просто ленивые. А невероятно ленивые! Им иногда не просто лень итерироваться по контейнеру, им иногда даже лень у контейнера `begin()` и `end()` лишний раз вызвать. Причина такой лени -- требования стандарта обеспечить, в среднем, константное время выполнения методов `begin()` и `end()`:

```
Given an expression t such that decltype((t)) is T&, T models range only if
(2.1)
[ranges​::​begin(t), ranges​::​end(t)) denotes a range ([iterator.requirements.general]),
(2.2)
both ranges​::​begin(t) and ranges​::​end(t) are amortized constant time and non-modifying, and
```

Поэтому некоторые views
1. Откладывают вызов begin/end у контейнера при конструировании
2. Кэшируют свои begin/end после их первого вычисления.

И получаются интересные спецэффекты:


```C++
// https://godbolt.org/z/n8Paf6svj

void print_range(std::ranges::range auto&& r) {
    for (auto&& x: r) {
        std::cout << x << " ";
    }
    std::cout << "\n";
}

void test_drop_print() {
    std::list<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::drop(2); // пропустить первые два
    ints.push_front(-5);
    print_range(v); // -5 и 1 пропущены. drop вызвал begin и end только сейчас
}

void test_drop_print_before_after() {
    std::list<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::drop(2);
    print_range(v); // 1, 2 пропущены
    ints.push_front(-5);
    print_range(v); // 1, 2 пропущены! drop не вызывает begin и end еще раз
}


void test_take_print() {
    std::list<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::take(2);
    ints.push_front(-5);
    print_range(v); // -5, 1 выведены
}

void test_take_print_before_after() {
    std::list<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::take(2);
    print_range(v); // 1, 2 выведены
    ints.push_front(-5);
    print_range(v); // -5, 1 выведены! take вызывает begin и end каждый раз
}
```

Здорово, совершенно естественно, а главное -- предсказуемо! Нет никакой магии, если знать как оно работает... Главное не ошибиться при использовании на практике.

Просто не надо брать и модифицировать контейнер, когда на него взят ranges::view. Это же так просто.

Кстати, если мы сделаем одно крохотное изменение

```C++
// https://godbolt.org/z/svb91qn8s
void print_range(std::ranges::range auto r) // by value теперь
{
    for (auto&& x: r) {
        std::cout << x << " ";
    }
    std::cout << "\n";
}

void test_drop_print_before_after() {
    std::list<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::drop(2);
    print_range(v); // 1, 2 пропущены
    ints.push_front(-5);
    print_range(v); // -5, 1 пропущены! мы же теперь сделали копию view и копия снова вызвала begin() и end()
}

```

Следующим вытекующим спецэффектом такого ленивого и иногда кэшурующего поведения является то, что в функцию, принимающую `const std::range::range&`, абы какой `view` подставить нельзя

```C++
// https://godbolt.org/z/hznzEqPEh
void print_range(const std::ranges::range auto& r) {
    for (auto&& x: r) {
        std::cout << x << " ";
    }
    std::cout << "\n";
}

void test_drop_print() {
    std::list<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::drop(2);
    print_range(v); // не компилируется! drop от std::list должен быть мутабельным
    /*
    <source>: In instantiation of 'void print_range(const auto:42&) [with auto:42 = std::ranges::drop_view<std::ranges::ref_view<std::__cxx11::list<int> > >]':
    <source>:19:16:   required from here
             19 |     print_range(v);
                |     ~~~~~~~~~~~^~~
    <source>:10:5: error: passing 'const std::ranges::drop_view<std::ranges::ref_view<std::__cxx11::list<int> > >' as 'this' argument discards qualifiers [-fpermissive]
             10 |     for (auto&& x: r) { 
    */

}

void test_drop_print_vector() {
    std::vector<int> ints = {1, 2, 3 ,4, 5};
    auto v = ints | std::views::drop(2);
    print_range(v); // все ок
}
```

Соответсвенно один и тот же абстрактный `view` ни в коем случае нельзя напрямую использовать по ссылке в нескольких потоках. Нужно требовать константности или синхронизировать доступ. Также писатели generic кода должны возложить на себя дополнительную когнитивную нагрузку и правильно указывать концепты-ограничения.

Для начала вот эти четыре
- `std::ranges::range` - слишком абстрактный: только begin и end
- `std::ranges::view`  - тоже range, но ему удовлетворяют только views
- `std::ranges::borrowed_range` - тоже слишком абстрактный, но его итераторы безопасно возвращать из функций
- `std::ranges::constant_range` (С++23) - тоже абстрактный, но итераторы дают только read-only доступ

А потом еще и оставшиеся подключатся.


Последним выдающимся спецэффектом ленивого кэширования является следующий курьез:

```C++
enum State {
    Stopped,
    Working,
    ...
};

struct Unit {
    State state;
    ....
};

...

std::vector<Unit> units; 
...

// stop all working units
for (auto& unit: units | std::views::filter{[](auto& unit){ return unit.state == Working; }}) {
    ...
    unit.state = State::Stopped; // UB!
    // https://eel.is/c++draft/range.filter#iterator-1
    /*
       Modification of the element a filter_view​::​iterator denotes is permitted, but results in undefined behavior if the resulting value does not satisfy the filter predicate.
    */
}
```

Стандарт явно запрещает изменять
элементы найденные с помощью `std::views::filter` так, чтоб результат предиката менялся! Все из-за предположения, что вы, возможно, еще раз будете итерироваться по тому же самому view. И чтобы не делать работу дважды, нужно закэшировать `begin()`

Cамое чудовищное в том, что такое поведение закреплено стандартом. Это не implementation defined:

```
https://eel.is/c++draft/range.filter#view-5
Remarks: In order to provide the amortized constant time complexity required by the range concept when filter_view models forward_range, this function caches the result within the filter_view for use on subsequent calls.
```


## Полезные ссылки
1. https://en.cppreference.com/w/cpp/ranges
2. https://eel.is/c++draft/range.req
3. https://eel.is/c++draft/range.filter#iterator-1
4. https://www.youtube.com/watch?v=O8HndvYNvQ4
