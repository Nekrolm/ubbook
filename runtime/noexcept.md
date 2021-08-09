# Ложный `noexcept`

Начиная с 11 стандарта, мы можем помечать функции и методы спецификатором `noexcept`, говоря тем самым компилятору, что эта функция или метод не бросает исключения.

И вроде бы все хорошо: получив такую информацию, компилятор может не генерировать дополнительные инструкции для обработки раскрутки стека. Бинарники становятся меньше, а программы быстрее.

Но проблема в том, что этот спецификатор не заставляет компиляторы проверять,
что функция действительно не бросает исключений.

Если мы пометим фунцию как `noexcept`, а она возьмет да кинет исключение,
произойдет что-то странное, заканчивающееся внезапным `std::terminate`.

Так, например, неожиданно [перестанут работать](https://godbolt.org/z/E9c9Ya) `try-catch` блоки.

```C++
void may_throw(){
    throw std::runtime_error("wrong noexcept");
}

struct WrongNoexcept {
  WrongNoexcept() noexcept {
     may_throw();
  }
};

// Попытки обернуть в try-catch эту функцию или любой код,
// использующий ее — бесполезны.
void throw_smth() {
    if (rand() % 2 == 0) {
        throw std::runtime_error("throw");
    } else {
        WrongNoexcept w;
    }
}
```

Может быть очень сложно понять почему это произошло, если код разнесен по разным единицам трансляции.

## Условный `noexcept`

В С++ любят экономить на ключевых словах.

- `= 0` для объявления чисто виртуальных методов
- новый `requires` имеет два значения, порождая странные конструкции `requires(requires(...))`
- `auto` и для автовывода, и для переключения на trailing return type
- `decltype`, у которого разный смысл при примении к переменной и к выражению
- и, конечно, `noexcept` — точно также два значения как у `requires`.

Есть спецификатор `noexcept(condition)`. И просто `noexcept` — синтаксический сахар
для конструкции `noexcept(true)`.

А есть предикат `noexcept(expr)`, проверяющий, что выражение `expr` не кидает исключений по самой своей природе (сложение чисел, например) или же
помечено как `noexcept`.

И вместе они порождают конструкцию для условного навешивания noexcept:
```C++
void fun() noexcept(noexcept(used_expr))
```

```C++
void may_throw(){
    throw std::runtime_error("wrong noexcept");
}

struct ConditionalNoexcept {
  ConditionalNoexcept() noexcept(noexcept(may_throw())) {
     may_throw();
  }
};

// теперь с этой функцией все хорошо
void throw_smth() {
    if (rand() % 2 == 0) {
        throw std::runtime_error("throw");
    } else {
        ConditionalNoexcept w;
    }
}
```

Чтобы избежать проблем, нужно всегда и везде использовать условный `noexcept` с аккуратной проверкой каждой используемой функции, либо вовсе не использовать `noexcept`. Но во втором случае стоит помнить,
что операции перемещения, а также `swap`, должны помечаться как `noexcept` (и быть действительно `noexcept`!) для эффективной работы со стандартными контейнерами.

Не забывайте писать негативные тесты. Без них
можно проморгать появление ложного `noexcept` и получить `std::terminate` на боевом стенде.

Также обратите внимание на тонкий и неприятный нюанс: если вам ну очень сильно надо кидать исключения из деструктора, обязательно явно пишите в его объявлении `noexcept(false)`. По умолчанию все ваши функции и методы помечены неявно `noexcept(false)`, но для деструкторов в C++ сделано исключение. Они неявно помечены `noexcept(true)`. [Так что](https://godbolt.org/z/5jo95d):

```C++
struct SoBad {
    // invoke std::terminate
    ~SoBad() {
        throw std::runtime_error("so bad dctor");
    }
};

struct  NotSoBad {
    // OK
    ~NotSoBad() noexcept(false) {
        throw std::runtime_error("not so bad dctor");
    }
};
```

## Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/noexcept
2. https://en.cppreference.com/w/cpp/language/noexcept_spec
3. https://www.modernescpp.com/index.php/c-core-guidelines-the-noexcept-specifier-and-operator
