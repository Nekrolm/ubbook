# Невыровненные ссылки

Программист форматировал байтики. Ведь это же самое любимое развлечение C++ программистов: писать снова и снова кода для форматного вывода пользовательских структур.

Байтики у программиста были упакованными, чтоб никакого лишнего выравнивания! И поля у него были упорядочены также, чтоб никакого лишнего выравнивания

```C++
#pragma pack(1)
struct Record {
    long value;
    int data;
    char status;
};

int main() {
    Record r { 42, 42, 42};
    static_assert(sizeof(r) == sizeof(int) + sizeof(char) + sizeof(long));
    std::cout << std::format("{} {} {}", r.data, r.status, r.value); // 42 -- '*'
}
```

О проверял этот код с санитайзером, и санитайзер [говорил](https://godbolt.org/z/nxGn3K1Td) ему что все в порядке.

```
Program returned: 0
42 * 42
```

Ну раз все впорядке, то можно больше байтиков отформатировать!

```C++
int main() {
    Record records[] = { { 42, 42, 42}, { 42, 42, 42}  };
    static_assert(sizeof(records) ==2 * ( sizeof(int) + sizeof(char) + sizeof(long) ));
    for (const auto& r: records) {
        std::cout << std::format("{} {} {}", r.data, r.status, r.value); // 42 -- '*'
    }
}
```

И что-то [взорвалось](https://godbolt.org/z/zj81GY8Ec) (под ARM бы уж точно):

```C++
Program returned: 0
/app/example.cpp:16:48: runtime error: reference binding to misaligned address 0x7ffd1eda9f85 for type 'const int', which requires 4 byte alignment
0x7ffd1eda9f85: note: pointer points here
 00 00 00 00 2a 00 00  00 2a 00 00 00 00 00 00  00 00 00 00 00 00 00 00  03 00 00 00 00 00 00 00  b0
```

Да, нельзя читать невыровненную память. Это влечет неопределенное поведение. Мы это уже знаем. Нельзя разыменовывать невыровненный указатель.
Но вот беда. В C++ же есть ссылки. И они тоже обязаны быть правильно выровненными.

Мы точно видим одну ссылку:

```C++
for (const auto& r: records);
```

Но там же не тип `const int`! Ну да. Это `Record` и с ней все в порядке. `#pragma pack(1)` задает требование к выравниванию 1, так что тут никакой проблемы.

Откуда же взялась ссылка на `const int`?

А она у нас неявно взялась. Ведь неявное создание ссылок это ключевая особенность C++!
```C++
template< class... Args >
std::string format( std::format_string<Args...> fmt, Args&&... args );
```

```C++
std::cout << std::format("{} {} {}", r.data, r.status, r.value); // все три поля будут переданы по ссылке!
```
Да, "универсальная ссылка" это все ссылка.

В упакованное структуре поля не выровнены. Ссылки на них брать нельзя.

Но ведь же в первоначальном варианте с одной структурой работало без предупреждений...

Ха! Нам просто повезло, что
- Поля в структуре упорядочены чтоб и без `pragma pack` не было паддинга между ними
- Стек обычно выровнен на `sizeof(void*)` чего достаточно для всех полей в структуре

Мы можем добавить один лишний `char` на стек и все [изменится](https://godbolt.org/z/eb7WM5ddb)
```C++
int main() {
    char data[1];
    Record r { 42, 42, 42};
    memset(data, 0, 1);
    std::cout << std::format("{} {} {}", r.data, r.status, r.value); // 42 -- '*'
}
```
```
Program returned: 0
/app/example.cpp:17:44: runtime error: reference binding to misaligned address 0x7ffe3b4e1f36 for type 'int', which requires 4 byte alignment
0x7ffe3b4e1f36: note: pointer points here
 00 00 00 00 2a 00  00 00 2a 00 00 00 00 00  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  00 00
```


Как же исправить это досадное недоразумение?

Нужно сделать отдельно чтение из каждого поля во временную правильно выровненную переменную -- сделать копию.

```C++
int main() {
    Record records[] = { { 42, 42, 42}, { 42, 42, 42}  };
    for (const auto& r: records) {
        // В C++23 для этого есть замечательный auto()
        std::cout << std::format("{} {} {}", auto(r.data), auto(r.status), auto(r.value)); 
        // В С++20 
        auto data = r.data; auto status = r.status; auto value = r.value;
        std::cout << std::format("{} {} {}", data, status, r.value); 
        // Или совершенно уродливо и не устойчиво к изменениям в типах
        std::cout << std::format("{} {} {}", static_cast<int>(r.data), 
                                             static_cast<char>(r.status), 
                                             static_cast<long>(r.value>));
    }
}
```

-----

В чуть более безопасных языках взятие невыровненных ссылок на поля упакованных структур просто не компилируется

В [Rust](https://godbolt.org/z/Po4bevG17)

```Rust
#[repr(C, packed)]
struct Record {
    value: i64,
    data: i32,
    status: i8, 
}

fn main() {
    let r = Record { value: 42, data: 42, status: 42 };
    // В Rust макросы -- одно из немногих мест, где ссылки могут появляться неявно для читающего код
    println!("{} {} {}", r.data, r.status, r.value); 
    /*
    error[E0793]: reference to packed field is unaligned
    --> <source>:10:26
        |
     10 |     println!("{} {} {}", r.data, r.status, r.value);
        = note: packed structs are only aligned by one byte, and many modern architectures penalize unaligned field accesses
        = note: creating a misaligned reference is undefined behavior (even if that reference is never dereferenced)
        = help: copy the field contents to a local variable, or replace the reference with a raw pointer and use `read_unaligned`/`write_unaligned` (loads and stores via `*p` must be properly aligned even when using raw pointers)
    */

    // Вот так правильно:
    println!("{} {} {}", {r.data}, {r.status}, {r.value});
}
```


