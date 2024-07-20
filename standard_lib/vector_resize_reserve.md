# std::vector::reserve и std::vector::resize

Очень многие программы на C++ становились и становятся жертвами подлого обмана со стороны этих двух братьев-близнецов. А также, естественно, невнимательности, спешки и автодополнения.

Почти все книжки по программированию на C++ учат, что при создании `std::vector` желательно заранее резервировать память, особенно если вы знаете сколько у вас будет элементов в нем. Тогда при наполнении вектора он не будет реаллоцироваться, а значит ваша программа будет работать быстрее, не тратя время на перевыделение памяти.

Но вот беда. У `std::vector` нет конструктора, в котором бы можно было указать: "Хочу пустой вектор, но с зарезервированной capacity = N".
у вектора есть другой конструктор -- заполняющий `N` элементами по-умолчанию 

```C++
// Создаст N пустых строк
std::vector<std::string> text(N); 
```

Но это ведь не оптимально. Создать целый вектор, **проинициализировать** каждый элемент в нем, чтоб потом переписать их... Нет-нет, это неправильное использование C++!

У нас есть метод `reserve()`, его надо вызвать после создания пустого вектора.

```C++
std::vector<std::string> text;
text.reserve(N);
```

Но снова напасть! Ведь у нас есть еще и метод `resize()`, который также может принять ровно один аргумент, если элементы вектора конструируемы по-умолчанию.

И разумеется программисты их путают
- Имена короткие и начинаются одинаково
- Имеют схожий смысл
- Стоят рядом в выдаче автодополнения
- И в наши дни еще и ИИ-ассистент может не того из них посоветовать!

В итоге программист успешно создаст вектор в два раза больше чем хотелось и будет долго недоумевать, почему у него все интересующие элементы пустые

```C++
auto read_text(size_t N_lines) {
    std::vector<std::string> text;
    text.resize(N_lines); // Ай!
    for (size_t line_no = 0; line_no < N_lines; ++line_no) {
        std::string line;
        std::getline(std::cin, line);
        text.emplace_back(std::move(line));
    }
    return text; 
}
```

Но никакого неопределенного поведения. Эх! 

Но что если программист написал `reserve()` там, где на самом деле требовался `resize()`? Ну случайно. Причем эта случайность имеет довольно неплохой шанс: ведь выдача автодополнения часто упорядочена по алфавиту, а  `reserve` стоит в нем раньше.

