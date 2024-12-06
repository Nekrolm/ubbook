# Унарный минус и беззнаковые числа

Вы разрабатываете графический интерфейс для игры. У вас уже есть кнопочки, панельки, иконки. Все отлично. И вот вы решаете, чтоб интерфейс ощущался интереснее, реализовать анимацию для _check box_ элемента -- при нажатии для снятия галочки, check_box элемент будет красиво отъезжать в сторону где-то на 30% своей ширины.


У вас были подобные структуры и функции
```C++
struct Element {
    size_t width; // original non scaled width
    ...
};

using ElementID = uint64_t; // you are using smart component system that uses IDs to refer to elements

// Positions in OpenGL/DirectX/Vulkan worlds are floats
struct Offset {
    float x;
    float y;
};

size_t get_width(ElementID);
float screen_scale();
void move_by(ElementID, Offset);
```

И вы добавили свою

```C++
void on_unchecked(ElementID el) {
    auto w = get_width(el);
    move_by(el, Offset {
        -w * screen_scale() * 0.3f,
        0.0f
    });
}
```

Ваш check_box имел ширину 50 пикселей. Вы запустили тест... И элемент улетел за пределы экрана!

Вы пошли смотреть логи и [обнаружили](https://godbolt.org/z/hbccqG5r8)
```
Offset: 5.5340234e+18 0
```

Как же так?! Неопределенное поведение?

Нет. Вполне определенное.

Всему виной унарный минус, который мы случайно применили к беззнаковой переменной.

```
For unsigned a, the value of -a is 2^N − a, where N is the number of bits after promotion.
```

Это очень злобная ошибка, которую **не** диагностируют Clang и GCC с флагами `-Wall -Wextra -Wpedantic`;
MSVC же имеет такую [диагностику](https://learn.microsoft.com/en-us/cpp/error-messages/compiler-warnings/compiler-warning-level-2-c4146?view=msvc-170)

Статические анализаторы, например, [PVS-studio](https://pvs-studio.ru/ru/docs/warnings/v2553/) также могут найти ошибку. 

В более современных языках программирования применение унарного минуса к беззнаковым значениям чаще всего не компилируется. Так, например, сделано в Rust, Zig и в Kotlin.


## Полезные ссылки
1. [C: unary minus operator behavior with unsigned operands](https://stackoverflow.com/questions/8026694/c-unary-minus-operator-behavior-with-unsigned-operands)