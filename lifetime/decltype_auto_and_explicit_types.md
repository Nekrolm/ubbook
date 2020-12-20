# Хоть auto, хоть не auto, все равно ссылка висячая!

При работе со стандартными контейнерами приходится иметь дело с очень длинными и громоздкими именами типов (`std::vector<std::pair<T1, T2>>::const_iterator`).

Начиная с 11 стандарта, ранее бестолковое ключевое слово `auto` используется для указания компилятору автоматически вывести тип переменной (или, начиная с 14 стандарта, возвращаемого значения функции). Также есть конструкция `decltype(auto)`, работающая точно так же, но по-другому.

Не все любят автоматическое выведение типов в C++. Призывают указывать явно, так как код становится более понятным. Особенно в случае возвращаемого значения функции:

```C++
template <class T>
auto find(const std::vector<T>& v, const T& x) {
    ...
    // очень длинное тело со множеством return 
    ...
}
```
Что такое это `auto` в данном конкретном случае? `bool`? индекс? итератор? еще что-то более страшное и сложное? Лучше б явно указали...

Но выбор между указать явно или использовать автовывод, на самом деле, не так прост. В обоих случаях можно влететь в ту или иную проблему.

---
## Проблема явного указания типа

Длинно и много писать -- решается с помощью `using`-псевдонимов. Так что это не проблема. Другое дело, что изменение типа в одном месте потребует синхронизированных изменений в других местах. 

И все могло быть хорошо: поменяли где-то в объявлении, получили ошибки компиляции -- исправили во всех местах, на которые указали ошибки. Но в C++ есть неявное приведение типов, которое особенно жестоко наказывает при использовании ссылок.

```C++
std::map<std::string, int> counters = { {"hello", 5}, {"world", 5} };
std::vector<std::string_view> keys; // получаем список ключей, используем string_view, чтобы не делать лишних копий
keys.reserve(counters.size());
std::transform(std::begin(counters), 
               std::end(counters),
               std::back_inserter(keys),
               [](const std::pair<std::string, int>& item) -> std::string_view {
                   return item.first;
               });
// как-то обрабатываем список ключей:
for (std::string_view k : keys) {
    std::cout << k << "\n"; // UB! dangling reference!
}
```

Мы немного ошиблись в аргументе лямбда-функции и получили ссылку на временный объект, а вместе с ней -- [неопределенное поведение](https://godbolt.org/z/EKcob3).

[Исправляем](https://godbolt.org/z/E6evof) ошибку, добавляя `const` перед `string`:
```C++
[](const std::pair<const std::string, int>& item) -> std::string_view 
```

Проходят недели, код рефакторится. `counters` отъезжают в поле какого-нибудь класса. Получение и обработка ключей -- в его второстепенный метод. А потом внезапно выясняется, что тип значений в ассоциативном массиве надо бы поменять на меньший -- пусть `short`.

Вы меняете его. Уже не помните про метод обработки ключей. Компилятор не ругается. Запускаете тестирование и возвращаетесь к той же самой [ошибке](https://godbolt.org/z/7Mzcs3).

В этом примере, решением будет замена явного типа аргумента лямбда-функции на `const auto&`.

Другой пример, но уже не с аргументом, а с возвращаемым значением:

```C++
template <class K, class V>
class MapWrapper : private std::map<K, V> {
public:
    template <class Predicate>
    const std::pair<K, V>& find_if_or_throw(Predicate p) const {
        auto item = std::find_if(cbegin(), cend(), p);
        if (item == cend()) {
            throw std::runtime_error("not found");
        }
        return *item;
    }
}
```
Опять ошиблись. Опять надо исправлять. 
`std::map` может в будущем поменяться на что-то другое, у чего итератор возвращает уже не настоящий `pair`, а прокси-объект. Универсальным решением будет в этом случае -- `decltype(auto)` в качестве возвращаемого значения.


## Проблемы автоматического вывода типа

Мы можем использовать автовывод как минимум в 4 различных формах

1. Голый `auto`. Минимум проблем. В результате всегда получается тип без ссылок:
```C++
auto x = f(...); // но может быть не то, чего вы хотите: копия вместо ссылки

class C {
public:
   auto Name() const {return name;} // копия вместо ссылки
private:
   T name;
};
```
2. `const auto&` -- может забиндиться к висячей ссылке:
```C++
   const auto& x = std::min(1, 2);
   // x -- dangling reference
```
В качестве возвращаемого значения функции `const auto&` использовать в 90% случаев [не выйдет](https://godbolt.org/z/hqf3nK) при правильно настроенных предупреждениях компилятора.

3. `auto&&` -- universal/forwarding reference. Точно также может забиндиться к висячей ссылке:
```C++
   auto&& x = std::min(1, 2);
   // x -- dangling reference
```
С возвращаемым значением -- [аналогично](https://godbolt.org/z/qx8e1M) варианту `const auto&`

4. `decltype(auto)` -- автовывод "как объявлено": справа ссылка -- слева ссылка. Справа нет ссылки -- слева нет ссылки. В каком-то смысле то же самое, что и `auto&&` при объявлении переменных, но [не совсем](https://godbolt.org/z/Yorrjo):
```C++
    auto&& x = 5;
    static_assert(std::is_same_v<decltype(x), int&&>);
    decltype(auto) y = 5;
    static_assert(std::is_same_v<decltype(y), int>);
```
Разница в том, что `auto&&` -- всегда ссылка, а `decltype(auto)` -- "как объявлено в возвращаемом значении". Что может быть важно при дальнейших вычислениях над типами.

`decltype(auto)` начинает стрелять при использовании его в качестве возвращаемого значения, требуя дополнительной внимательности при написании [кода](https://godbolt.org/z/PPcPYK):
```C++
class C {
public:
    decltype(auto) Name1() const {
        return name; // копия. name объявлен как T
    }

    decltype(auto) Name2() const {
        return (name); // ссылка. Выражение (name) имеет тип const T&: 
                       // само по себе (name) -- T&, но this помечен const, поэтому
                       // получается const T&
    } 

    decltype(auto) ConstName() const {
        return const_name; // const копия. const_name объявлен как const T
    }

    decltype(auto) DataRef() const {
        return data_ref; // DataT&, как объявлено.
        // return (data_ref); будет то же самое. const от this не распространяется дальше под
        // поля-ссылки и указатели.
    }

    decltype(auto) DanglingName1() const {
        auto&& local_name = Name1(); // возвращает копию. Копия прибивается к rvalue ссылке
        return local_name; // local_name -- ссылка на локальную переменную
    }

    decltype(auto) DanglingName2() const {
        auto local_name = Name1(); // возвращает копию.
        return (local_name); // (local_name) -- ссылка на локальную переменную
    }

    decltype(auto) NonDanglingName() const {
        decltype(auto) local_name = Name1(); // возвращает копию.
        return local_name; // возвращает копию
    }
private:
    T name;
    const T const_name;
    DataT& data_ref;
};
```

`decltype(auto)` это хрупкий и тонкий механизм, способный перевернуть все с ног на голову с помощью минимального изменения в коде -- "лишних" скобок или `&&`.

---
## Полезные ссылки
1. https://docs.microsoft.com/en-us/cpp/cpp/decltype-cpp?view=msvc-160
2. https://quuxplusone.github.io/blog/2018/04/05/lifetime-extension-grudgingly-accepted/
3. https://stackoverflow.com/questions/24109737/what-are-some-uses-of-decltypeauto
