# Повторный захват mutex

Deadlock это, конечно, печально. Система завязалась в узел и никогда не развяжется.
А сколько мьютексов нужно, чтобы уйти в deadlock?

Немного подумав, можно решить, что одного достаточно — просто захвати его два раза подряд, не отпуская, в одном и том же потоке.

Возможно, под какой-то платформой это и так. Но в C++ это неопределенное поведение и
для красивого показательного дедлока нужно два мьютекса. А с одним — ваш фокус не удастся и превратится в фокус от мира UB.

```C++
struct Test{
    std::mutex mutex;
    std::vector<int> v = { 1,2,3,4,5};
    
    auto fun(int n){
        mutex.lock();  // захватываем
        return std::shared_ptr<int>(v.data() + n, 
                                    [this](auto...){mutex.unlock();});
                                    // освободим при смерти указателя
    }
};
    
    
int main(){
    
    Test tt;
    auto a = tt.fun(1); // захватили первый раз
    std::cout << *a << std::endl;
    // указатель жив
    auto b = tt.fun(2); // захватили второй раз. UB
    std::cout << *b << std::endl;
    
    return 0;
}
```

Этот пример дает [разные](https://godbolt.org/z/aoren4) результаты на одном и том же компиляторе, на одной и той же платформе, на одном и том же уровне оптимизаций. Просто подключили `pthread` или нет.

Кто в здравом уме будет такое делать-то? Никто же никогда не захватывает один и тот же мьютекс два раза подряд.

Даже не знаю... Зачем-то же существуют рекурсивные мьютексы, которые можно захватывать по нескольку раз.

Да и сводить задачу к уже решенной и переиспользовать написанный код любят:

```C++
template <class T>
struct ThreadSafeQueue<T> {

bool empty() const {
    std::scoped_lock lock { mutex_ };
    ...
}

void push(T x) {
    std::scoped_lock lock { mutex_ };
    ...
}

std::optional<T> pop() {
    std::scoped_lock lock { mutex_ };
    if (empty()) { // ! ПОВТОРНЫЙ ЗАХВАТ !
        return std::nullopt;
    }
    ...
}

...
std::mutex mutex_;
};
```

Чтобы исправить, нужно либо подумать, либо использовать рекурсивный мьютекс.
Но лучше подумать.

Методов у объекта может быть много. Разработчиков тоже. Они могут не помнить, где есть блокировка, а где нет. Могут засунуть блокировку в один метод, забыв про другие. Так что от повторного захвата мьютекса в одном и том же потоке никто не застрахован.
