# std::function

С++11 предоставил разработчикам очень удобный класс-шаблон для описания абстрактных вызываемых объектов:

```C++
std::function<R(Args)> f = /* все что угочно, 
                            что можно вызвать как f(args) 
                            и результатом будет R */ 
```

Благодаря технике _type-erasure_ (стирание типа), `std::function` может хранить в себе что угодно. У этого, конечно, есть цена — посредственная производительность: конкретный вызываемый объет должен быть перемещен в кучу, выделение памяти, динамическая диспетчеризация вызова... Если мы не пишем чего-то высоконагруженного, то цена не очень высока.

Однако, благодаря тому же самомуму стиранию типов и тому, как оно реализовано, `std::function` обладает еще некоторыми потрясающими спецэффектами!

### Спецэффект 1. Вариантность

С этим понятием знакомы далеко не все разработчики, так что начнем с примера.

Пожалуй, я не позволю себе использовать забитый пример с классами `Animal` и `Dog`. Вместо этого у меня будет `InputDevice` и `Keyboard`. Вопрос: если `Keyboard` это подтип `InputDevice`, то является ли `Containter<Keyboard>` подтипом `Container<InputDevice>`?

С точки зрения абстрактной достаточно высокоуровневой иерархии в системе типов — вполне себе. Коробка клавиатур это коробка устройств ввода. И так, например, будет в языках Java, Kotlin или Rust 

```Rust
// https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=679a17722f69cf5178fb751f42c8a543
trait InputDevice {
    fn input(&self) -> char;
}

trait Keyboard: InputDevice {
    fn lock_keys(&self);
}

struct MBox<T>(T);

// Коробка клавиатур это коробка устройств ввода!
// Keyboard       is subtype of InputDevice
// MBox<Keyboard> is subtype of MBox<InputDevice>
fn relabel(b: MBox<impl Keyboard>) -> MBox<impl InputDevice> {
    b
}
```

Теоретики скажут, что коробка ковариантна типу содержимого! Типы контейнеров и типы содержимого вложены согласованно. Поэтому **ко**вариантна.

А бывает и обратная ситуация. Например, работник, который умеет чинить клавиатуры, совсем не факт что может чинить произвольные устройства ввода. А вот если наоборот — то без проблем. Он и клавиатуру вам починит.

```Rust
// https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=ef93662f6bf210b61483a06ac56b4b4a
trait Senior {
    fn repair(&self, _: impl InputDevice);
}
trait Junior {
    fn repair(&self, _: impl Keyboard);
}
impl <M: Senior> Junior for M {
    fn repair(&self, k: impl Keyboard) {
        Senior::repair(self, k)
    }
}

// Keyboard                     is subtype of InputDevice
// Senior (repairs InputDevice) is subtype of Junior (repairs only keyboards)
fn demote(m: impl Senior) -> impl Junior {
    m
}
```
Вложенность типов работников противоположна вложенности типов объектов, с которыми они работают. Они **контра**вариантны.


Шаблоны в C++ **ин**вариантны: отношения между типами-контейнерами никак не зависят от типов элементов... Так по крайней мере должно быть и так и задумывалось. Если бы нe внезапное исключение в виде `std::function`!

```C++
// https://godbolt.org/z/oecYYcfvj
class InputDevice {
public:
    virtual ~InputDevice() = default;
    virtual char input() = 0; 
};

class Keyboard : public InputDevice {
public:
    char input() override {
        return '*';
    }
};

// мы можем сделать так
Keyboard* keyboard = ...;
// ведь клавиатура это устройство ввода
InputDevice* device = keyboard;

std::vector<Keyboard*> box_of_keyboards { keyboard, keyboard, keyboard };
// Compilation error. Шаблоны инвариантны. Такого оператора присваивания нет
std::vector<InputDevice*> box_of_input_devs = box_of_keyboard; 


// Но!
std::function<Keyboard* ()> keyboard_maker = ...;
// Отлично компилируется! std::function ковариантен по возвращаемому значению
std::function<InputDevice* ()> input_device_maker = keyboard_maker;
std::function<void(InputDevice*)> senior = ...;
// Тоже компилируется! std::function контравариантен по принимаемым параметрам
std::function<void(Keyboard*)> junion = senior;
```

С одной стороны выглядит очень даже здорово и правильно. А с другой стороны оно, разумеется, работает не потому что `std::function` на самом деле поддерживает вариантность. Так происходит из-за неявного приведения типов аргументов и возвращаемого значения, а также повторного стирания типов (со всеми накладными расходами).

```C++
std::function<void(InputDevice*)> senior = [](auto){}; // Аллокация и перемещение лямбды на кучу! Стерли тип лямбды внутри
std::function<void(Keyboard*)> junion = std::move(senior); // Типы разные. Шаблоны инвариантны. Еще одна аллокация! и перемещенине исходной std::function на кучу. Стираем ее тип.
```
А если цепочки передачи таких функций с изменением типов будут более длинными — становится уже не так здорово.

