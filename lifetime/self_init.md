# Еще не мертв, но еще и не жив. Self-reference

Область видимости объекта начинается сразу же после его объявления. В той же строчке. Поэтому
в С++ очень легко сконструировать синтаксически корректное выражение, использующее еще не сконструированный объект.

```C++
// просто и прямолинейно
int x = x + 5; // UB

//--------------
// менее явно
const int max_v = 10;

void fun(int y) {
   const int max_v = [&]{
       // локальный max_v перекрывает глобальный max_v
       return std::min(max_v, y);
   }();
   ...
}
```

Конечно, такой код вряд ли кто-то будет писать целенаправлено.
Но он может возникать самопроизвольно при применении средств автоматического
рефакторинга. Локальный `max_v` во втором примере мог изначально называться как-то по-другому. Применили автоматическое переименование и получили вместо некомпилирующегося кода, код с неопределенным поведением.

Причем в следующей версии никакой проблемы не возникает:

```C++
const int max_v = 10;

void fun(int y) {
   const int max_v = [y]{
       // тут виден только глобальный max_v
       return std::min(max_v, y);
   }();
   ...
}
```

Код, уходящий в область неопределенного поведения при добавлении лишь одного символа — все как мы любим.

---
Такой код синтаксически валиден и никто не собирается его запрещать. Более того, он еще и [не всегда](https://godbolt.org/z/7jqo61) приводит к UB.
К UB приводит только использование с, грубо говоря, разыменованием ссылки на этот объект. Почему грубо? Потому что правила такие же, как и с разыменованием `nullptr` — то есть довольно [путанные](https://habr.com/ru/post/513058/), а не просто лишь «никогда нельзя — всегда UB». Хотя использование такой радикальной трактовки уберет вас от многих бед.

```C++
struct ExtremelyLongClassName {

    using UnspeekableInternalType = size_t;

    UnspeekableInternalType val;

    static UnspeekableInternalType Default() { return 5;}
};

ExtremelyLongClassName x { x.Default() + 5 }; // Ok, well-defined


ExtremelyLongClassName y {
    [] ()-> ExtremelyLongClassName::UnspeekableInternalType {
        // сложные вычисления
        return 1;
    }()
};

ExtremelyLongClassName z {
    [] ()-> decltype(z.Default()) { // Ok, well-defined
        // сложные вычисления
        return 1;
    }()
 };
```

Также эта фича может быть полезна в каких-то [специфических](https://godbolt.org/z/qY18c8) случаях, в которых вам зачем-то нужен объект, ссылающийся сам на себя


```C++
struct Iface {
    virtual ~Iface() = default;
    virtual int method(int) const = 0;
};

struct Impl : Iface {
    explicit Impl(const Iface* other_ = nullptr) : other(other_) {

    };

    int method(int x) const override {
        if (x == 0) {
            return 1;
        }
        if (other){
           return x * other->method(x - 1);
        }
        return 0;
    }
    const Iface* other = nullptr;
};

int main() {
    Impl impl {&impl};
    std::cout << impl.method(5);
}
```

Точно таким же образом, но более запутанно, можно завязать объекты в узел, используя делегирующие конструкторы. Но об этом в отдельной заметке.

Избежать использования объекта при инициализации его же самого можно, следуя правилу `AAA` (almost always auto):

Всегда, если это возможно, использовать запись `auto x = T {....}` для объявления и инициализации переменных.

В такой записи использование объявляемой переменной внутри инициализирующего дает [ошибку компиляции](https://godbolt.org/z/P1rj66).


## Полезные ссылки
1. https://habr.com/ru/post/513058/
2. http://cginternals.github.io/guidelines/articles/almost-always-auto/
