# Неинициализированные переменные 

Очень известный и распространенный источник проблем и не только в C/C++. 

Новые современные языки программирования обычно запрещают использование неинициализированных переменных. Переменные либо всегда инициализируются значением по умолчанию (например, в [Go](https://golang.org/ref/spec#The_zero_value)). Либо попытка чтения из неинициализированной переменной дает ошибку компиляции (в [Kotlin](https://pl.kotl.in/PoVXtB7AB) или в [Rust](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=261f92c8ca39b10c1ac565e4f8a1e28a)). 

C и C++ — старые языки. В них можно легко и просто объявить переменную, а инициализировать ее как-нибудь потом. Или забыть инициализировать вовсе. Но в отличие от совсем низкоуровневого ассемблера, в котором читать из неинициализированной переменной никто не запрещает — ну получите вы свои мусорные байтики и ладно — в C/C++ (а также в Rust, см [MaybeUninit](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html)) это влечет за собой неопределенное поведение.

Но время не стоит на месте даже для C++ и в последних версиях стандарта (C++26 и новее) всё же произошли некоторые изменения: стандарт вводит новое понятие *ошибочного* *(erroneous)* поведения. И ошибка чтения неинициализированной переменной считается теперь ошибочным, а не неопределенным поведением. На практике же это значит, что вы все также успешно отстрелите себе ногу, но компиляторам рекомендуется выдать диагностику и запрещается делать оптимизации — какой-то определенный мусор должен быть успешно прочитан, а оптимизации кода до и после такого чтения не должны делать никаких предположений об этом мусорном значении.

Например, 
```C++
void test() {
    int x;
    if (x) {
        printf("non zero\n");
    } else {
        printf("zero\n");
    }
}
int main() {
    test();
    test();
}
```
Сейчас Clang 19 с -O3 оптимизирует код выше в [ничего](
https://godbolt.org/z/cx88ezrnG), считая код недостижимым из-за неопределенного поведения в нем!
```
test():

main:

ASM generation compiler returned: 0
Execution build compiler returned: 0
Program returned: 48
```
В C++26 ожидается, что все-таки какой-то результат должен быть.

При этом если прочитанный мусор представит собой все-таки невалидное значение для данного типа переменной, то все спецэффекты неопределенного поведения остаются в силе! 

Неожиданный вариант такого UB можно наблюдать на следующем примере (взято [тут](https://stackoverflow.com/questions/54120862/does-the-c-standard-allow-for-an-uninitialized-bool-to-crash-a-program)):


```C++
struct FStruct {
    bool uninitializedBool;
    
   // Конструктор, не инициализирующий поля.
   // Чтобы проблема воспроизвелась, конструктор должен быть определен в другой единице трансляции
   // Можно сымитировать с помощью атрибута noinline 
   __attribute__ ((noinline)) 
   FStruct() {};
};

char destBuffer[16];

void Serialize(bool boolValue) {
    const char* whichString = boolValue ? "true" : "false";
    size_t len = strlen(whichString);
    memcpy(destBuffer, whichString, len);
}

int main()
{
    // Конструируем объект с неинициализированным полем
    FStruct structInstance;
    
    // UB!
    Serialize(structInstance.uninitializedBool);
    
    //printf("%s", destBuffer);
    return 0;
}
```

Программа [падает](https://godbolt.org/z/rvren9er8). Поскольку неинициализированных переменных в корректной программе не бывает, компилятор полагает `boolValue` всегда валидным и выполняет следующую занятную оптимизацию:
```C++
// size_t len = strlen(whichString); // 4 или 5!
   size_t len = 5 - boolValue;
```
--------------

Так если отсутствие неинициализированных переменных способствует оптимизациям, почему бы их не запретить совсем, с жесткой ошибкой компиляции?

Во-первых, они позволяют экономить на спичках:

```C++
int answer;
if (value == 5) {
    answer = 42;
} else {
    answer = value * 10;
}
```

Если бы нам было запрещено объявлять переменную без инициализации, мы бы либо вынуждены были написать
```C++
int answer = 0;
```
И потратить в отладочной сборке целую одну лишнюю инструкцию `xor` на зануление!

Либо завернуть вычисление `answer` в отдельную функцию (или лямбда-функцию) и получить целый `call` вместо `jmp`, если компилятор не отоптимизирует!

Либо использовать тернарный оператор и получить что-то совершенно нечитаемое, если веток условий будет больше.

Во-вторых, иногда спички большие и дорогие. И экономия оправдана:

```C++
    constexpr int data_size = 4096;
    char buffer[data_size];
    read(fd, buffer, data_size);
```

Инициализировать целый массив чтобы тут же его перетереть — не разумно. И маловероятно что компилятор эту инициализацию отоптимизирует: для этого ему нужны гарантии что условная функция `read` не читает ничего из буфера. Такие гарантии могут быть зашиты для функций стандартной библиотеки, но не для пользовательских.

## Получаем неинициализированные переменные и избегаем их

Прежде всего: какие конструкции порождают неинициализированные переменные?

Специальные функции, например, `std::make_unique_for_overwrite` мы не рассматриваем. Функции выделения сырой памяти: `*alloc` тоже. Хотя напомнить, что писать `(T*)malloc(N)` в ожидании инициализированной памяти нельзя. 


В более общем случае, если верно, что  `is_trivially_constructible<T> == true`, то

1. `T x;`
2. `T x[N];`
3. `T* p = new T;`
4. `T* p = new T[N]`;

[Порождают](https://godbolt.org/z/41d99n5Mr) неинициализированные переменные/массивы (или указатели на неинициализированные переменные/массивы)

Если тип нетривиально конструируемый, не спешите радоваться. Его конструктор по умолчанию мог [забыть](https://godbolt.org/z/T3bs5fb98) что-то проинициализировать. Или кто-то [предоставил деструктор](https://godbolt.org/z/hr6r1Ys6T) чтобы всех запутать. Или [виртуальный метод](https://godbolt.org/z/4q9qE4a1e). 

Распространенный совет по повсеместному использованию {} при объявлении переменных работает и гарантирует инициализацию нулями только с тривиальными типами. Для нетривиальных — все на совести [конструктора](https://godbolt.org/z/j4zjrdo8E).

Но иногда вам может «повезти» и инициализация пройдет (гарантированно стандартом!) в два этапа: сначала нулями, потом вызовется конструктор по умолчанию. Подробнее [тут](https://en.cppreference.com/w/cpp/language/value_initialization).
Мне удалось воспроизвести этот эффект только при использовании `std::make_unique`;

Как бороться с неинициализированными переменными и связанным с ним неопределенным поведением?

1. Не разрывать объявление и инициализацию. Вместо этого использовать конструкции:
```C++
auto x = T{...};
auto x = [&] { ... return value }();
```
2. Проверять свои конструкторы, что в них инициализированы все поля.
3. Пользоваться инициализаторами по умолчанию при объявлении полей структур
4. Использовать свежие версии компиляторов: последний (на момент написания заметки) gcc 11.2 [способен предупреждать](https://godbolt.org/z/663P1Wq59) об обращении к неинициализированным значениям. Но не всегда способен. 

5. Не использовать `new T`, если вы не уверены в том, что делаете. Всегда `new T{}` или `new T()`.
6. Не забывать про динамический и статический анализ внешними утилитами. Valgrind умеет ловить обращения к неинициализированной памяти.


### И последнее

Если к вам когда-нибудь придет светлая мысль использовать неинициализированную память в качестве источника случайности, гоните её как можно быстрее! Некоторые пробовали — [не получилось](https://kqueue.org/blog/2012/06/25/more-randomness-or-less/).