```C++
std::pair<std::vector<std::byte>, size_t> read_text(std::istream& in, size_t buffer_len) {
    std::vector<std::byte> buffer;
    buffer.reserve(buffer_len);
    in.read(reinterpret_cast<char*>(buffer.data()), buffer_len);
    return {
        std::move(buffer), static_cast<size_t>(in.gcount())
    };
}

int main() {
    auto [buffer, actual_size] = read_text(std::cin, 45);
    for (size_t i = 0; i < actual_size; ++i) {
        std::cout << static_cast<int>(buffer[i]) << "\n";
    }
}
```
Программа будет успешно [работать!](https://godbolt.org/z/qoTx4GsWa)
```
// Пример ввода/вывода
>> hello
104
101
108
108
111
```

И вот здесь стоит на минуту остановиться и отвлечься.
Я множество раз видел на самых разных технических форумах заявления вида:
- С/C++ это языки для работы близко к железу. Undefined behavior это просто формальность. Есть программист знает, как работает память, как работает его программа и типы, он может эту ерунду игнорировать. И так далее.

Случай с `reserve()` выше может быть использован в защиту такой спорной позиции.

Действительно. Я знаю, что
1. `reserve()` действительно выделяет память так что диапазон `[buffer.data(), buffer.data() + buffer_len)` валиден.
2. `std::istream::read` проинициализировал память в диапазоне `[buffer.data(), buffer.data() + actual_size)`
3. `operator[]` вектора по умолчанию **не проверяет** переданный индекс
4. Итерации цикла доступа к вектору проходят в инициализированных пределах.

Поэтому все *работает*. Более того, оно даже работает без language undefined behaviour. Если вы скопируете реализацию std::vector к себе, удалите из нее все asserts с условными инструментациями санитайзеров, и станете пользоваться вот таким же странным образом, у вас в коде не будет неопределенного поведения. По крайней мере в этом примере.

Но. Это `std::vector`. `std`! И он декларирует library undefined behaviour -- вы обратились к элементу с индексом пределами `[0, vector::size())`. А это не прощается.

Я не смог найти ни одного компилятора доступного онлайн, на котором бы можно было бы воспроизвести последовательность оптимизаций, приводящих к падению невероятной красоты. Но я видет несколько закрытых bug репортов в отношении Apple Clang, который такое проворачивал.

LLVM может генерировать под x86 инструкцию `ud2` -- это недопустимая инструкция, часто используема как индикатор недостижимого кода. Если программа попытается ее выполнить, она умрет от сигнала SIGILL.
Код, который провоцирует неопределенное поведение может быть помечен как недостижимы и в дальнейшем заменен на `ud2` или выброшен.
В нашем замечательном примере компилятору вполне известно, что `buffer.size() == 0`. И его не меняли.

Так, например, если мы попробуем 1 в 1 переписать это же безобразие в Rust, агрессивно использующем возможности LLVM:

```Rust
fn read(n: usize, mut reader:  impl std::io::Read) -> std::io::Result<(Vec<u8>, usize)> {
    // Резервируем память. Она будет неинециализированной
    let mut buf = Vec::<u8>::with_capacity(n);
    // unsafe Rust довольно сложен и формально здесь
    // нельзя напрямую создавать &mut [u8] на неинициализированную память
    // но "мы знаем что делаем" -- на результат это не повлияет. Пока.
    let actual_size = reader.read(unsafe {
        std::slice::from_raw_parts_mut(buf.as_mut_ptr(), n)
    })?;
    // В отличие от C++, в Rust можно сделать
    // unsafe { buf.set_len(actual_size) };
    // И сделать этот пример практически корректным.
    // Но мы здесь собрались смотреть на Undefined Behavior
    Ok((buf, actual_size))
}

pub fn main() {
    let (buf, n) = read(42, std::io::stdin()).unwrap();
    for i in 0..n {
        println!("{}",
            unsafe { buf.get_unchecked(i) }
        )
    }
}
```
В дебажной сборке мы упадем с ошибкой сегментации и сообщением
```
thread 'main' panicked at library/core/src/panicking.rs:220:5:
unsafe precondition(s) violated: slice::get_unchecked requires that the index is within the slice
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread caused non-unwinding panic. aborting.
Program terminated with signal: SIGSEGV
```

В релизной с `-C opt-level=3` мы [увидим](https://godbolt.org/z/jvz5Y66n6) пустой вывод и успешный выход.
И если посмотрим на сгенерированный код, то цикла мы в нем не обнаружим. Код обращения к элементам вектора был полностью выброшен как недостижимый. Спасибо аннотации `assert_unsafe_precondition!(check_language_ub, ...)`.
```
example::main::h67df0b7f9b5f8d1a:
        ...
        call    qword ptr [rip + <std::io::stdio::Stdin as std::io::Read>::read::h30ce8d6974df759c@GOTPCREL]
        mov     esi, 42
        test    rax, rax
        jne     .LBB1_3    # Это unwrap
        mov     edx, 1
        mov     rdi, rbx
        call    qword ptr [rip + __rust_dealloc@GOTPCREL]
        add     rsp, 8
        pop     rbx
        pop     r14
        ret
.LBB1_7:
```

"Ну это же Rust!" -- можно возразить, закатывая глаза. Да. Но это лишь дело времени, когда Clang начнет применять те же оптимизации к C++.

### Что же делать?!

По-хорошему, конечно, не ошибаться и не путать `reserve()` и `resize()`...

Как выяснилось после множества экспериментов с разными утилитами, состояние диагностики подобного standard library level неопределенного поведения в C++, в 2024 году остается весьма плачевным

1. Статические анализаторы, к сожалению, [молчат](https://godbolt.org/z/sjYjcrbKf).
2. Санитайзеры по-умолчанию тоже [не реагируют](https://godbolt.org/z/P1P8en6zh) 
3. `_ITERATOR_DEBUG_LEVEL` от msvc молчаливо [падает](https://godbolt.org/z/GEKWsabWj)
4. `-fsanitize=address` [перестает молчать](https://godbolt.org/z/3vc7GGTGv) только лишь с `-stdlib=libc++`
```
==1==ERROR: AddressSanitizer: container-overflow on address 0x504000000050 at pc 0x59481461d1b0 bp 0x7ffcf01b08b0 sp 0x7ffcf01b08a8
READ of size 1 at 0x504000000050 thread T0
```

Но стойте-стойте! А что если это не ошибка. Мы сознательно использовали `reserve()`, так как он не инициализирует память. И хотели ее, как в примере с Rust, переписать какими-нибудь данными из файла и в конце изменить `size()`.  Но вектор просто не предоставляет такое API...

На этот случай в стандартной библиотеке C++ есть целых два более корректных способа

### std::make_unique_for_overwrite

```C++
std::pair<std::unique_ptr<std::byte[]>, size_t> read_text(std::istream& in, size_t buffer_len) {
    auto buffer = std::make_unique_for_overwrite<std::byte[]>(buffer_len);
    in.read(reinterpret_cast<char*>(buffer.get()), buffer_len);
    // Выделяем default-инициализированный буфер, но default инициализация
    // массива тривиальных объектов это отсутствие инициализации.
    return {
        std::move(buffer), static_cast<size_t>(in.gcount())
    };
}
```

Такой вариант также успешно [работает](https://godbolt.org/z/qrh9xferr) как и первоначальный неправильный, но уже без неопределенного поведения

Но разумеется мы таким образом успешно потеряли информацию об оставшейся вместимости. Ведь она не привязана к `unique_ptr`. Нужно привязать ее отдельно в рамках собственной структуры. Или забыть.

### C++23. std::basic_string::resize_and_overwrite

Да! Чудо случилось и в C++23 мы можем сделать почти также замечательно эффективно как в Rust. Но только для "строк". Но ведь по старой доброй традиции из С, у нас строки это просто последовательность байт...

```C++
// Придется написать немного СharTraits магии, если мы хотим использовать std::basic_string c типом std::byte
struct ByteTraits {
    using char_type = ::std::byte;
    static char_type* copy(char_type* dst, char_type* src, size_t n) {
        memcpy(dst, src, n);
        return dst;
    }
    static void assign(char_type& dst, const char_type& src) {
        dst = src;
    } 
};

std::basic_string<std::byte, ByteTraits> read_text(std::istream& in, size_t buffer_len) {
    std::basic_string<std::byte, ByteTraits> buffer;
    buffer.resize_and_overwrite(buffer_len, [&](std::byte* buf, size_t len) {
        in.read(reinterpret_cast<char*>(buf), buffer_len);
        return static_cast<size_t>(in.gcount());
    });
    
    return buffer;
}

int main() {
    auto buffer = read_text(std::cin, 45);
    size_t actual_size = buffer.size();
    std::cout << actual_size << std::endl;
    for (size_t i = 0; i < actual_size; ++i) {
        std::cout << static_cast<int>(buffer[i]) << "\n";
    }
}
```

И ура! Оно также [работает](https://godbolt.org/z/PGzasjbcn) как ожидается.
