# Путеводитель C++ программиста по неопределенному поведению

#### *Паникуй!*

--------------

### Коротко о том, зачем и почему

Все начинается просто и незатейливо: обычный десятиклассник увлекается программированием, знакомится с алгоритмическими задачками, решения которых должны быть быстрыми. Узнает о языке C++, учит минимальный синтаксис, основные конструкции, контейнеры, решает задачи с предопределенным и всегда корректным форматом ввода и вывода, и горя не знает...

В это же время, где-то в большом мире, матерые разработчики каждый день ругают то одни языки программирования, то другие. По самым разным причинам: не удобно, нет какой-то возможности, много лишних букв писать, ошибки в стандартной библиотеке... Но есть язык, который ругают за все и особенно за такую непонятную и таинственную вещь как неопределенное поведение (undefined behavior, UB).

Спустя лет пять или шесть наш простой десятиклассник, горя не видавший в море оторванных от реальности программ, внезапно узнает, что тем самым горячо нелюбимым языком всегда был, остается и будет его C++.

А потом еще в течение нескольких лет он наткнется на самые кошмарные и невероятные ужасы, поджидающие программистов на C++ почти на каждом шагу. Так и появится эта серия заметок, собирающая наиболее отвратительные примеры, на которые очень легко наткнуться при решении повседневных задач.

--------------

«Преждевременная оптимизация — корень всех зол» (Д. Кнут или Э. Хоар — в зависимости от того, какой источник смотрите). Язык С++, пожалуй, наиболее яркая тому демонстрация: огромное количество ошибок в C++ программах связаны
с неопределенным поведением, заложенным в фундаменте языка просто для того, чтобы дать простор оптимизациям на этапе компиляции.

Если вы собираетесь писать на C++ код, в работоспособности которого хотите быть хоть немного уверенными, стоит знать о существовании различных  подводных камней и ловко расставленных мин в стандарте языка, его библиотеке, и всячески их избегать. Иначе ваши программы будут работать правильно только на конкретной машине и только по воле случая.


**Важно:** этот сборник **не является учебным пособием** по языку и рассчитан на тех, кто уже знаком с программированием, с C++, и понимает основные его конструкции.

----


# Содержание
0. [Что такое UB и как оно проявляется](what_is_ub.md)
1. [Как искать UB?](how_to_find_ub.md)
2. [Сужающие преобразования](numeric/narrowing.md)
3. Целые и вещественные числа
   1. [Переполнение знаковых целых чисел](numeric/overflow.md)
   2. [Числа с плавающей точкой](numeric/floats.md)
   3. [Integer promotion](numeric/integer_promotion.md)
   4. [char и знаковое расширение](numeric/char_sign_extension.md)
4. Нарушения lifetime объектов
   1. [Висячие ссылки — общие случаи](lifetime/use_after_free_in_general.md)
   2. [Автовывод типов и висячие ссылки](lifetime/decltype_auto_and_explicit_types.md)
   3. [std::string_view](lifetime/string_view.md)
   4. [Range-based for](lifetime/for_loop.md)
   5. [Cамоинициализация](lifetime/self_init.md)
   6. [std::vector и инвалидация ссылок](lifetime/vector_invalidation.md)
   7. [Висячие ссылки в лямбдах](lifetime/lambda_capture.md)
   8. [Создание кортежей](lifetime/tuple_creation.md)
   9. [Внезапная мутабельность](lifetime/unexpected_mutability.md)
   10. [Proxy-объекты и ссылки](lifetime/proxy_objects.md)
   11. [use-after-move](lifetime/use-after-move.md)
   12. [lifetime extension](lifetime/lifetime_extension.md)
5. (Не)работающий синтаксис
   1. [Most Vexing Parse](syntax/most_vexing_parse.md)
   2. [Const](syntax/const_launder.md)
   3. [std::move](syntax/move.md)
   4. [Потерянный return](syntax/missing_return.md)
   5. [Эллипсис и функции с произвольным числом аргументов](syntax/c_variadic.md)
   6. [`operator ,`](syntax/comma_operator.md)
   7. [function-try-block](syntax/function-try-catch.md)
   8. [Пустые структуры и типы нулевого размера](syntax/zero_size.md)
   9. [(Не)явное приведение типов](syntax/explicit_but_implicit.md)
   10. [Многомерный operator[]](syntax/multidimensional_subscript.md)