И проблему можно усугубить тем, что такая "вариантность" работает не только лишь с указателями/ссылками на наследуемые классы в сигнатуре функции. Как было замечено ранее: оно работает из-за неявного приведения типов. А значит мы можем сделать так

```C++
// https://godbolt.org/z/cjfTz5dEW
std::function<void(std::string_view)> by_view = [](std::string_view v){
    std::cout << v;
};
std::function<void(const std::string&)> by_str_ref = by_view;
std::function<void(std::string)> by_str_val = by_str_ref;
std::function<void(const char*)> by_char_ptr = by_str_val;
by_char_ptr("hello");
```
И если туда попадет `nullptr`... Мы помним что будет:
```
Program returned: 139
Program stderr
terminate called after throwing an instance of 'std::logic_error'
  what():  basic_string: construction from null is not valid
Program terminated with signal: SIGSEGV
```
И разумеется при наличии таких цепочек сохранять переданные в аргументах ссылки становится особенно сомнительным занятием.

Проблему с переизбытком аллокаций C++26 пердлагает решать с помощью `std::function_ref` — невладеющих ссылок на вызаваемый объект. Создайте ваш объект один раз и храните, а дальше передавайте на него ссылку с удобным интерфейсом. Вариантность остается в комплекте с возможностью получить dangling reference на другой `std::function_ref` в цепочке присваиваний.

### Спецэффект 2. Отломанный const

Усердно стирая типы, `std::function` перестарался и подтер пробрасывание `const`. 

```C++
int main() {
    const auto counter = [count = 0]() mutable {
        return count++;
    };
    // cannot call: mutable lambda requires mutable access to operator()
    // counter();
    const std::function<int()> f = counter;
    f(); // this is fine! const is not propagated!
}
```

Ну подтер и подтер... Ничего ж страшного. Код компилируется и вызывается — это ж главное! А некорректный код, который тоже компилируется из-за этого, просто не нужно писать...

О важности правильного пробрасывания const, а особенно с приходом C++23 и deducing this, можно вспомнить один забавный факт о популярном фреймворке Qt: в нем свои свобственные контейнеры с Copy-on-Write поведением по умолчанию. И это приводит к тому, что совершенно очевидная read-only итерация по контейнеру внезапно [вызывает копирование всего контейнера!](https://doc.qt.io/qt-6/containers.html#implicit-sharing-iterator-problem) А также к инвалидациям ссылок и другим занятным последствиям и порчей памяти. 

```C++
// Бах! range based for вызывает begin()/end() методы, 
// которые в не const версии вызывают копирование!
for (const auto& x: qlist) {}; 
// Вот так надежнее и корректнее
for (const auto& x: std::as_const(qlist)) {}; 
```

Если мы теперь вернемся обратно к `std::function` и будем передавать в нее вызываемый объект с существенно разными `const` и non-`const` перегрузками `operator()` мы можем получить довольно неприятные результаты:

```C++
// https://godbolt.org/z/KcWxoYqPo
struct Proxy {
    int& operator()(int index) {
        return data[index];
    }

    int operator()(int index) const {
        return data.at(index);
    }

    std::map<int, int>& data;
};

int main() {
    std::map<int, int> data = { {42, 42}};
    const std::function<int(int)> f = Proxy{data};
    f(43); // expect throw?
    for (auto [k, v]: data){
        std::cout << k << " " <<  v << "\n";
    }
}
```
Но `const` был потерян, так что результатом будет:
```
42 42
43 0
```

В C++26 проблему решили: используйте, пожалуйста `std::copyable_function`.

```C++
// const -- часть сигнатуры! 
// также можно дописывать noexcept -- это еще одно улучшение
std::copyable_function<int(int) const> f = Proxy{data};
```

Чинить старый `std::function` не стали по соображениям обратной совместимости со старым кодом, который 
1. может полагаться на ее странное поведение с проглатыванием const.
2. рисково может использовать `std::function` в C++ ABI — и изменение в сигнатуре `operator()` могут его сломать (хотя и так гарантий по нему не дается)

Читатель может заинтересоваться, а почему для исправленной версии выбрано такое странное название. А это из-за еще одной проблемы `std::function`

### Спецэффект 3. move-only не поддерживается

```C++
std::function<int(int)> f = [data = std::make_unique<int>(42)](int x) { return *data + x; };
```

```
/opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/bits/std_function.h:439:69: error: static assertion failed: std::function target must be copy-constructible
  439 |           static_assert(is_copy_constructible<__decay_t<_Functor>>::value,
      |                                                                     ^~~~~
```

Для решения этой проблемы C++26 ввел `std::move_only_function`. А `std::copyable_function` добавили уже на замену старого `std::function`, который поддерживает только копируемые типы.

Возможно, в C++29 `std::function` будет помечен как устаревший и deprecated.

## Полезные ссылки
1. [Copyable Function proposal](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2548r6.pdf)
2. [Variance](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))
3. [Type Erasure](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Type_Erasure)