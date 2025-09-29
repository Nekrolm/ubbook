# Невиртуальные виртуальные функции

Вы разрабатываете иерархию классов и хотите описать интерфейс для вычислителя, который можно запускать и останавливать.
Скорее всего в первой итерации он будет выглядеть так

```C++
class Processor {
public:
    virtual ~Processor() = default;
    virtual void start() = 0;
    // stops execution, returns `false` if already stopped
    virtual bool stop() = 0;
};
```

Пользователи интерфейса нареализовывали своих имплементаций. Все были счастливы, пока кто-то не сделал асинхронную реализацию. С ней почему-то приложение стало падать. Проведя небольшое расследование, вы выяснили, что пользователи интерфейса не позаботились вызвать метод `stop()` перед разрушением объекта. Какая досада!

Вы были уставши и злы. А быть может это были и не вы, а какой-то менее опытный коллега, которому поручили доработать интерфейс. В общем, на свет родилась правка

```C++
class Processor {
public:
    virtual void start() = 0;
    // stops execution, returns `false` if already stopped
    virtual bool stop() = 0;
    virtual ~Processor() {
        stop();
    }
};
```

Логично? — Да!

Правильно? — Нет!

Если вам повезет, то достаточно умный компилятор [сможет](https://godbolt.org/z/PGeob9bn1) сообщить о проблеме. 
В конструкторах и деструкторах в C++ виртуальная диспетчеризация методов не работает (В других языках — например, в C# или Java — наоборот, что доставляет свои проблемы).

Почему так? При конструировании часть объекта-наследника, используемая в переопределенном методе, может быть еще не создана: конструкторы вызываются в порядке от базового класса к производному.
При деструктурировании наоборот — часть объекта-наследника уже уничтожена, и если позволить динамический вызов, можно легко получить use-after-free.

Радуйтесь! Это одно из немногих мест в C++, где вас защитили от неопределенного поведения со временами жизни!

Хорошо. А если так?

```C++
// processor.hpp
class Processor {
public:
    void start();
    // stops execution, returns `false` if already stopped
    bool stop();

    virtual ~Processor();

protected:
    virtual bool stop_impl()  = 0;
    virtual void start_impl() = 0;
};


// processor.cpp
Processor::~Processor() {
    stop();
}

bool Processor::stop() {
    return stop_impl();
}
void Processor::start() {
    start_impl();
}
```

Компиляторы уже [не выдают](https://godbolt.org/z/66fG6cjnd) замечательного предупреждения и подвох заметить стало сложнее. А ведь мы повысили уровень индирекции всего на один! А что будет если код нашего базового класса окажется сложнее?.. Наследование имплементаций — источник многих проблем, прячущихся за невинным желанием переиспользовать код.

Вызов виртуальных функций класса в его конструкторах и деструкторах почти всегда является ошибкой сейчас или в будущем.
Если же это не ошибка и так и задумывалось, то стоит использовать явный статический вызов с указанием имени класса (name qualified call).

```C++
// processor.cpp
Processor::~Processor() {
    Processor::stop();
}
```

Также стоит отметить, что в C++ у pure virtual методов может быть имплементация, к которой можно обращаться. 
Иногда это даже полезно. Таким образом можно потребовать от пользователя обязательно явно принять решение: изменять поведение метода или использовать поведение по умолчанию.

```C++
class Processor {
public:
    virtual void start() = 0;
    // stops execution, returns `false` if already stopped
    virtual bool stop() = 0;

    virtual ~Processor() = default;
};

void Processor::start() {
    std::cout << "unsupported";
}

class MyProcessor : public Processor {
public:
    void start() override {
        // call default implementation
        Processor::start();
    }
};
```

Вернемся опять к нашей остановке при вызове деструктора. Как же с ней быть?

Есть два пути.

Путь первый: потребовать, чтоб реализующий интерфейс обязательно предоставил свою версию деструктора, которая выполнит корректную остановку.

Насильно, с проверкой на этапе компиляции, к этому никого, к сожалению никого не принудишь. Можно [попытаться](https://godbolt.org/z/cWrd7P89r) выразить намерение объявлением деструктора интерфейса чисто виртуальным, но это не поможет, поскольку деструктор, если не указан, всегда генерируется

```C++
class Processor {
public:
    virtual void start() = 0;
    // stops execution, returns `false` if already stopped
    virtual bool stop() = 0;
    virtual ~Processor() = 0;
};

// required!
Processor::~Processor() = default;

class MyProcessor : public Processor {
public:
    void start() override {
    }
    bool stop() override { return false; }
    // ~MyProcessor() override = default; missing destructor does not trigger CE
};


int main() {
    MyProcessor p;
}
```

Путь второй — добавить еще один слой. И пользоваться им во всех публичных API

```C++
class GuardedProcessor {
    std::unique_ptr<Processor> proc;
    // ...
    ~GuardedProcessor() {
        assert(proc != nullptr);
        proc->stop();
    }
};
```


