# Поддержка сборщика мусора (не актуально для C++23 и новее)

Да, вы не ослышались. И глаза вам не врут. И я не сошел с ума. И вы тоже. Скорее всего.

C++ — уникальный язык. В его стандарте есть описание того, чего в языке почти наверняка не появится. Есть поддержка сборщика мусора, но самого сборщика мусора нет. И поддержка эта
сделана самым естественным для C++ способом: введением неопределенного поведения.

Неопределенное поведение возникает в следующей ситуации:
- У вас есть указатель на выделенную в куче память
- Это единственный указатель на эту память
- Вы его каким-то образом прячете: т.е. уничтожаете сам указатель, не освобождая память, но сохраняете возможность этот указатель каким-то образом восстановить
- Восстанавливаете указатель
- Разыменование этого указателя влечет неопределенное поведение

Ну, действительно: если у нас когда-нибудь будет сборщик мусора, то уничтожение последнего указателя на объект позволит сборщику мусора этот объект удалить. А значит последующий доступ к этому объекту ни к чему хорошему не приведет. Сборщик мусора может его успеть удалить. А может не успеть. Вот вам и UB.

Но у нас нет сборщика мусора! Ни один из компиляторов его не поддерживает! А стандарт [поддерживает](https://eel.is/c++draft/util.dynamic.safety)

Так что, если вы, например, храните в младших битах указателя (а это иногда можно делать из-за выравнивания) какую-то метаинформацию, экономя память, скорее всего в вашей программе есть UB, связанное с поддержкой сборщика мусора. Оно почти наверное никогда не выстрелит, но оно есть.

```C++
template <class T>
struct MayBeUninitialized {
    static_assert(alignof(T) >= 2);
    
    MayBeUninitialized() {
        // Выделяем сырую память с помощью явного вызова operator new.
        // Вся эта ерунда с поддержкой сборщика мусора описана
        // только для глобального operator new. std::malloc, placement new 
        // и прочие не участвуют.
        ptr_repr_ = reinterpret_cast<uintptr_t>(
                                    operator new (sizeof(T), 
                                                  std::align_val_t(alignof(T))));
        // единственный указатель только был создан и сразу же уничтожился
        ptr_repr_ |= 1; // set unitialized flag
    }

    ~MayBeUninitialized() {
        Deinit();
        operator delete(GetPointer(), sizeof(T), std::align_val_t(alignof(T)));
    }

    void Deinit() {
        if (!IsInitialized()) {
            return;
        }
        GetPointer()->~T();
    }

    bool IsInitialized() const {
        return !(ptr_repr_ & 1);
    }

    void Set(T x) {
        Deinit();
        new (GetPointer()) T(std::move(x));
        // drop unitialized flag
        ptr_repr_ &= (~static_cast<uintptr_t>(1));
    }


    const T& Get() const {
        if (!IsInitialized()) {
            throw std::runtime_error("not init");
        }
        return *GetPointer(); // UB
    }

private:
    T* GetPointer() const {
        constexpr auto mask = ~static_cast<uintptr_t>(1);
        auto ptr = reinterpret_cast<T*>(ptr_repr_ & mask);
        // восстановили указатель. но разыменование его — UB
        return ptr;
    }

    uintptr_t ptr_repr_;
};
```

Устраняется данное недоразумение с бессмысленным для текущего положения дел в C++ неопределенным поведением при помощью пары функций
`declare_reachable` и `undeclare_reachable`.

```C++
    MayBeUninitialized() {
        void* ptr = operator new (sizeof(T), std::align_val_t(alignof(T)));
        std::declare_reachable(ptr);
        ptr_repr_ = reinterpret_cast<uintptr_t>(ptr);
        // единственный указатель только был создан и сразу же уничтожился, но
        // мы пометили память под ним достижимой, чтобы отвадить мифический сборщик мусора
        ptr_repr_ |= 1; // set unitialized flag
    }
    
    ~MayBeUninitialized() {
        Deinit();
        void* ptr = GetPointer();
        std::undeclare_reachable(ptr);
        operator delete (ptr, sizeof(T), std::align_val_t(alignof(T)));
    }
```

Эти функции в настоящеее время ничего не делают. Они нужны только для формального следования букве стандарта.

Если вы верите, что когда-нибудь в C++ появится сборщик мусора, будьте любезны пользоваться этими прекрасными функциями, чтобы ваша программа оставалась корректной и в далеком будущем.

Если не верите, можете про них забыть. Пожалуй, это единстенное UB, которое нигде и никак не проявляется. И не проявится. Скорее всего не проявится. Даже есть предложения [удалить](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2186r0.html) эту совершенно дурную для C++ «фичу».

--------

Надо понимать, что сам по себе сборщик мусора для C++ не является чем-то сверхъестественным. На C/C++ написаны, например, сборщики мусора для JVM. Никто не мещает задействовать их же в C++-программах: просто используем альтернативные функции для выделения памяти. С их помощью даже можно переопределить поведение операторов `new` и `delete`. Но очень мало какой код на C++ пишется в предположении, что под этими операторами работает сборщик мусора.

Проверить, не запустили ли вашу программу в светлом мире со сборщиком мусора, можно вызвав функцию `get_pointer_safety`. Она возвращает одно из трех значений:
- `pointer_safety::strict`    — играть с восставновлением указателей абы откуда просто так нельзя; сборщик мусора, возможно, работает.
- `pointer_safety::relaxed`   — с указателями нет никаких проблем, выделенная память никуда сама по себе не денется.
- `pointer_safety::preferred` — с указателями нет никаких проблем, выделенная память никуда сама по себе не денется, но, возможно, работает детектор утечек, которому важны пометки `declare_reachable`/`undeclare_reachable`.

```C++
int main() {
    switch (std::get_pointer_safety())
    {
    case std::pointer_safety::strict:
        std::cout << "strict" << std::endl;
        break;
    case std::pointer_safety::relaxed:
        std::cout << "relaxed" << std::endl;
        break;
    default:
        std::cout << "preferred" << std::endl;
    }
 }
```

Отмечу, что при запуске этого кода под `valgrind-3.15.0` для `Ubuntu 20.04 (x86_64)` выводимое сообщение (`relaxed`) никак не меняется.


## Полезные ссылки
1. https://en.cppreference.com/w/cpp/memory/gc/undeclare_reachable
2. https://en.cppreference.com/w/cpp/memory/gc/declare_reachable
3. https://eel.is/c++draft/util.dynamic.safety
4. http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2186r0.html
5. http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2670.htm
6. https://en.wikipedia.org/wiki/Boehm_garbage_collector
