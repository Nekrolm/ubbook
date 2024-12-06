# Корутины: время жизни и смерти

`async/await` синтаксис плотно вошел в жизнь современных разработчиков: от фронтенда до бэкенда и низкоуровневой системщины. В 2007 году он появился в `F#` и за следующие годы разбежался по множеству языков: C# (2012), Python (2015), JavaScript (2017), Kotlin (2018), Rust (2019), Zig (2020, но в 2024 [убрали](https://github.com/ziglang/zig/wiki/FAQ#what-is-the-status-of-async-in-zig) из-за проблем реализации в self-hosted компиляторе).

Несмотря на все его неоднозначности ([знаменитая проблема "цветных" функций](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)), возможность писать простой линейный код вместо классического callback-спагетти для сложных асинхронных задач — полезна и сокращает усилия на прототипирование.

Чтобы не отставать, С++20 также добавил долгожданную поддержку: вместо `async` функций у нас относительно явные и более общие типы - корутины. И для них тоже есть `await`... простите, `co_await`! A также `co_return` и `co_yield` — в С++ одним махом решилт как проблемы асинхронных-функций, так функций-генераторов... Или создали проблемы с ними...

К сожалению, поддержка есть, а вот корутин в cтандартной библиотеке нет! Если хотите, реализуйте свои. Но я пожалуй, возьму корутины из boost::asio, чтобы продемонстрировать следующий восхитительный пример.

```C++
#include <iostream>
#include <concepts>
#include <vector>
#include <string>
#include <ranges>
#include <chrono>

#include <boost/asio/co_spawn.hpp>
#include <boost/asio/detached.hpp>
#include <boost/asio/io_context.hpp>
#include <boost/asio/steady_timer.hpp>

using namespace boost::asio;
namespace this_coro = boost::asio::this_coro;

using namespace std::literals::string_literals;
using namespace std::chrono;

using Request = std::string;

struct MetricEmitter {
    std::string metric_class;
    void emit(std::chrono::milliseconds elapsed) const {
        std::cout << metric_class << " " << elapsed << "\n";
    }
};

// Демонстрационная корутина для иммитации ввода-выводы
awaitable<void> some_io(int delay) {
    steady_timer timer(co_await this_coro::executor);
    timer.expires_after(milliseconds(delay));
    co_await timer.async_wait(use_awaitable);
    co_return;
}


awaitable<void> handle_request(const Request& r) {
    co_await some_io(15);
    std::cout << "Hello " << r << "\n";
    co_return;
}

template <std::ranges::range Requests>
awaitable<void> process_requests_batch(Requests&& reqs) 
requires std::convertible_to<std::ranges::range_value_t<Requests>, Request>  {
    auto executor = co_await this_coro::executor;
    // добавляем к обработке запроса метрики времени выполнения
    auto handle_with_metrics = [metrics = MetricEmitter { "batch_processor"} ](auto&& request) -> awaitable<void> {
        auto start = steady_clock::now();
        co_await handle_request(std::move(request));
        auto finish = steady_clock::now();
        metrics.emit(duration_cast<milliseconds>(finish - start));
    };
    for (auto&& r: std::move(reqs)) {
        // запускаем конкурентное исполнение для каждого реквеста.
        co_spawn(executor, handle_with_metrics(std::move(r)), detached);
    }
    co_return;
}

awaitable<std::vector<Request>> accept_requests_batch() {
    co_return std::vector{ "Adam"s, "Helen"s, "Bob"s };
}

awaitable<void> run() {
   co_await process_requests_batch(co_await accept_requests_batch());
   co_await some_io(100);
}

int main()
{
    // Запускаем наши корутины в однопоточном контексте исполнения
    boost::asio::io_context io_context(1);
    co_spawn(io_context, run(), detached);
    io_context.run();
}
```

Вы могли бы предположить, что этот код успешно напечатает три раза
```
Hello <имя>
batch_processor <время обработки>
```
в каком-то порядке.

Но на самом деле он с очень большой вероятностью упадет с ошибкой сегментации.

