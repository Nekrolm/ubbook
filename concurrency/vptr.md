# Гонки за vptr

Одна команда инженеров очень любила модель акторов. А еще они очень любили писать на C++ и реализовывать все с нуля, чтобы ни в коем случае не брать внешние зависимости. Поэтому они стали делать свою собственную реализацию акторной модели.

Так сначала у них появился базовый класс

```C++
class Actor {
public:
    virtual ~Actor() = default;
    // мы опустим детали и обхекты
    // для передачи сообщений между акторами
    // важно лишь что был метод run
    virtual void run() = 0; 
};
```

И этого было им достаточно и довольно долго. Пока внезапно не обнаружилось, что надо бы уметь запускать несколько акторов конкурентно и параллельно.

Тогда появился класс-наследник

```C++
class AsyncActor: public Actor {
protected:
    virtual void RunImpl() = 0;

    void run() final {
        actor_thread_ = std::make_unique<std::thread>([this]{ 
            LOG_DEBUG("Started Asynchronously");
            this->RunImpl(); 
        })
    }
private:
    std::unique_ptr<std::thread> actor_thread_;
}
```

И все было также здорово как и раньше. И все наследники `AsyncActor` работали исправно как и было задумано. Да вот только некрасиво как-то это все было! Метод `RunImpl` вместо `run` нужно переопределять. Да и вообще все использования асинхронных акторов следовали паттерну

```C++
SomeAsyncActor actor{...};
actor.run();
```

`run` всегда надо было вызывать явно. А если его забыть — то ведь ничего не запустится же. А если запустить дважды, то произойдет страшное! Неудобно. Да еще и `unique_ptr` глаза мозолит...
А что если сделать по-умному?!

И тогда они переписали `AsyncActor`

```C++
// protected, чтоб не вызвать run больше извне
class AsyncActor: protected Actor {
private:
    // Хопа! Можно написать так красиво и будет компилироваться!
    // используем jthread, чтоб не думать про join
    std::jthread actor_thread_ {
        [actor=this]{
            LOG_DEBUG("Started Asynchronously");
            actor->run();
        }
    };
};
```

И теперь достаточно было просто сделать

```C++
    class SomeAsyncActor : public SomeAsyncActor {
        void run() override {...}
    };

    SomeAsyncActor actor{...};
```
чтобы актор был успешно создан и сразу же запущен, как и требовалось всегда.

Команда была крайне довольна таким изящным решением. Они тестировали его долго и упорно вручную. И все было отлично. На радостях они решили, что можно отключить печать отладочных логов на этапе компиляции. Так что строка `LOG_DEBUG(...)` превратилась в ничто.

Они пересобрали программу. Протестировали несколько раз. Все продолжало работать. Они задеплоили приложение... И оно упало с ошибкой сегментации при старте. Открывши core dump, разработчики увидели: `Pure virtual function called`. Но ведь все же работало?! Они включили логи обратно... И программа снова стала работать корректно.

-----

В нашей истории оказалось целых два race conditions, спрятанных в самом неожиданном месте: в неявном обращении к указателю на таблицу виртуальных методов (vptr)!

В главе про [невиртуальные виртуальные функции](../runtime/virtual_functions.md) мы уже обсуждали, что при вызове виртуального метода из конструктора или деструктора вызывается метод текущего класса, а не переопределенный.

В основных реализациях (Clang и GCC) виртуальных методов такой эффект достигается за счет **переписывания** указателя на таблицу виртуальных функций при входе в конструктор/деструктор и выходе из них. 

Таким образом, если `actor_thread` вызовет `run` раньше чем исполнение дойдет до конструктора `SomeAsyncActor`, будет вызван метод не того класса, который ожидается быть вызванным.
Аналогично с деструктором. Ну а чего вы хотели, случайно вызывая методы на частично уничтоженном или еще не сконструированном объекте?! Это вообще-то неопределенное поведение.

Мы можем легко продемонстрировать эффект:

#### Падение на деструкторе

```C++
class SomeAsyncActor : public AsyncActor {
public:
    SomeAsyncActor() = default;
private:
    void run() override {
        printf("DO DO DO\n");
    }
};

int main() {
    SomeAsyncActor s;
    // Раскомментируйте, чтоб перестало падать
    // std::this_thread::sleep_for(std::chrono::milliseconds(5));
}
```
[Результат](https://gcc.godbolt.org/z/fhbrE135q) c `Clang 18.1 -O3 -std=c++20 -pthread`:
```
pure virtual method called
terminate called without an active exception
Program terminated with signal: SIGSEGV
```

#### Падение на конструкторе
```C++
...
class AsyncActor: protected Actor {
private:
    std::jthread actor_thread_ {
        [actor = this]{
            actor->run();
        }
    };
    // искусственная задержка начала конструирования наследника
    std::string metadata {
        (std::this_thread::sleep_for(std::chrono::milliseconds(1)), "metadata")
    };
};

class SomeAsyncActor : public AsyncActor {
public:
    SomeAsyncActor() = default;
private:
    void run() override {
        printf("DO DO DO\n");
    }
};

int main() {
    SomeAsyncActor s;
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
}
```
[Результат](https://gcc.godbolt.org/z/5vrWzWPGe) c `Clang 18.1 -O3 -std=c++20 -pthread`:
```
pure virtual method called
terminate called without an active exception
Program terminated with signal: SIGSEGV
```
----
Что ж, а на вопрос "при чем же тут отладочные логи?" я предлагаю теперь читателю ответить самостоятельно.
