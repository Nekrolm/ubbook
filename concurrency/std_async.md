# std::async

В каком-то виде поддержка асинхронного программирования появилась в C++11 вместе с моделью памяти и описанием поведения программы в многопоточной среде. И если посмотреть на предложенные возможности глазами разработчиков из 2011 года, то они даже в каком-то смысле могут показаться удачными... Однако в 2024 году мы имеем примеры куда более успешного дизайна как в других языках, так и в самом C++. Да, при всей монструозности, C++26 execution выглядит перспективно и, может быть, им даже научатся пользоваться правильно... Но C++26 еще не скоро станет стандартом по умолчанию в коммерческой разработке, так что вернемся к тому, что уже есть.

С++11 дал нам тип `std::future<T>` — примитив для отложенного (in future) получения результата типа `T`. Результат может вычисляться асинхронно, возможно, в другом потоке, возможно просто отложенно. Кто его знает — зависит от конкретной реализации вычислений, которые вам этот `std::future` выдали в качестве обещания результата. 

И вроде бы все здорово. Но есть нюанс, который разочарует любого, кто поработал с похожими сущностями из других языков — с Promise из JavaScript или с Future из Rust — эргономичность std::future совершенно ужасна. Если вы привыкли к монадическим цепочкам `map`, `and_then`, отвыкайте! std::future не поддерживает их. Вы можете только синхронно ждать результат. Вы хотите ждать результат сразу нескольких вычислений и отреагировать на любой из них? Такой роскоши тоже нет в стандартной библиотеке. Cуществует, конечно, `std::experimental::when_any`, но это очевидно экспериментальная функция.

Тем не менее для не очень серьезных приложений std::future может и сгодится даже сегодня.
Давайте, например, напишем простенький сервер, который будет принимать запросы (мы не ожидаем большой нагрузки) и исполнять их асинхронно. Если вы подумали, что для этого нам сейчас понадобится сделать свой собственный пул потоков... То вы, конечно, правильно подумали, но C++11 предлагает нам готовое решение — `std::async`


```C++
#include <functional>
#include <future>
#include <iostream>
#include <thread>
#include <chrono>

struct Request { std::string data; };
struct Response { std::string data; };

Request accept_request(int i) {
    std::string data = "request " + std::to_string(i);
    return {data};
}

Response process_request(Request r) {
    // imitate IO
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    return { "processed: " + r.data + "\n" };
}

int main() {
    for (int i = 0; i < 5; ++i) {
        auto r = accept_request(i);
        std::async([r = std::move(r)]() mutable {
            auto response = process_request(std::move(r));
            std::cout << response.data;
        });
    }
}
```

Компилируем c `-std=c++14`, запускаем
```
processed: request 0
processed: request 1
processed: request 2
processed: request 3
processed: request 4
```

Вроде всё верно? А сколько времени должен исполняться этот цикл? Измерим
```C++
int main() {
    auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < 5; ++i) {
        auto r = accept_request(i);
        std::async([r = std::move(r)]() mutable {
            auto response = process_request(std::move(r));
            std::cout << response.data;
        });
    }
    auto end = std::chrono::steady_clock::now();
    std::cout << "elapsed: " << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << "\n";
}
```