6. Стандартная библиотека
   1. [NULL-терминированные строки](standard_lib/null_terminated_string.md)
   2. [Конструирование std::shared_ptr](standard_lib/shared_ptr_constructor.md)
   3. [потоки ввода/вывода](standard_lib/iostreams.md)
   4. [std::aligned_storage](standard_lib/aligned_storage.md)
   5. [функции стантарной библиотеки как параметры](syntax/function_pass_and_address_restriction.md)
   6. [std::ranges::views](standard_lib/ranges_views_lazy.md)
   7. [`operator[] ` ассоциативных контейнеров](standard_lib/map_subscript.md)
   8. [std::enable_if/std::void_t](standard_lib/enable_if_void_t.md)
   9. [Конструкторы контейнеров](standard_lib/stl_constructors.md)
   10. [std::uniform_int_distribution](standard_lib/uniform_int_distribution.md)
7. Исполнение программы
   1.  [Бесконечные циклы](runtime/endless_loop.md)
   2.  [Рекурсия](runtime/recursion.md)
   3.  [Ложный noexcept](runtime/noexcept.md)
   4.  [Переполнение буфера](runtime/array_overrun.md)
   5.  [Сборщик мусора](runtime/garbage_collector.md)
   6.  [RAII vs (N)RVO](runtime/rvo_vs_raii.md)
   7.  [Разыменование nullptr](runtime/nullptr_dereference.md)
   8.  [Static initialization order fiasco](runtime/static_initialization_order_fiasco.md)
   9.  [Static inline](runtime/static_inline.md)
   10.  [ODR violation](runtime/odr_violation.md)
   11. [Зарезервированные имена](runtime/reserved_names.md)
   12. [Тривиальные типы и ABI](runtime/trivial_types_and_ABI.md)
   13. [Неинициализированные переменные](runtime/uninitialized.md)
   14. [Ranges. Unreachable sentinel](runtime/unreachable_sentinel.md)
   15. [Невиртуальные виртуальные функции](runtime/virtual_functions.md)
   16. [Variable length array](runtime/vla.md)
   17. [ODR violation и разделяемые библиотеки](runtime/dll_and_odr_violation.md)
8. Происхождение указателей
   1. [Невалидные указатели](pointer_prominence/invalid_pointer.md)
   2. [Placement `operator new[]`](pointer_prominence/array_placement_new.md)
9. Параллелизм
   1. [Race condition](concurrency/race_condition.md)
   2. [shared_ptr](concurrency/shared_ptr.md)
   3. [thread::join](concurrency/jthread.md)
   4. [Повторный захват mutex](concurrency/double_lock.md)
   5. [Signal-unsafe](concurrency/signal_unsafe.md)
   6. [condition_variable](concurrency/condition_variable.md)


---
## Помощь

В тексте могут быть ошибки, опечатки, неточности, он может устаревать. Пожелания, предложения и замечания приветствуются: можно завести `issue` или сделать `pull request`.

## И еще кое-что

Бегать за вами и судиться автор сего сборника не будет, но все-таки:

На этот проект **можно** ссылаться. Можно приводить примеры из него, со ссылками, конечно же.

Для копирования и иного воспроизведения **надо** получить согласие автора

**Нельзя** использовать в платных сервисах или взимать плату за обучение по этим материалам.

#### Ну и самое последнее примечание

Черновое название этой работы, "Ружье достаточной огневой мощи, чтобы на нем повеситься", как могли догадаться искушенные читатели, было эдаким реверансом в сторону известного (но очень плохо состарившегося) сборника по C++ "Веревка достаточной длины, чтобы выстрелить себе в ногу" от Алана Голуба. Но, к сожалению, мы живем в нежном мире победивших алгоритмов ранжирования и надзорных органов, то и дело стремящихся кого-нибудь от чего-нибудь защитить. 

Автор, конечно, очень бы хотел защитить всех от C++, и именно этому и служит данных сборник, но с заблокированным и пессимизированным репозиторием прогресса в этом направлении не будет.


_Copyright 2020-2024 Dmitry Sviridkin_