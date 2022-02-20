# С++20 unbounded ranges

Поддержка работы с коллекциями как с кастомными, так и со стандартными в C++ от версии к версии все лучше и лучше.

Алгоритмы стандратной библиотеки образца 11 — 17 стандартов работали и работают с парами итераторов, задающих диапазон элементов коллекции.

```C++
const std::vector<int> v = {1,2,3,4,5};
std::vector<int> odds;
std::copy_if(v.begin(), v.end(), std::back_inserter(odds), [](int x){ return x % 2 == 0;});
std::vector<int> squares;
std::transform(odds.begin(), odds.end(), std::back_iserter(squares), [](int x) { return x * x;});
// return squares;
```

Многословно, неудобно.
Да еще и совсем не zero cost — лишние аллокации обычно ни один компилятор C++ не оптимизирует. Но, конечно, мы можем провести все оптимизации самостоятельно — алгоритмы старого STL, работающие с итераторами, довольно гибкие в выборе того, что и как вы хотите сделать.

```C++
std::vector<int> v = {1,2,3,4,5};
v.erase(
    std::remove_if(v.begin(), v.end(), [](int x){ return x % 2 != 0; }),
    v.end()
);
std::transform(v.begin(), v.end(), [](int x){return x * x;});
return v;
```

Отлично, ни одной лишней аллокации! Но все также многословно и путанно. Да еще и странно выглядящая конструкция `erase-remove`.

Большинству людей обычно нужно сначала написать простое и понятное решение, а потом уже его оптимизировать по мере надобности. Простыми и понятными решения, использующие старые алгоритмы над парами итераторов, назвать сложно.

В C++11 появился range-based for. И стало удобно просто итерироваться по коллекции.

```C++
for (auto x : v) {
    // do something with x
}
```

Но так итерироваться можно лишь по всей коллекции. А что если вы хотите от пятого элемента в векторе до десятого?
Пишите цикл со счетчиком. Либо используйте `std::for_each` с парой итераторов.

```C++
std::for_each(v.begin() + 5, v.begin() + 10, [&](auto x) {
    // do something
});
```

Либо вам нужно откуда-то (из книг, курсов или из самого стандарта) узнать, что `range-based-for` автоматически работает для любого объекта, у которого есть методы `begin` и `end`.