Посмотрим, какое приветственное сообщение [покажет](https://godbolt.org/z/KrY5avbqe) нам address sanitizer

`gcc -std=c++23  -O0 -fsanitize=address`


```
AddressSanitizer:DEADLYSIGNAL
=================================================================
==1==ERROR: AddressSanitizer: SEGV on unknown address 0x00000000001b (pc 0x7a9f129aedf4 bp 0x7a9f12a1b780 sp 0x7fff9f00a228 T0)
==1==The signal is caused by a READ memory access.
==1==Hint: address points to the zero page.
    #0 0x7a9f129aedf4  (/lib/x86_64-linux-gnu/libc.so.6+0x1aedf4) (BuildId: 490fef8403240c91833978d494d39e537409b92e)
    #1 0x7a9f1288b664 in _IO_file_xsputn (/lib/x86_64-linux-gnu/libc.so.6+0x8b664) (BuildId: 490fef8403240c91833978d494d39e537409b92e)
    #2 0x7a9f1287ffd6 in fwrite (/lib/x86_64-linux-gnu/libc.so.6+0x7ffd6) (BuildId: 490fef8403240c91833978d494d39e537409b92e)
    #3 0x7a9f12e900ab  (/opt/compiler-explorer/gcc-14.2.0/lib64/libasan.so.8+0x820ab) (BuildId: e522418529ce977df366519db3d02a8fbdfe4494)
    #4 0x7a9f12ce8d1c in std::basic_ostream<char, std::char_traits<char> >& std::__ostream_insert<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*, long) (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0x14cd1c) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #5 0x407e03 in handle_request /app/example.cpp:39
    #6 0x40f697 in std::__n4861::coroutine_handle<void>::resume() const /opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/coroutine:137
    #7 0x449403 in boost::asio::detail::awaitable_frame_base<boost::asio::any_io_executor>::resume() /app/boost/include/boost/asio/impl/awaitable.hpp:501
    #8 0x445ba4 in boost::asio::detail::awaitable_thread<boost::asio::any_io_executor>::pump() /app/boost/include/boost/asio/impl/awaitable.hpp:770
    #9 0x454bc7 in boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>::operator()(boost::system::error_code const&) /app/boost/include/boost/asio/impl/use_awaitable.hpp:93
    #10 0x4517e6 in boost::asio::detail::binder1<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::system::error_code>::operator()() /app/boost/include/boost/asio/detail/bind_handler.hpp:115
    #11 0x44f337 in void boost::asio::detail::handler_work<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::asio::any_io_executor, void>::complete<boost::asio::detail::binder1<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::system::error_code> >(boost::asio::detail::binder1<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::system::error_code>&, boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>&) /app/boost/include/boost/asio/detail/handler_work.hpp:433
    #12 0x44ccfe in boost::asio::detail::wait_handler<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::asio::any_io_executor>::do_complete(void*, boost::asio::detail::scheduler_operation*, boost::system::error_code const&, unsigned long) /app/boost/include/boost/asio/detail/wait_handler.hpp:76
    #13 0x41747c in boost::asio::detail::scheduler_operation::complete(void*, boost::system::error_code const&, unsigned long) /app/boost/include/boost/asio/detail/scheduler_operation.hpp:40
    #14 0x41e925 in boost::asio::detail::scheduler::do_run_one(boost::asio::detail::conditionally_enabled_mutex::scoped_lock&, boost::asio::detail::scheduler_thread_info&, boost::system::error_code const&) /app/boost/include/boost/asio/detail/impl/scheduler.ipp:493
    #15 0x41dcfb in boost::asio::detail::scheduler::run(boost::system::error_code&) /app/boost/include/boost/asio/detail/impl/scheduler.ipp:210
    #16 0x41f27a in boost::asio::io_context::run() /app/boost/include/boost/asio/impl/io_context.ipp:64
    #17 0x4099bf in main /app/example.cpp:75
```
Похоже что ссылка:
```C++
awaitable<void> handle_request(const Request& r)
```
Немножечко испортилась... Но что же пошло не так?!

------

Корутины — очень сложные объекты, которые обманчиво просты в использовании из-за синтаксического сахара. В этом же весь смысл! Поддержка `async/await` на уровне языка и компиляторов делает простым то, что всегда было делать сложно вручную... Так в происходит в высокоуровневых и безопасных языках с автоматическим управлением памятью: Python, C#, JavaScript, Kotlin.
Но не в C++. И не в Rust. (И не в Zig).

В примере выше есть как минимуи **три** точки отказа, содержащих ошибки. Можете подумать об этом, пока мы будем разворачивать проблемы корутин С++.

### Что такое корутина?

Здесь можно поупражняться в терминологии, ведь определений много разных. Есть довольно абстрактное и высокоуровневое:
*корутина — это функция, которая может приостановить свою работу, и которую можно возобновить позже*.

Но есть один нюанс, ведущий к частому недопониманию: "магическим" свойством на самом деле обладает **не функция**, а **объект**, который из функции **возвращается**.

JavaScript разработчики знают (я надеюсь), что

```JavaScript
async function myFunction() {
  return "Hello";
}

// то же самое что и

function myFunction() {
  return Promise.resolve("Hello");
}
```

Аналогично в Rust

```Rust 

async fn my_foo() -> String
{
    "Hello".to_string()
}

// то же самое что и 
fn my_foo() -> impl Future<Output = String> {
    // создает анонимный объект Future
    async {
        "Hello".to_string()
    }
}
```

В C++ же нет специального синтаксиса для объявления функции корутинами. Вместо этого нужно явно указать тип возвращаемого значения (например, `awaitable`). И требования к этому типу специфичны и не сразу понятны.
1. `awaitable` должен удовлетворять концепту `std::coroutine_handle_traits`. То есть у него должен быть ассоциированный тип: `promise = typename awaitable::promise_type`
2. Тип `promise` должен удовлетворять [документации](https://eel.is/c++draft/dcl.fct.def.coroutine#def:coroutine,promise_type) концепта `Promise` столь сложному для описания, что про корутины в C++ приходится писать отдельные книжки. Но если кратко: promise контролирует поведение операций `co_await`, `co_yield` и `co_return`.
3. Инстанциация `handle = std::coroutine_handle<promise>` должна быть успешной
4. Должна быть возможность сконструировать `awaitable` c помощью `promise.get_return_object()`

И вот только тогда внутри тела функции, возвращающей `awaitable`, можно будет (и часто нужно будет) использовать синтаксический сахар `co_await`, `co_yield`, `co_return`
```C++
awaitable<std::string> myFunction() {
    co_return "Hello";
}
```

Который рассахаривается в нечто подобное (это приблизительный не-код)

```C++
awaitable<std::string> myFunction() {
    using Promise = awaitable::promise_type;
    using Handle = std::coroutine_handle<Promise>;
    Promise p;
    auto state = new ImplicitlyGeneratedStateMachine<Handle>(p);
    // state = _ 0;
    // ... 
    //{  
    //  switch(state) {
    //     case_0: { state = _1; p.initial_suspend(); }
    //     case _1: { p.yield_value("Hello"); }
    //   }
    // }
    return p.get_return_object();
}
```

Если же ни одно из `co_*` ключевых слов в теле функции не присутствует, то никаких магических преобразований не происходит! И без злого умысла, кажется, такое придумать было нельзя! Смотрите-ка

```C++
awaitable<void> process_request(const std::string& r) { 
    co_await some_io(1);
    std::cout << r; 
}

awaitable<void> send_dummy_request() {
    return process_request("hello");
}

int main(){
    boost::asio::io_context io_context(1);
    co_spawn(io_context, send_dummy_request(), detached);
    io_context.run();
}
```
Запускаем. Проверяем. Работает? Успешно ничего [не печатает](https://godbolt.org/z/deEss1dxM)...
Странно... Давайте-ка уберем `some_io()`.

```C++
awaitable<void> process_request(const std::string& r) { 
    std::cout << r; 
}
```

Запускаем. Проверяем. Получаем что? [Правильно](https://godbolt.org/z/oxx5bs3sb)

```C++
<source>: In function 'boost::asio::awaitable<void> process_request(const std::string&)':
<source>:19:73: warning: no return statement in function returning non-void [-Wreturn-type]
   19 | awaitable<void> process_request(const std::string& r) { std::cout << r; }
      |                                                                         ^
ASM generation compiler returned: 0
<source>: In function 'boost::asio::awaitable<void> process_request(const std::string&)':
<source>:19:73: warning: no return statement in function returning non-void [-Wreturn-type]
   19 | awaitable<void> process_request(const std::string& r) { std::cout << r; }
      |                                                                         ^
Execution build compiler returned: 0
Program returned: 132
Program terminated with signal: SIGILL
```
Ну разумеется. Ну конечно. Ведь без волшебных ключевых слов магии нет и наша `process_request` функция ничего не возвращает. А это же [неопределенное поведение](../syntax/missing_return.md)!

Добавим `co_return`.

```C++
awaitable<void> process_request(const std::string& r) { 
    std::cout << r; 
    co_return;
}
```
Снова [пусто](https://godbolt.org/z/P7nEWoErv)...

Подключаем санитайзер!

```
=================================================================
==1==ERROR: AddressSanitizer: stack-use-after-return on address 0x758796100150 at pc 0x7587985d01e6 bp 0x7ffda95e0290 sp 0x7ffda95dfa50
READ of size 5 at 0x758796100150 thread T0
    #0 0x7587985d01e5  (/opt/compiler-explorer/gcc-14.2.0/lib64/libasan.so.8+0x821e5) (BuildId: e522418529ce977df366519db3d02a8fbdfe4494)
    #1 0x758798428d1c in std::basic_ostream<char, std::char_traits<char> >& std::__ostream_insert<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*, long) (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0x14cd1c) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #2 0x405c47 in process_request /app/example.cpp:21
    #3 0x4082f5 in std::__n4861::coroutine_handle<void>::resume() const /opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/coroutine:137
    #4 0x427e8b in boost::asio::detail::awaitable_frame_base<boost::asio::any_io_executor>::resume() /app/boost/include/boost/asio/impl/awaitable.hpp:501
    #5 0x425e2c in boost::asio::detail::awaitable_thread<boost::asio::any_io_executor>::pump() /app/boost/include/boost/asio/impl/awaitable.hpp:770
```

Ой, ссылка `const std::string& r` умерла похоже. Какое несчастье. Почему?

А вот же: 
```C++
awaitable<void> send_dummy_request() {
    return process_request("hello");
}
```

Тут то же нет никаких магических `co_*` ключевых слов. А значит мы просто вызываем функцию `process_request`, она возвращает объект-корутину, который...

Самое время уточнить что происходит с аргументами функций, использующих `co_*`: они **неявно** копируются **внутрь** неявно создаваемого объекта. Значение копируется как значение. Ссылка — как ссылка.

```C++
awaitable<std::string> myFunction(int arg1, const std::string& arg2) {
    using Promise = awaitable::promise_type;
    using Handle = std::coroutine_handle<Promise>;
    Promise p;
    auto state = new ImplicitlyGeneratedStateMachine<Handle>(p);
    //  state = _ 0;
    //  int __arg1 { arg1 };
    //  const std::string& __arg2 { arg2 };
    //{  
    //  switch(state) {
    //     case_0: { state = _1; p.initial_suspend(); }
    //     case _1: { p.yield_value("Hello"); }
    //   }
    // }
    return p.get_return_object();
}
```

То есть

```C++
awaitable<void> process_request(const std::string& r) {...}

awaitable<void> send_dummy_request() {
    // неявно конструируется локальный временный std::string, ссылка на него передается в
    // функцию process_request. Где ссылка дальше копируется внутрь стейт-машины
    return process_request("hello");
    // возвращаем стейт-машину, а локальный временный объект умирает. 
    // Use-after-free при попытке получить результат стейт-машины
}
```

Нужны волшебные слова... Между тем, вы еще не чувствуете насколько все может стать плохо в контексте шаблонов? Нет? Когда может быть совершенно не ясно, корутину нам передали или нет? Нет? Ну ничего страшного. Возьмите это в качестве упражнения на дом — написать `std::invoke`, поддерживающий корутины. А мы пока продолжим добавлять волшебные слова

```C++
awaitable<void> send_dummy_request() {
    co_return process_request("hello");
}
```

```
<source>: In function 'boost::asio::awaitable<void> send_dummy_request()':
<source>:26:5: error: no member named 'return_value' in 'std::__n4861::coroutine_traits<boost::asio::awaitable<void> >::promise_type' {aka 'boost::asio::detail::awaitable_frame<void, boost::asio::any_io_executor>'}
   26 |     co_return process_request("hello");
      |     ^~~~~~~~~
Compiler returned: 1
```

Ну естественно. Ведь `process_request` возвращает корутину... 
Давайте, шаг за шагом, напишем, что же на самом деле должно происходить. Ведь мы же хотим понять и разобраться...

```C++
awaitable<void> send_dummy_request() {
    auto task = process_request("hello"); // функция вернула корутину-стейт-машину
    auto result = co_await task; // нужно дождаться завершения
    co_return result; // и вернуть результат
}
```
Ой, небольшая заминочка...

```
<source>: In function 'boost::asio::awaitable<void> send_dummy_request()':
<source>:27:28: error: use of deleted function 'boost::asio::awaitable<T, Executor>::awaitable(const boost::asio::awaitable<T, Executor>&) [with T = void; Executor = boost::asio::any_io_executor]'
   27 |     auto result = co_await task;
      |                            ^~~~
In file included from /app/boost/include/boost/asio/co_spawn.hpp:22,
                 from <source>:8:
/app/boost/include/boost/asio/awaitable.hpp:123:3: note: declared here
  123 |   awaitable(const awaitable&) = delete;
      |   ^~~~~~~~~
```
Это особенность boost asio, для большей безопасности он требует применять `co_await` только к rvalue.
Небольшое исправление.
```C++
auto result = co_await std::move(task);
```
И мы получаем...

```
<source>:28:10: error: deduced type 'void' for 'result' is incomplete
   28 |     auto result = co_await std::move(task);
      |          ^~~~~~
```

Вы все еще не почувствовали, насколько все может стать плохо с шаблонами? Ничего страшного, помним, что у С++ всегда были проблемы с таким замечательным типом `void`.
Просто объединим `co_return` и `co_await` в одну строчку

```C++
awaitable<void> send_dummy_request() {
    auto task = process_request("hello"); // функция вернула корутину-стейт-машину
    // нужно дождаться завершения
    co_return co_await std::move(task); // и вернуть результат
}
```

Компилируется [и...](https://godbolt.org/z/16sTer15x)
```
=================================================================
==1==ERROR: AddressSanitizer: stack-use-after-return on address 0x76cc8d801070 at pc 0x76cc8fa541e6 bp 0x7ffed68d7510 sp 0x7ffed68d6cd0
READ of size 5 at 0x76cc8d801070 thread T0
    #0 0x76cc8fa541e5  (/opt/compiler-explorer/gcc-14.2.0/lib64/libasan.so.8+0x821e5) (BuildId: e522418529ce977df366519db3d02a8fbdfe4494)
    #1 0x76cc8f8acd1c in std::basic_ostream<char, std::char_traits<char> >& std::__ostream_insert<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*, long) (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0x14cd1c) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #2 0x405c47 in process_request /app/example.cpp:20
    #3 0x408cc7 in std::__n4861::coroutine_handle<void>::resume() const /opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/coroutine:137
    #4 0x428cef in boost::asio::detail::awaitable_frame_base<boost::asio::any_io_executor>::resume() /app/boost/include/boost/asio/impl/awaitable.hpp:501
    #5 0x426e00 in boost::asio::detail::awaitable_thread<boost::asio::any_io_executor>::pump() /app/boost/include/boost/asio/impl/awaitable.hpp:770
    #6 0x432170 in boost::asio::detail::awaitable_async_op_handler<void (), boost::asio::any_io_executor>::operator()() /app/boost/include/boost/asio/impl/awaitable.hpp:804
    #7 0x431a95 in boost::asio::detail::binder0<boost::asio::detail::awaitable_async_op_handler<void (), boost::asio::any_io_executor> >::operator()() /app/boost/include/boost/asio/detail/bind_handler.hpp:56
    #8 0x431f04 in void boost::asio::detail::executor_function::complete<boost::asio::detail::binder0<boost::asio::detail::awaitable_async_op_handler<void (), boost::asio::any_io_executor> >, std::allocator<void> >(boost::asio::detail::executor_function::impl_base*, bool) /app/boost/include/boost/asio/detail/executor_function.hpp:113
    #9 0x40a5ac in boost::asio::detail::executor_function::operator()() /app/boost/include/boost/asio/detail/executor_function.hpp:61
    #10 0x42a66d in boost::asio::detail::executor_op<boost::asio::detail::executor_function, std::allocator<void>, boost::asio::detail::scheduler_operation>::do_complete(void*, boost::asio::detail::scheduler_operation*, boost::system::error_code const&, unsigned long) /app/boost/include/boost/asio/detail/executor_op.hpp:70
    #11 0x410754 in boost::asio::detail::scheduler_operation::complete(void*, boost::system::error_code const&, unsigned long) /app/boost/include/boost/asio/detail/scheduler_operation.hpp:40
    #12 0x41728b in boost::asio::detail::scheduler::do_run_one(boost::asio::detail::conditionally_enabled_mutex::scoped_lock&, boost::asio::detail::scheduler_thread_info&, boost::system::error_code const&) /app/boost/include/boost/asio/detail/impl/scheduler.ipp:493
    #13 0x41680d in boost::asio::detail::scheduler::run(boost::system::error_code&) /app/boost/include/boost/asio/detail/impl/scheduler.ipp:210
    #14 0x417be0 in boost::asio::io_context::run() /app/boost/include/boost/asio/impl/io_context.ipp:64
    #15 0x406b67 in main /app/example.cpp:36
```

Та же самая проблема. Строка умерла. По той же самой причине.
А что если мы теперь соединим все в одну строку кода?

```C++
awaitable<void> send_dummy_request() {
    co_return co_await process_request("hello");
}
```
А вот теперь все наконец-то правильно. И [печатает](https://godbolt.org/z/cjG3dTjnr) заветное слово `"hello"`;

Здорово, неправда ли, как мы прошли путь от неправильного
```C++
awaitable<void> send_dummy_request() {
    return process_request("hello");
}
```
к правильному 
```C++
awaitable<void> send_dummy_request() {
     co_return co_await process_request("hello");
}
```

Разве могла бы такая красота получиться, если бы C++ решил все-таки требовать маркировку `[co_]async` у объявления функций? Тогда бы у нас было скучно (прям как в Rust):
```C++
[[co_async]] awaitable<void> send_dummy_request() {
    return process_request("hello"); // Compilation error! Type mismatch / co_return should be used
} 
```

Но это все была синтаксическая забава, в процессе которой мы поймали ошибку: const ссылки, rvalue ссылки и неявное создание временных объектов. Ссылки неявно захватываются стейт-машиной, а неявные временные объекты неявно умирают.
Создавайте временные переменные явно. Контролируйте их время жизни. Избегайте ссылочных параметров у корутин при возможности. Рецепт простой. Вернемся к нашему самому первому примеру и исправим ошибки и потенциальные проблемы со ссылками.

```C++
// Принимаем теперь все параметры для корутин по значению. 

awaitable<void> handle_request(Request r) {
    co_await some_io(15);
    std::cout << "Hello " << r << "\n";
    co_return;
}

template <std::ranges::range Requests>
awaitable<void> process_requests_batch(Requests reqs) 
requires std::convertible_to<std::ranges::range_value_t<Requests>, Request>  {
    auto executor = co_await this_coro::executor;
    // добавляем к обработке запроса метрики времени выполнения
    auto handle_with_metrics = [metrics = MetricEmitter { "batch_processor"} ](auto request) -> awaitable<void> {
        auto start = steady_clock::now();
        co_await handle_request(std::move(request));
        auto finish = steady_clock::now();
        metrics.emit(duration_cast<milliseconds>(finish - start));
    };
    for (auto&& r: std::move(reqs)) {
        // запускаем конкурентное исполнение для каждого реквеста.
        co_spawn(executor, handle_with_metrics(std::move(r)), detached);
    }
    co_return;
}

awaitable<std::vector<Request>> accept_requests_batch() {
    co_return std::vector{ "Adam"s, "Helen"s, "Bob"s };
}

awaitable<void> run() {
   co_await process_requests_batch(co_await accept_requests_batch());
   co_await some_io(100);
}
```

Компилируем. Запускаем. И получаем... Правильно, [новую ошибку сегментации](https://godbolt.org/z/Y3GnYePvf)!

```
=================================================================
==1==ERROR: AddressSanitizer: heap-use-after-free on address 0x511000000228 at pc 0x7c812eca71e6 bp 0x7ffded723390 sp 0x7ffded722b50
READ of size 15 at 0x511000000228 thread T0
    #0 0x7c812eca71e5  (/opt/compiler-explorer/gcc-14.2.0/lib64/libasan.so.8+0x821e5) (BuildId: e522418529ce977df366519db3d02a8fbdfe4494)
    #1 0x7c812eaffd1c in std::basic_ostream<char, std::char_traits<char> >& std::__ostream_insert<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*, long) (/opt/compiler-explorer/gcc-14.2.0/lib64/libstdc++.so.6+0x14cd1c) (BuildId: 998334304023149e8c44e633d4a2c69800a2eb79)
    #2 0x41f592 in MetricEmitter::emit(std::chrono::duration<long, std::ratio<1l, 1000l> >) const /app/example.cpp:24
    #3 0x40ab8e in operator() /app/example.cpp:52
    #4 0x40f771 in std::__n4861::coroutine_handle<void>::resume() const /opt/compiler-explorer/gcc-14.2.0/include/c++/14.2.0/coroutine:137
    #5 0x4494e7 in boost::asio::detail::awaitable_frame_base<boost::asio::any_io_executor>::resume() /app/boost/include/boost/asio/impl/awaitable.hpp:501
    #6 0x445c86 in boost::asio::detail::awaitable_thread<boost::asio::any_io_executor>::pump() /app/boost/include/boost/asio/impl/awaitable.hpp:770
    #7 0x454cab in boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>::operator()(boost::system::error_code const&) /app/boost/include/boost/asio/impl/use_awaitable.hpp:93
    #8 0x4518ca in boost::asio::detail::binder1<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::system::error_code>::operator()() /app/boost/include/boost/asio/detail/bind_handler.hpp:115
    #9 0x44f41b in void boost::asio::detail::handler_work<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::asio::any_io_executor, void>::complete<boost::asio::detail::binder1<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::system::error_code> >(boost::asio::detail::binder1<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::system::error_code>&, boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>&) /app/boost/include/boost/asio/detail/handler_work.hpp:433
    #10 0x44cde2 in boost::asio::detail::wait_handler<boost::asio::detail::awaitable_handler<boost::asio::any_io_executor, boost::system::error_code>, boost::asio::any_io_executor>::do_complete(void*, boost::asio::detail::scheduler_operation*, boost::system::error_code const&, unsigned long) /app/boost/include/boost/asio/detail/wait_handler.hpp:76
    #11 0x417556 in boost::asio::detail::scheduler_operation::complete(void*, boost::system::error_code const&, unsigned long) /app/boost/include/boost/asio/detail/scheduler_operation.hpp:40
    #12 0x41e9ff in boost::asio::detail::scheduler::do_run_one(boost::asio::detail::conditionally_enabled_mutex::scoped_lock&, boost::asio::detail::scheduler_thread_info&, boost::system::error_code const&) /app/boost/include/boost/asio/detail/impl/scheduler.ipp:493
    #13 0x41ddd5 in boost::asio::detail::scheduler::run(boost::system::error_code&) /app/boost/include/boost/asio/detail/impl/scheduler.ipp:210
    #14 0x41f354 in boost::asio::io_context::run() /app/boost/include/boost/asio/impl/io_context.ipp:64
    #15 0x4099ae in main /app/example.cpp:75
```

Судя по трейсу, теперь у нас умерла другая строка... Та, что была сохранена в `MetricEmitter`

```C++
template <std::ranges::range Requests>
awaitable<void> process_requests_batch(Requests reqs) 
requires std::convertible_to<std::ranges::range_value_t<Requests>, Request>  {
    auto executor = co_await this_coro::executor;
    // добавляем к обработке запроса метрики времени выполнения
    auto handle_with_metrics = [metrics = MetricEmitter { "batch_processor"} ](auto request) -> awaitable<void> {
        auto start = steady_clock::now();
        co_await handle_request(std::move(request));
        auto finish = steady_clock::now();
        metrics.emit(duration_cast<milliseconds>(finish - start));
    };
    for (auto&& r: std::move(reqs)) {
        // запускаем конкурентное исполнение для каждого реквеста.
        co_spawn(executor, handle_with_metrics(std::move(r)), detached);
    }
    co_return;
}
```

Если вы еще не догадались, позвольте напомнить о еще кое-чем неявном... А также о том что корутинами могут быть и методы классов

```C++
struct MetricEmitter {
    std::string metric_class;
    void emit(std::chrono::milliseconds elapsed) const {
        std::cout << metric_class << " " << elapsed << "\n";
    }

    awaitable<void> wrap_request(Request r) const {
        auto start = steady_clock::now();
        co_await handle_request(std::move(r));
        auto finish = steady_clock::now();
        // Корутина также неявно захватывает указатель this!
        emit(duration_cast<milliseconds>(finish - start));
    }
};

// так что, наверное, очевидно, что
auto task = MetricEmitter{"batch_process"}.wrap_request(request);
co_await task; // MetricEmitter умер. Будет use-after-free 
```

У нас в примере кое-что похожее, но другое

```C++
// handle_with_metric это анонимная структура с определенным operator()
auto handle_with_metrics = [metrics = MetricEmitter { "batch_processor"} ](auto request) -> awaitable<void> {
    auto start = steady_clock::now();
    co_await handle_request(std::move(request));
    auto finish = steady_clock::now();
    // корутина неявно захватывает this... A this в этом случае -- указатель на 
    // лямбда-функцию! 
    metrics.emit(duration_cast<milliseconds>(finish - start));
};
// если лямбда-функция умрет раньше чем завершится исполнение корутины, будет use-after-free
```

Смотрим как мы ее вызываем
```C++
    for (auto&& r: std::move(reqs)) {
        // запускаем конкурентное исполнение для каждого реквеста...
        // В ФОНЕ!!! результат вызова handle_with_metrics -- корутина -- сохраняется куда-то
        // внутри функции boost::asio::co_spawn
        co_spawn(executor, handle_with_metrics(std::move(r)), detached);
        // и будет обработана позже
    }
    co_return; // а вот тут наша лямбда и умрет.
```

Под такие неприятные случаи у `co_spawn` есть перегрузка, принимающая напрямую функцию, а не `awaitable<T>`. Но у функции тогда не должно быть аргументов.

Ошибку можно исправить разными способами, следуя рекомендации: все параметры корутины нужно передать явно и переместить внутрь ее тела. В C++23, для методов классов, с этим может помочь `deduced this`

```C++
struct MetricEmitter {
    std::string metric_class;
    void emit(std::chrono::milliseconds elapsed) const {
        std::cout << metric_class << " " << elapsed << "\n";
    }

    awaitable<void> wrap_request(this auto self, Request r) {
        // self скопирован по значению!
        auto start = steady_clock::now();
        co_await handle_request(std::move(r));
        auto finish = steady_clock::now();
        // Корутина также неявно захватывает указатель this!
        self.emit(duration_cast<milliseconds>(finish - start));
    }
};

```
А из statefull лямбд возвращать корутины не рекомендуется. Убирайте список захвата!


```C++
template <std::ranges::range Requests>
awaitable<void> process_requests_batch(Requests reqs) 
requires std::convertible_to<std::ranges::range_value_t<Requests>, Request>  {
    auto executor = co_await this_coro::executor;
    // добавляем к обработке запроса метрики времени выполнения
    auto handle_with_metrics = [](auto request) -> awaitable<void> {
        auto metrics = MetricEmitter { "batch_processor"}; 
        auto start = steady_clock::now();
        co_await handle_request(std::move(request));
        auto finish = steady_clock::now();
        metrics.emit(duration_cast<milliseconds>(finish - start));
    };
    for (auto&& r: std::move(reqs)) {
        // запускаем конкурентное исполнение для каждого реквеста.
        co_spawn(executor, handle_with_metrics(std::move(r)), detached);
    }
    co_return;
}
```

Вот теперь все [работает](https://godbolt.org/z/oTW466Y6Y).
```
Hello Adam
batch_processor 15ms
Hello Helen
batch_processor 15ms
Hello Bob
batch_processor 15ms
```

------
Отслеживать время жизни ссылок в асинхронном коде вручную крайне тяжело. Автоматика, как в Rust, делает это намного лучше, но при этом может выдавать совершенно непонятные репорты, в которых можно разобраться только если знаешь, что именно могло пойти не так — за это async в Rust и не любят и критикуют. А в качестве самого простого и продуктивного решения, чтоб убложить borrow checker, выбирается копирование всего подряд (`.clone()`, везде `.clone()`)

С++ отдает полный контроль вам с невероятной кучей неявных захватов ссылок! Делайте с ними что хотите и как хотите. Скомпилируется без проблем и проверок. Вы можете приложить ментальные усилия, отследить все ссылки и убедиться что объекты не умрут не вовремя. Либо можно отчаяться, прочитать гайдлайны и передавать все и всегда by value. Копируя и перемещая. Никаких ссылок.

Проблема со ссылками это лишь начало. Все становится намного хуже, если мы еще и подключим многопоточное выполнение корутин. 
- Только внимательность, опыт и, может быть, [статический анализатор](https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/no-suspend-with-lock.html), помогут вам избежать случайной попытки переместить `std::unique_lock<std::mutex>` между потоками, если вы его по какой-то причине продержите через `co_await`.
- Разумеется еще больше возможности для race conditions. Особенно если забыть что-то скопировать внутрь тела корутины.

Ах да, совсем забыл: в зависимости от реализации, корутины могут быть ленивыми или не очень ленивыми (смотри `promise::initial_suspend()`). Boost::asio::awaitable  — ленивые. И потому мы сразу получали прекрасные use-after-free. Для корутин, у которых `promise::initial_suspend()` возвращает `suspend_never`, код вида
```C++
awaitable<void> process_request(const std::string& r) { 
    std::cout << r; 
    co_await something();
    /* r не используется */
    co_return;
}
```
может продолжать успешно работать и долгое время не вызывать проблем

----

Корутины C++ — гибки, мощны и совершенно небезопасны. Надеюсь, вы настроили сборку и тесты с санитайзерами прежде чем решили ими воспользоваться. 

По состоянию на 2024 год статические анализаторы C++ **частично подсвечивают** подобные проблемы. 
- Clang-Tidy имеет агрессивную проверку на [использование ссылок в корутинах](https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/avoid-reference-coroutine-parameters.html)
- Также может быть настроен на подсвечивание [statefull лябмд](https://clang.llvm.org/extra/clang-tidy/checks/cppcoreguidelines/avoid-capturing-lambda-coroutines.html)
- Но они не скажут, что вам нужно использовать `co_return co_await foo()` вместо `return foo()`. 

Но в общем случае это не ошибки. Можно успешно использовать ссылки и не платить за копии. Можно также избегать лишнего обертывания в слой корутины и делать `return foo()` напрямую.


## Полезные ссылки

1. https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rcoro-capture
2. https://en.cppreference.com/w/cpp/language/coroutines
3. https://reductor.dev/cpp/2023/08/10/the-downsides-of-coroutines.html