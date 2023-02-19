# char и знаковое расширение

Возьмем следующую простенькую структуру

```C++
// пример утащен и изменен отсюда:
// https://twitter.com/hankadusikova/status/1626960604412928002
struct CharTable {
    static_assert(CHAR_BIT == 8);
    std::array<bool, 256> _is_whitespace {};

    CharTable() {
        _is_whitespace.fill(false);
    }

    bool is_whitespace(char c) const {
        return this->_is_whitespace[c];
    }
};
```

Все ли впорядке с этим безобидным методом `is_whitespace`? Ну кроме того, что `char` в C/C++ обычно восьмибитный, а в unicode [есть](https://jkorpela.fi/chars/spaces.html) пробельные символы, кодируемые 16 битами.


Давайте [потестируем](https://godbolt.org/z/75rTW1nMG)

```C++
int main() {
    CharTable table;
    char c = 128;
    bool is_whitespace = table.is_whitespace(c);
    std::cout << is_whitespace << "\n";
    return is_whitespace;
}
```

При сборке с `-fsanitize=undefined` получаем дивный результат

```
/opt/compiler-explorer/gcc-12.2.0/include/c++/12.2.0/array:61:36: runtime error: index 18446744073709551488 out of bounds for type 'bool [256]'
/opt/compiler-explorer/gcc-12.2.0/include/c++/12.2.0/array:61:36: runtime error: index 18446744073709551488 out of bounds for type 'bool [256]'
/app/example.cpp:14:38: runtime error: load of value 64, which is not a valid value for type 'bool'
```

Конкретное значение в третьей строке -- совершенно случайное. Было бы очень здорово стабильно увидеть 42, но увы.

Зато индекс в первых двух строках совсем не случайный.

Но погодите! 
`char c = 128;`  это же точно меньше 256. Откуда `18446744073709551488`?

Будем разбираться. В деле замешаны две удачно разложенные ловушки.

1. С/C++ специфичная ловушка: знаковость типа `char` не специфицирована. В зависимости от платформы он может быть как знаковым, так и беззнаковым. На x86 чаще всего является знаковым. И из `char c = 128` получается `c = -128`.

2. Ловушка, распространенная во многих языках, имеющих разные типы целых чисел, разной знаковости и длины. Например, [Rust](https://godbolt.org/z/cY1v3rvrK)
```Rust
pub fn main() {
    let c : i8 = -5;
    let c_direct_cast = c as u16;
    let c_two_casts = c as u8 as u16;
    println!("{c_direct_cast} != {c_two_casts}");
}
```
Мы увидим `65531 != 251`.

При преобразовании знакового целого меньшей длины к беззнаковому целому большей длины происходит знаковое расширение: старшие биты заполняются битом знака.

Тоже [действует и в C/C++](https://godbolt.org/z/cfcdb5fr3).


А теперь остается только взглянуть на сигнатуру `std::array::operator[]`:
```C++
reference operator[]( size_type pos );
```

`size_type` это беззнаковый `size_t`. Под x86 он определенно больше чем `char`.
Происходит прямой каст знакового `char` в `size_t`, знак расширяется, код ломается. Дело закрыто.

## Что делать

Со знаковым расширением иногда способны помочь статические анализаторы.
Нужно понимать что вы делаете при касте чисел и что хотите получить. Часто можно встретить конструкцию вида `uint32_t extended_val = static_cast<uint32_t>(byte_val) & 0xFF`, чтоб гарантированно занулить верхние байты и избежать знакового расширения. Аналогичная конструкция может быть и при преобразовании `int32 -> uint64`, и при любых других комбинациях -- только константу правильную писать не забывайте. 

Из-за своей знаковой неспецифицированности тип `char` очень опасен при работе с ним как с типом чисел. Крайне рекомендуется пользоваться соответствующими типами `uint8_t` или `int8_t`. Или другими подходящими, если на вашей целевой платформе в `char` внезапно не 8 бит. 


## Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/types
2. https://en.cppreference.com/w/cpp/container/array/operator_at
3. https://en.cppreference.com/w/cpp/types/climits
4. https://docs.oracle.com/cd/E19205-01/819-5265/bjamz/index.html