`begin()` и `end()` должны возвращать итераторы. Если они возвращают что-то другое, то в 99.9% случаев вы получите ошибку компиляции ([иногда вразумительную](https://godbolt.org/z/r75K98Mjj)). В экзотических случаях может быть что-то [неожиданное](https://godbolt.org/z/xrEqhxrKc). 


Из всего этого возникает вполне здравая идея: а что если для итерирования по части коллекции сделать структуру с итераторами? И для всяких transform, filter, reverse... Ух! Да это же как раз C++20 ranges (изначально [ranges-v3](https://github.com/ericniebler/range-v3)).

И вот мы уже разрабатываем свою библиотеку для удобной работы с коллекциями, также удобно работающую с `range-based-for`.
И все хорошо до тех пор, пока нас не посещает идея: а не сделать ли нам ленивую процедурно генерируемую последовательность с совместимым интерфейсом? Пусть `begin()` вернет стейт генерации, `operator ++` в комбинации с `operator *` будут порождать элементы. А `end()`? А он пусть создаст пустую структуру только для того, чтобы проверить, пора прекратить генерацию или нет.


Например, мы можем сделать "бесконечный" генератор чисел

```C++
struct Numbers {
    struct End {};
    struct Number {
        int x;
        bool operator != (End) const { 
            return true;
        }
        int operator*() const {
            return x;
        }
        Number operator++() {
            ++x;
            return *this;
        }
    };

    explicit Numbers(int start) : begin_{start} {}

    Number begin_;

    auto begin() { return begin_; }
    End end() { return {}; }
};
```

И вот тут начинается засада.

Ни старые алгоритмы STL, ни range-based-for не работают — [не компилируются](https://godbolt.org/z/vWEGfvdPh).
Потому что требуют, чтобы `begin` и `end` имели одинаковый тип.

Хорошо, мы можем исправить это относительно [безболезненно](https://godbolt.org/z/h9jado937) в нашем простеньком примере:

```C++
struct Numbers {
    struct Number {
        int x;
        bool operator != (Number) const { 
            return true;
        }
        int operator*() const {
            return x;
        }
        Number operator++() {
            ++x;
            return *this;
        }
    };

    explicit Numbers(int start) : begin_{start} {}

    Number begin_;

    auto begin() { return begin_; }
    auto end() { return begin_; }
};
```
Правда, семантика `operator !=` стала странной. Да и нужно `end()` из чего-то конструировать. Если стейт нашего генератора будет более сложным, например, выделяющим что-то на куче, мы получим дополнительные накладные расходы. Не очень zero-cost.

Поэтому в C++17 `range-based-for` исправили. Теперь он может [работать](https://godbolt.org/z/7vYxc4K6q) с граничиными итераторами разных типов.

Но STL-алгоритмы все также [не работают](https://godbolt.org/z/MddGYWMdq).
```C++
   auto nums = Numbers(10);
   auto pos = std::find_if(nums.begin(), nums.end(), [](int x){ return x % 7 == 0;}); // Compilation error
   std::cout << *pos;
```

В С++20 наконец-то все пофиксили. Нет, старые STL-алгоритмы все также не работают. Просто теперь есть новые STL-алгоритмы, почти такие же, как старые, толко в пространстве имен `std::ranges` и жестко требующие удовлетворения новых концептов итераторов.
Поэтому пример ниже слегка распухает.

```C++
struct Numbers {
    struct End {

    };
    struct Number {
        using difference_type = std::ptrdiff_t;
        using value_type = int;
        using pointer =	void;
        using reference = value_type;
        using iterator_category = std::input_iterator_tag;

        int x;
        bool operator == (End) const { 
            return false;
        }

        int operator*() const {
            return x;
        }
        Number& operator++() {
            ++x;
            return *this;
        }
        Number operator++(int) {
            auto ret = *this;
            ++x;
            return ret;
        }
        
    };

    explicit Numbers(int start) : begin_{start} {}

    Number begin_;

    auto begin() { return begin_; }
    End end() { return {}; }
};
```
С ними компилируется и [работает](https://godbolt.org/z/efh3qsxMd)
``` C++
    auto nums = Numbers(10);

    auto pos = std::ranges::find_if(nums.begin(), nums.end(), [](int x){ return x % 7 == 0;});
    std::cout << *pos;
```

Что ж. Это было небольшое введение. Теперь мы наконец можем начать отстреливать себе ноги.

# std::unreachable_sentinel

Выдумывать на каждый итератор, генерирующий бесконечную последовательность, новый тип (EndSentinel) для метода `end()` нам, благодаря C++20, не надо. В стандартной библиотеке определен тип `std::unreachable_sentinel_t`, задизайненный именно для этой цели. Он сравним на равенство с любым объектом, "похожим" на ForwardIterator. И результат сравнения всегда отрицательный.

С ним наш пример с числами [упрощается](https://godbolt.org/z/e7MYTvWhT).

```C++
struct Numbers {
    struct Number {
        using difference_type = std::ptrdiff_t;
        using value_type = int;
        using pointer =	void;
        using reference = value_type;
        using iterator_category = std::input_iterator_tag;

        int x;
        int operator*() const {
            return x;
        }
        Number& operator++() {
            ++x;
            return *this;
        }
        Number operator++(int) {
            auto ret = *this;
            ++x;
            return ret;
        }
        
    };

    explicit Numbers(int start) : begin_{start} {}

    Number begin_;

    auto begin() { return begin_; }
    auto end() { return std::unreachable_sentinel; } // !
};
```

Сравнение с `unreachable_sentinel` не требует выполнения никаких операций. Так что его можно использовать, например, чтобы сформировать range, итерирование по которому будет происходить без проверки границ.

Например

```C++
    // Если у нас есть вектор, задающий перестановку.
    std::vector<size_t> perm = { 1, 2, 3, 4, 5, 6, 7, 8, 9};
    std::ranges::shuffle(perm, std::mt19937(std::random_device()()));

    // И нам поуступают запросы на поиск позиции элемента, заведомо находящегося в векторе
    // size_t p = 7;
    assert(p < perm.size())
    return std::ranges_find(perm.begin(), std::unreachable_sentinel, p) - perm.begin();
```

Очевидно, это крайне небезопасный ход. К которому стоит пребегать только в случае, если вы точно все проверили и эта оптимизация критична и необходима. Если в примере выше по какой-то причине будет запрошен элемент, не присутствующий в векторе, мы получим [неопределенное поведение](https://godbolt.org/z/459Y68PcW).

Рефакторинг больших участков кода, использующего подобные фичи, может закончиться поиском трудноуловимых багов. В отличие от Rust, в C++ мы не можем гарантированно пометить участок кода, как потенциально опасный и проблемный. В C++ любой участок кода потенциально небезопасен и подчеркнуть это можно только комментарием или какими-нибудь ухищрениями в именовании функций или переменных.

### Ссылки
1. https://en.cppreference.com/w/cpp/iterator/unreachable_sentinel_t
2. https://www.modernescpp.com/index.php/c-20-the-ranges-library
3. https://habr.com/ru/company/otus/blog/456452/