[500 миллисекунд](https://gcc.godbolt.org/z/Wd5b7csKx). Никакого совпадения. Пять запросов. На каждый 100 миллисекунд. Как заказывали. Да наш код оказался никаким не асинхронным. Обработка выполняется строго последовательно.

Неожиданно? Если вы никогда не видели C++, но при этом написали множество асинхронного кода на Rust или Go, то, думаю, для вас это станет неожиданным результатом. Ведь с go routines и tokio::spawn таких неприятностей не было.

Если мы соберем этот код с `-std=c++17`, то обнаружим предупреждение компилятора!

```
<source>: In function 'int main()':
<source>:25:19: warning: ignoring return value of 'std::future<typename std::__invoke_result<typename std::decay<_Tp>::type, typename std::decay<_Args>::type ...>::type> std::async(_Fn&&, _Args&& ...) [with _Fn = main()::<lambda()>; _Args = {}; typename __invoke_result<typename decay<_Tp>::type, typename decay<_Args>::type ...>::type = void; typename decay<_Tp>::type = main()::<lambda()>]', declared with attribute 'nodiscard' [-Wunused-result]
   25 |         std::async([r = std::move(r)]() mutable {
      |         ~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   26 |             auto response = process_request(std::move(r));
      |             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   27 |             std::cout << response.data;
      |             ~~~~~~~~~~~~~~~~~~~~~~~~~~~
   28 |         });
```

Да-да. И оно непосредственно относится к нашей проблеме. Мы проигнорировали результат `std::async`. А результатом была `std::future<T>`. У проигнорированной std::future естественным образом вызывается дестркутор. Ну-ка, что про него пишут?

Начиная с C++14 есть приписка
```
These actions will not block for the shared state to become ready, except that they may block if all following conditions are satisfied:
The shared state was created by a call to std::async.
The shared state is not yet ready.
The current object was the last reference to the shared state.
```

В этом конкретном специальном случае, когда `std::future<T>` создан с помощью `std::async`, у нее блокирующий деструктор! И это даже в каком-то смысле безопасно: ваша «асинхронная» задача в фоновом потоке не будет оборвана внезапно и безрезультатно, если основной поток программы достигнет конца main функции.

Так что мы должны внести изменения и собрать полученные `std::future` в какой-нибудь контейнер

```C++
int main() {
    std::list<std::future<void>> pending_tasks;
    auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < 5; ++i) {
        auto r = accept_request(i);
        pending_tasks.push_back(std::async([r = std::move(r)]() mutable {
            auto response = process_request(std::move(r));
            std::cout << response.data;
        }));
    }
    // drop all to wait
    { auto _ = std::move(pending_tasks); }
    auto end = std::chrono::steady_clock::now();
    std::cout << "elapsed: " << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count() << "\n";
}
```
И вот теперь у нас получен ожидаемый [результат](https://gcc.godbolt.org/z/qPTaca3eh)
```
processed: request 0
processed: request 3
processed: request 1
processed: request 4
processed: request 2
elapsed: 100
```

Победа? Не совсем. Мы все еще продолжаем неправильно использовать `std::async`! При вызове его в такой форме поведение **не специфицировано**. Есть два варианта:
- Переданная функция каким-то образом начнет исполнятся асинхронно (в фоновом потоке или в пуле потоком ­— это тоже не специфировано). И такое поведение по умолчанию во всех современных версиях GCC, Clang и MSVC.
- Переданная функция не будет выполянтся до тех пор, пока вы не вызовете `wait` у возвращенной `std::future`. И такое поведение долгое время было со старыми версиями компиляторов. Например, [GCC 5.4](https://gcc.godbolt.org/z/nY6Kv4Gdz)

Какое именно поведение вы хотите можно и нужно контроллировать с помощью вызова перегрузки `std::async` с дополнительным первым параметром типа `std::launch`:
- `std::launch::async` — если вы действительно хотите асинхронное исполнение
- `std::launch::deferred` — если нужно отложить до точки вызова `wait`

Пожалуй это одна из изысканных шуток стандартной библиотеки C++: `std::async(f)` по умолчанию не async, a `std::async(std::launch::async | std::launch::deferred, f)`.

Так что правильный код будет выглядеть так

```C++
int main() {
    std::list<std::future<void>> pending_tasks;
    ...
    for (int i = 0; i < 5; ++i) {
        auto r = accept_request(i);
        pending_tasks.push_back(std::async(std::launch::async, 
            [r = std::move(r)]() mutable {
                auto response = process_request(std::move(r));
                std::cout << response.data;
            }));
    }
    // drop all to wait
    { auto _ = std::move(pending_tasks); }
    ...
}
```

Теперь точно победа? Да, но только в таком совсем простом примере... Ведь в реальности же ваш сервер будет принимать не фиксированное количество запросов. А значит, список из `std::future` будет расти... и расти... и расти... И вам все-таки придется
взять и самостоятельно запустить фоновый поток, который будет их потихоньку из списка выбрасывать. Считайте это домашним заданием. После его выполнения можете ответить на вопрос:
- Стоит ли использовать `std::async`, чтоб не думать о собственном пуле потоков?


### Полезные ссылки
1. https://en.cppreference.com/w/cpp/thread/async
2. https://stackoverflow.com/questions/46102206/how-does-a-c-compiler-choose-between-deferred-and-async-execution-for-stdasy
