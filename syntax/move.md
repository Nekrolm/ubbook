# Семантика перемещения

Начиная с C++11, у нас есть rvalue-ссылки и семантика перемещения. Причем
перемещение не деструктивно: исходный объект остается жив, что порождает
множество ошибок.
Еще есть проблемы с тем, как избегать накладных расходов при использовании
перемещаемых объектов, но с этим можно жить.

## Накладные расходы

Несмотря на все громкие заявления, абстракции в C++ имеют далеко не нулевую стоимость.
Занятным примером является `std::unique_ptr`, завязанный на семантику перемещения.

```C++
void run_task(std::unique_ptr<Task> ptask) {
    // do something
    ptask->go(); 
}

void run(...){
    auto ptask = std::make_unique<Task>(...);
    ...
    run_task(std::move(ptask));
}
```
При вызове `run_task` параметр передается по значению: создается новый объект `unique_ptr`, а старый останется,
но окажется пустым. Раз два объекта, то и два вызова деструктора. С деструктивной 
семантикой перемещения (например, в Rust) вызов деструктора будет только один.

Можно исправить ситуацию -- передать по rvalue-ссылке:

```C++
void run_task(std::unique_ptr<Task>&& ptask) {
    // do something
    ptask->go(); 
}
```

Тогда дополнительного объекта не будет. И произойдет только один вызов деструктора.
При этом, из-за ссылки, имеется дополнительный уровень индирекции и обращение к памяти.

Но самое главное: никакого перемещения [на самом деле не будет](https://godbolt.org/z/4bbrh1), что может скрыть ошибку в логике программы:

```C++
void consume_v1(std::unique_ptr<int> p) {}
void consume_v2(std::unique_ptr<int>&& p) {}

void test_v1(){
    auto x = std::make_unique<int>(5);
    consume_v1(std::move(x));
    assert(!x); // ok
}

void test_v2(){
    auto x = std::make_unique<int>(5);
    consume_v2(std::move(x));
    assert(!x); // fire!
}
```

И мы переходим к основной проблеме

## Use-after-move

Во-первых, функция `std::move` ничего не делает. 
Это всего лишь явное преобразование lvalue-ссылки в rvalue. Оно никак не влияет на состояние объкта.
Обозреваемые эффекты от перемещения могут давать функции, работающие с этой самой rvalue-ссылкой. В основном это конструкторы и операторы перемещения.

Во-вторых, стандарт C++ не специфицирует состояние, в котором должен остаться объект, _из_ которого произвели перемещение. 
Оно должно быть валидным в смысле вызова деструктора. Но более ничего не требуется. Объект не обязан быть пустым после перемещения. Его поля не обязаны быть зануленными. Так у `std::thread` после перемещения нельзя вызывать ни один из методов. А `std::unique_ptr` гарантированно становится пустым (`nullptr`).

Чаще всего и проще всего натолкнуться на use-after-move можно при реализцации конструкторов, заполняющих поля переданными аргументами -- достаточно дать одинаковые (или почти одинаковые) имена полям и аргументам.

```C++
struct Person {
public:
    Person(std::string first_name, 
           std::string last_name) : first_name_(std::move(first_name)),
                                    last_name_(std::move(last_name)) {
        std::cerr << first_name; // wrong, use-after-move
    }
private:
    std::string first_name_;
    std::string last_name_;
};
```

Конечно, в таком случае ошибка будет быстро найдена -- для `std::string` есть гарантия, что после перемещения объект окажется пустым. Но если сделать конструктор шаблонным и передавать в него тривиально перемещаемые типы, ошибка долго может не проявляться.

```C++
    template <class T1, class T2>
    Person(T1 first_name, 
           T2 last_name) : first_name_(std::move(first_name)),
                           last_name_(std::move(last_name)) {
        std::cerr << first_name; // wrong, use-after-move
    }
    ...

    Person p("John", "Smith"); // T1, T2 = const char*
```

Другой интересный случай использования после перемещения -- self-move-assignment.
В результате которого из объекта могут внезапно пропадать данные. А могут и не пропадать. В зависимости от того, как
реализовали перемещение для конкретного типа.

Так, например, вот такая наивная реализация алгоритма `remove_if` содержит ошибку:

```C++
template <class T, class P>
void remove_if(std::vector<T>& v, P&& predicate) {
    size_t new_size = 0;
    for (auto&& x : v) {
        if (!predicate(x)) {
            v[new_size] = std::move(x); // self-move-assignment!
            ++new_size;
        }
    }
    v.resize(new_size);
}
```

[Ошибка](https://godbolt.org/z/qY5MMn) не даст о себе знать до тех пор, пока элементы контейнера не будут содержать 
полей, не учитывающих возможность самоприсваивания.

```C++
struct Person {
    std::string name;
    int age;
};

std::vector<Person> persons = {
    Person { "John", 30 }, Person { "Mary", 25 }
};
remove_if(persons, [](const Person& p) {  return p.age < 20; });

for (const auto& p : persons){
    std::cout << p.name << " " << p.age << "\n"; // все name пустые!
}
```

Отследить использование после перемещения способны некоторые статические анализаторы.
Для clang-tidy тоже [есть проверки](https://clang.llvm.org/extra/clang-tidy/checks/bugprone-use-after-move.html).

Если вы реализуете перемещаемые классы и хотите учесть возможность самоприсваивания/самоперемещения, либо используйте идиому copy/move-and-swap, либо не забывайте проверить совпадение адресов текущего и перемещаемого объектов:

```C++
MyType& operator=(MyType&& other) noexcept {
    if (this == std::addressof(other)) { // addressof сработает, 
                                         // если у вас перегружен &
        return *this; 
    }
    ...
}
```


## Полезные ссылки
1. https://clang.llvm.org/extra/clang-tidy/checks/bugprone-use-after-move.html
2. https://youtu.be/rHIkrotSwcc?t=1065
3. https://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object
4. https://herbsutter.com/2020/02/17/move-simply/