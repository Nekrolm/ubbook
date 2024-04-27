# Многомерный operator[]

C++23 подарил нам долгожданную возможность перегружать `operator[]` с более чем одним параметром. Любители матриц и numpy ликуют. C++ очень похорошел за последние годы!

Вместе с долгожданной фичей разумеется поставляются новые грабли, которые любезно разложены под разработчиков крупных проектов со смесью стандартов разных версий, от которых бизнес требует релизить фичи как можно быстрее, так что на предупреждения компилятора они могут иногда и подзабить...

У вас была библиотека с классом матриц, поддерживающих обращение к отдельным строкам

```C++


#if __has_include(<span>) && __cplusplus >= 202002L
#include <span>

template <class T>
using Span = std::span<T>;

#else 

template <class T>
struct Span {
    T* data;
    size_t size;

    auto begin() { return data; }
    auto end() { return data + size; }

    auto subspan(size_t ofs, size_t len) {
        return Span<T> {
            data + ofs,
            len
        };
    }
};

#endif

struct Row {
    Span<int> data;

    void operator = (int c) {
        for (auto& x: data) x = c;
    }
};

struct Matrix {
    std::vector<int> data;
    size_t cols;

// Эта перегрузка была у вас 20 лет
    Row operator[](size_t row_idx) {
        return {
            Span<int>{data.data(), data.size()}
                    .subspan(row_idx * cols, cols)
        };
    }

// И вот месяц назад вы добавили восхитетельную 
// перегрузку для доступа к элементу.
// Библиотека используется с разными версиями C++, так что 
// перегрузка под feature-control флагом -- все отлично
#ifdef __cpp_multidimensional_subscript
    int& operator[](size_t row_idx, size_t col_idx) {
        return data[row_idx * cols + col_idx];
    }
#endif
};
```

[Some horrible misguided fool](https://x.com/ericniebler/status/1734997577274380681) из соседней команды использует вашу библиотеку и пишет свой прикладной модуль на C++23, не сильно задумываясь о feature-flags стандартов (вы удивитесь, но это невероятно распространенный сценарий!). И он совершает чудовищную ошибку: помещает определение функции в заголовочный файл

```C++
auto compute() -> Matrix {
    Matrix m {
        std::vector<int>(12, 0),
        4, // matrix 3 x 4
    };

    for (int i = 0; i < 3; ++i) {
        for (int j = 0; j < 4; ++j) {
            m[i, j] = i * j;
        }
    }
    return m;
}
```

У него [все работает, все отлично](https://godbolt.org/z/hsfcTaqse).

К ниму приходит коллега из еще одной команды и берет его пакет к себе, в проект с C++17 и вызывает функцию `compute()`

У него все компилируется без каких-либо предупреждений. Запускается. Иногда работает. А иногда валится с [segmentation fault](https://godbolt.org/z/hTWjqE3WM).

Они идут смотреть вывод санитайзеров и valgrind, смотреть core dump, подключаться отладчиком и развлекаться прочими интересными способами,
чтобы обнаружить:

```
=================================================================
==1==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x604000000040 at pc 0x000000401923 bp 0x7ffe36f379e0 sp 0x7ffe36f379d8
WRITE of size 4 at 0x604000000040 thread T0
    #0 0x401922 in Row::operator=(int) /app/example.cpp:38
    #1 0x401922 in compute() /app/example.cpp:68
    #2 0x40116b in main /app/example.cpp:75
    #3 0x749978829d8f  (/lib/x86_64-linux-gnu/libc.so.6+0x29d8f) (BuildId: 962015aa9d133c6cbcfb31ec300596d7f44d3348)
    #4 0x749978829e3f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x29e3f) (BuildId: 962015aa9d133c6cbcfb31ec300596d7f44d3348)
    #5 0x401274 in _start (/app/output.s+0x401274) (BuildId: 18a2ccdd2845a7b183f1249366db2569b4a1d038)

```

Ну да, ведь до C++23 была только одна перегрузка `operator[]`
и в строке `m[i, j] = i * j;` вместо многомерного оператора вызывается одномерный, в который передается результат `operator ,`, то есть второй элемент. 
Вот и приплыли с выходом за границы буфера, если у матрицы столбцов больше чем строк. Не говоря уже о том, что код в принципе теперь сломан и делает что-то другое.

Ах да. Обещанное предупреждения компилятора... Есть предупреждение, да. Но по умолчанию только в [C++20](https://godbolt.org/z/nq4eenaaM).

```
<source>: In function 'Matrix compute()':
<source>:68:16: warning: top-level comma expression in array subscript is deprecated [-Wcomma-subscript]
   68 |             m[i, j] = i * j;
```

В C++17 и ранее нет. А c `-Wall` есть, но другое

```
<source>:68:15: warning: left operand of comma operator has no effect [-Wunused-value]
   68 |             m[i, j] = i * j;
```

Но `-Wunused-value` бывает слишком "шумным", особенно в легаси проектах, и его часто отключают.

Веселого вам перехода с C++17 на C++23, минуя C++20!


