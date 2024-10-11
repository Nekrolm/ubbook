# std::shared_from_this

Обсуждая особенности [`std::make_shared`](shared_ptr_constructor.md), я упоминул, что иногда крайне необходимо убедиться, что объекты вашего класса всегда создаются только в куче и управляются с помощью умного указателя. Вот сейчас будет еще один такой случай.

Вы разрабатываете графический интерфейс и, как это было принято лет 20 назад, решили что все доллжно быть объектно-ориентированно и красиво. Компонентики. Виджеты. Всех мы будем по требованию создавать, управлять умными shared и weak указателями, чтоб не очень сильно задумываться о владении — очень стандартный подход, между прочим. Rust библиотеки для GUI, например, часто [критикуют](https://www.warp.dev/blog/why-is-building-a-ui-in-rust-so-hard) за чудовищную сложность именно из-за владений.

И так у вас появились некоторые базовые типы:

```C++
enum class EventType {
    Clicked,
    Created,
    // и другие
};

class Widget {
public:
    virtual ~Widget() = default;
};

class EventListener {
public:
    // Уведомить Listener, что такой-то Widget породил некоторое событие
    void notify(EventType, std::weak_ptr<Widget> event_source);
};
```

Хорошо. Дальше давайте заведем кнопку! Куда же без кнопки в хорошем UI?!

```C++
class Button: public Widget {
public:
    Button(std::shared_ptr<EventListener> listener): listener_ {listener} 
    {
        // (1) хотелось бы уведомить listener, что кнопка создана!
        listener_->notify(EventType::Created, this); // Это неправильно...
    }

    void click() {
        // (2) хотелось бы уведомить listerner, что на кнопочку нажали!
        listener_->notify(EventType::Clicked, this); // И это разумеется неправильно
    }

private:
    std::shared_ptr<EventListener> listener_;
};
```

К нашему большому счастью, этот [код не компилируется](https://godbolt.org/z/v77aG9rhz)

```
<source>:26:47: error: cannot convert 'Button*' to 'std::weak_ptr<Widget>'
   26 |         listener_->notify(EventType::Created, this); // Это неправильно...
      |                                               ^~~~
      |                                               |
      |                                               Button*
```

Ах, ну да, типы разные... Опытного разработчика на C++ это, конечно, заставило бы задуматься. А вот неопытного... Он может просто выполнить  «преобразование» типов!

```C++
    Button(std::shared_ptr<EventListener> listener): listener_ {listener} 
    {
        // (1) хотелось бы уведомить listener, что кнопка создана!
        listener_->notify(EventType::Created, std::shared_ptr<Button>(this)); // Это ОЧЕНЬ неправильно...
    }

    void click() {
        // (2) хотелось бы уведомить listerner, что на кнопочку нажали!
        listener_->notify(EventType::Clicked, std::shared_ptr<Button>(this)); // И это разумеется ТОЖЕ неправильно
    }
```
И взорвется
```C++
int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = std::make_shared<Button>(listener);
}
```
```
Program returned: 139
free(): invalid pointer
Program terminated with signal: SIGSEGV
```

`std::shared_ptr<Button>(this)` создает новый shared_ptr, который ничего знаеть не знает о существовании другого умного указателя, управляющего объектом. Что разумеется приводит к попытке повторного освобождения памяти: сначала одним указателем, затем другим. Что иногда даже может работать успешно... Неопределенное поведение все-таки!

Хорошо, давайте чинить. Забудем пока про конструктор. Попробуем хотя бы починить метод `click`.

Для этого необходимо сообщить кнопке о том, что она управляется умным указателем. Например, вручную добавить в нее `weak_ptr<Button>` поле и заполнить его после конструирования.
Да, это должна быть слабая ссылка, чтобы не создавать цикл из shared_ptr и сопутствующую ему утечку памяти



```C++
class Button: public Widget {
public:
    // Мы не можем передать weak_ptr<Button> в конструктор! Ведь мы же еще кнопку не создали!
    Button(std::shared_ptr<EventListener> listener): listener_ {listener} {}

    // Придется сделать что-то такое мерзкое
    void set_self(std::weak_ptr<Button> self) {
        self_ = self;
    }

    void click() {
        listener_->notify(EventType::Clicked, self_); 
    }

    void
private:
    std::shared_ptr<EventListener> listener_;
    std::weak_ptr<Button> self_;
};
```

И пользоваться бы этим добром пришлось так
```C++
int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = std::make_shared<Button>(listener);
    button->set_self(button);
    button->click();
}
```

Некрасиво, но работает...


Но С++11 предлагает нам решенине получше! `std::enable_shared_from_this` 
Который позволит нам автоматически добавить такое же self-поле и позаботится о его правильное заполнении 
при конструировании `shared_ptr`.

```C++
class Button: public Widget, public std::enable_shared_from_this<Button> {
public:
    // Мы не можем передать weak_ptr<Button> в конструктор! Ведь мы же еще кнопку не создали!
    Button(std::shared_ptr<EventListener> listener): listener_ {listener} {}

    void click() {
        listener_->notify(EventType::Clicked, this->weak_from_this()); 
        // есть также shared_from_this
    }

private:
    std::shared_ptr<EventListener> listener_;
};

...
int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = std::make_shared<Button>(listener);
    button->click();
}
```
[Оно не падает](https://godbolt.org/z/EvxsaKcvz). Красота!

Но что-то все еще не то. Полная красота не достигнута... Ну разумеется!
Ведь в вашем приложении будут не только кнопки. Но и комбо-боксы, текстовые поля и прочее...
И что к каждому классу поотдельности `public std::enable_shared_from_this<ClassName>` пририсовывать?

Нет. Разумнее прицепить его к базовому классу и забыть. Давайте так сделаем.

```C++
class Widget: public std::enable_shared_from_this<Widget> {
public:
    virtual ~Widget() = default;
};

// Для отладки, добавим печать в notify
class EventListener {
public:
    void notify(EventType, std::weak_ptr<Widget> event_source) {
        std::cout << "event received\n";
        if (auto w = event_source.lock()) {
            std::cout << "widget valid\n";
        }
    }
};
```

Все [работает](https://godbolt.org/z/8fGr5vcPr) как надо
```
Program returned: 0
event received
widget valid
```

Прекрасно. Кстати, нам же не очень-то бы хотелось выпячивать этот метод `shared_from_this()` ведь он же для внутренних нужд... Может, сделать `protected` наследование?

```C++
class Widget: protected std::enable_shared_from_this<Widget> {
public:
    virtual ~Widget() = default;
};
```
И... уже [неправильно](
https://godbolt.org/z/876T4Y8rn), но продолжает компилироваться!
```
Program returned: 0
event received
```
Получили nullptr и рады... Да, ни в коем случае нельзя его наследовать ни приватно, ни защищенно. Так и в документации написано. Настройте себе правило для линтера, пожалуйста. 

Кстати, если у вас будет несколько абстрактных интерфейсов, каждый с `enable_shared_from_this` и вы попытаетесь унаследовать сразу хоть пару из них... Вы получите...

```C++

class Widget: public std::enable_shared_from_this<Widget> {
public:
    virtual ~Widget() = default;
};

class Gadget: public std::enable_shared_from_this<Gadget> {
public:
    virtual ~Gadget() = default;
};
...
class Button: public Widget, public Gadget {
    ...
    void click() {
        listener_->notify(EventType::Clicked, this->weak_from_this()); 
    }
};
```

Правильно! Добро пожаловать в ад вариаций ромбовидного наследования!
```
<source>:38:53: error: member 'weak_from_this' found in multiple base classes of different types
   38 |         listener_->notify(EventType::Clicked, this->weak_from_this()); 
 
```

Неоднозначно, откуда метод брать. Но мы можем устранить неоднозначность

```C++
class Button: public Widget, public Gadget {
    ...
    void click() {
        listener_->notify(EventType::Clicked, static_cast<Widget*>(this)->weak_from_this()); 
    }
};
```
Компилируется. Запускается. [Работает неправильно](https://godbolt.org/z/GEeczGxEE)... Опять nullptr.
```
Program returned: 0
event received
```

Просто не используйте множественное наследование с `std::enabled_shared_from_this`. И всё будет хорошо. 

Ладно. Договоримся, что нас устраивает результат. Мы починили метод `click`. А как насчет конструктора?

```C++
class Button: public Widget {
public:
    Button(std::shared_ptr<EventListener> listener): listener_ {listener} {
        listener_->notify(EventType::Created, this->weak_from_this());
    }

    void click() {
        listener_->notify(EventType::Clicked, this->weak_from_this()); 
    }

private:
    std::shared_ptr<EventListener> listener_;
};

int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = std::make_shared<Button>(listener);
    button->click();
}
```

Работет, [но не так как хочется](https://godbolt.org/z/MY93EjPEa).
```
Program returned: 0
event received
event received
widget valid
```

Из конструктора мы отправили в Listener нулевой указатель... Ну а чего еще мы хотели?! Ведь работа конструктора еще не завершена. А как мы видели из ручной имитации shared_from_this, внутренний weak_ptr инициализируеься только после полного создания объекта. Так что все работает правильно. Хоть и неожиданно.

Так что если хотите посылать сообщения из конструктора таким образом, то желательно перехотеть. И сделать статический фабричный метод вместо конструктора. Из него уже можно будет посылать уведомления сколько угодно.

```C++
class Button: public Widget {
public:

    static std::shared_ptr<Button> create(std::shared_ptr<EventListener> listener) {
        auto button = std::make_shared<Button>(listener);
        listener->notify(EventType::Created, button);
        return button;
    }

    Button(std::shared_ptr<EventListener> listener): listener_ {listener} {}

    void click() {
        listener_->notify(EventType::Clicked, this->weak_from_this()); 
    }

private:
    std::shared_ptr<EventListener> listener_;
};

int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = Button::create(listener);
    button->click();
}
```
Вот теперь [все правильно](https://godbolt.org/z/9795d79vY)

```
Program returned: 0
event received
widget valid
event received
widget valid
```

Осталось пойти и сделать конструтор приватным с помощью привантного-тэга. Ведь иначе кто-нибудь обязательно создаст кнопку на стэке. От чего либо наполучает нулевых указателей

```C++
int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = Button(listener);
    button.click();
}
```
```
Program returned: 0
event received
```

Либо, если вы бесстрашно использовали `shared_from_this()` вместо `weak_from_this()` (который появился только в C++17, если что), то вас ждет что-то более интересное

```C++
class Button: public Widget {
public:
    Button(std::shared_ptr<EventListener> listener): listener_ {listener} {}

    void click() {
        listener_->notify(EventType::Clicked, this->shared_from_this()); 
    }

private:
    std::shared_ptr<EventListener> listener_;
};

int main() {
    auto listener = std::make_shared<EventListener>();
    auto button = Button(listener);
    button.click();
}
```

```
// https://godbolt.org/z/szYd841a1
Program returned: 139
terminate called after throwing an instance of 'std::bad_weak_ptr'
  what():  bad_weak_ptr
Program terminated with signal: SIGSEGV
```

Да, сейчас можно получить исключение. А вот до выхода C++17 никакого исключения не гарантировалось. Только неопределенное поведение. Этот дефект исправили и портировали во все версии стандарта, начиная c С++11. Главное, не используйте очень старые версии компиляторов и стандартной библиотеки в их составе.

## Полезные ссылки

1. https://en.cppreference.com/w/cpp/memory/enable_shared_from_this
2. https://cplusplus.github.io/LWG/issue2529