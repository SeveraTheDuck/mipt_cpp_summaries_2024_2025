# Type deduction and `auto` <!-- omit from toc -->

## `auto`

### Мотивация

Программисты - ленивые люди. Мы не хотим писать дофига длинных слов, когда это
можно не делать.

Пусть есть вектор векторов отрезков. Разраб гений, так что отрезок - пара из
двух пар `int`:

```cpp
std::vector<std::vector<std::pair<std::pair<int, int>, std::pair<int, int>>>>
    vec {{ {{1, 2}, {3, 4}} }};

// писать цикл вот так - извращение
for (std::vector<std::vector<std::pair<std::pair<int, int>,
                             std::pair<int, int>>>>::iterator
     it = vec.begin(), end = vec.end(); it != end; ++it) {
  // тело цикла
}
```

### Решение с C++11: `auto`

Теперь у нас есть `auto`:

```cpp
for (auto it = vec.begin(), end = vec.end(); it != end; ++it) { }
```

Он сам выводит тип объекта в зависимости от выражения, стоящего справа.

## Особенности вывода типа `auto`

Рассмотрим примеры вывода типа, который делает `auto`:

```cpp
int x = 10;
const int y = 20;
const int& z = 30;
int& r = x;

// Здесь везде auto выведется как int, отбрасывая const и ref
auto a = 10;
auto ax = x;
auto ay = y;
auto az = z;
auto ar = r;
```

Что нам это напоминает? Вывод типа в шаблонах:

```cpp
template <typename T>
void foo(T obj) {}

// Тут везде T выведется как int
foo(10);
foo(x);
foo(y);
foo(z);
foo(r);
```

Если к 'auto 'добавим `&` или `&&`, то правила немного поменяются, как и в
шаблонах:

```cpp
template <typename T>
void foo(T& obj) {}

// Тут везде вывод типа как int
// auto& aref = 10; Вывод как int, CE из-за lvalue сслыки на rvalue
auto& axr = x;
auto& ayr = y;
auto& azr = z;
auto& arr = r;

// foo(10); T выведется как int, CE из-за lvalue сслыки на rvalue
foo(x);
foo(y);
foo(z);
foo(r);
```

У `auto` такие же правила работы с массивами и указателями на функции, что и в
шаблонах. Мне лень это писать.

Существует исключение из этого правила конкретно для переменных -
`std::initializer_list<T>`.

```cpp
template <typename T>
void bar(T x) {}

// bar({1, 2, 3}); CE, шаблоны не умеют в initializer_list
auto = {1, 2, 3}; // а вот auto умеет
```

## Лирическое отступление: `std::initializer_list`

### Что такое `std::initializer_list`?

`std::initializer_list<T>` - штука, реализованная на уровне языка. Она позволяет
передавать набор объектов одним другим объектом, по которому можно ещё и
итерироваться:

```cpp
void bar(std::initializer_list<int> il) {
  for (auto elem : il) {
    std::cout << elem << '\n';
  }
}

bar({1, 2, 3});
```

### Проблемы `std::initializer_list`

Эта штука вносит некоторые проблемы в конструкторах:

```cpp
template <typename T>
struct S {
  S() = default;
  S(int) {}
  S(double, double) {}
  S(std::initializer_list<int>) {}
};

S s;            // default
S s1(10);       // int
S s2{10};       // initialize_list
S s3{10, 10};   // initialize_list
S s4(3.5, 4.5); // double, double
S s5{2.5, 3.5}; // initialize_list, CE in double->int conversion
S s6{s};        // copy constructor wtf
S s7{};         // default constructor
```

Любой конструктор с фигурными скобками, кроме дефолтного конструктора и (к
сожалению) копирующего конструктора, становится конструктором от
`initializer_list`.

## `auto` в возвращаемом типе функций (С++14)

Можно использовать `auto` для возвращаемого типа из функции. Но он не будет
думать, что именно ему возвращать, если в разных `return` разные типы:

```cpp
// это решение не сработает, т.к. в двух return разные типы
// auto заморачиваться не станет, просто CE
template <typename T, typename U>
auto get_min(const T& lhs, const U& rhs) {
  if (lhs < rhs) {
    return lhs;
  }
  return rhs;
}

// а вот это сработает, т.к. тернарный оператор сам ищет общий тип
template <typename T, typename U>
auto get_min(const T& lhs, const U& rhs) {
  return lhs < rhs ? lhs : rhs;
}
```

Ясно, что можно навешивать на возвращаемый тип ссылки, `const` и т.д.

В этом контексте `auto` работает буквально по тем же правилам, что и шаблоны.

## `auto` в типе аргументов функций (С++17)

Часто нам не нужно лезть внутрь шаблонного аргумента передаваемого в функцию
другого аргумента или в целом использовать какие-то трёхэтажные конструкции на
шаблонах, как тут:

```cpp
// тут мы смотрим на внутренний тип вектора
// это делается только шаблонами
template <typename T>
void foo(std::vector<T> vec) {}
```

Если таких сложностей не надо, удобно использовать `auto` для аргументов
функции:

```cpp
void foo(auto x) {}

// это БУКВАЛЬНО то же самое, что и:
template <typename T>
void foo(T x) {}
// только нельзя имя типа использовать
```

## `auto` в шаблонных аргументах (С++20)

Теперь можно вот так:

```cpp
template <auto T>
void foo(T x) {}
```

То есть можно прокидывать какие-то литералы и из них выводить тип шаблонного
аргумента функции. Это позволяет ещё сильнее обобщить код.

## Range-based for

### Потерянные квалификаторы

Мы уже проходили range-based for:

```cpp
std::map<int, double> map{{1. 2}};

// тут просто длинно писать, но в целом всё ок, так можно
// обратите внимание на const в первом типе пары
for (const std::pair<const int, double>& pair : map) {
  std::cout << pair.first << ' ' << pair.second << '\n';
}
```

Однако в контейнерах велик шанс продолбаться в `const` и вызвать ненужные и
очень неожиданные копирования:

```cpp
for (const std::pair</*забыли const*/ int, double>& pair : map) {
  // у нас создалась временная пара вместо ссылки на уже существующую
  // на этом моменте у нас UB из-за const_cast, а ещё мы модифицируем
  // временный объект, а не пару в map
  const_cast<std::pair<int, double>&>(pair).second = 1000;
  std::cout << pair.first << ' ' << pair.second << '\n';
}
```

Из-за этого рекомендуется в range-based for использовать `auto`:

```cpp
// все const внутри типа объекта подставятся сами, не надо за ними следить
for (const auto& pair : map) { ... }
```

### Создание объектов в range-based for

Сейчас (с С++20 по стандарту) можно создавать объекты в range-based for.
Компиляторы это вводили даже раньше из-за бага с ссылками на внутренние объекты
rvalue объектов:

```cpp
// тут всё ок
std::vector<int> bar() {
  return {1, 2, 3};
}

// получили временный объект и смотрим на элементы внутри него
for (auto elem : bar()) {
  std::cout << elem << '\n';
}

// а вот тут всё плохо
std::vector<std::vector<int>> foo() {
  return {{1, 2, 3}};
}

// получили временный объект, но сохранили ссылку не на него, а на внутренний
// другой объект
for (auto elem : foo().front()) {
  std::cout << elem << '\n';
}
```

Проблема в том, что временный объект, который возвращается из `foo()`, умирает,
т.к. на него нет ссылок. Из-за этого вызывается деструктор и, как следствие,
умирает и внутренний вектор, по которому мы пытаемся итерироваться.

Компиляторы предложили решение раньше, чем его ввели в стандарт, но сейчас так
можно и по стандарту:

```cpp
for (auto vec = foo().front(); auto elem : vec) {
  std::cout << elem << '\n';
}
```

## Тонкости `auto`

Иногда использование `auto` может вызвать проблемы, когда мы забыли об
устройстве объекта. Например, вектор `bool` устроен по-особенному:

```cpp
std::vector<bool> vec{true, false};
auto elem = vec[0]; // это не bool, а зависящий от компилятора bit_reference
```

Прикол в том, что `std::vector<bool>` возвращает не `bool`, а некий
`bit_reference`, чтобы сжимать данные и значение `bool` занимало не 1 байт, а 1
бит.

## `decltype`

Допустим, мы хотим вывести тип, учитывая его квалификаторы константности, ссылки
и т.д.

В этом нам помогает `decltype`:

```cpp
int x = 0;
int& y = x;

auto z = y; // тут z это просто int
decltype(y) dz = x; // а тут dz - это int&
x += 1000;
std::cout << z << '\n'; // вывели 1000
```

### Разновидности `decltype`

1. `decltype(auto)` - про него позже.
2. `decltype(name)` - когда мы передаём объект. Это в точности тип name,
  ничего странного.
3. `decltype(expr)` - когда мы передаём выражение. Возвращаемый тип зависит ещё
  и от expression category (lvalue или rvalue):

```cpp
int x = 0;

decltype(x) a = 1000;    // int a = 1000;
decltype(x += 10) b = x; // int& b = x;
x = 123;
std::cout << b << '\n';
```

## Expression category

Введём честную иерархию категорий выражений (нужно для `decltype(expression)`)

Две больших категории:

- glvalue (generalized) - обладает идентичностью
- rvalue - moveable

От них наследуются следующие, с которыми мы работаем:

- glvalue:
  - lvalue (pure) - всё, что возвращает `T&`
- rvalue:
  - prvalue (pure rvalue) - всё, что возвращает `T`
- и glvalue, и rvalue:
  - xvalue (expired) - `T&&` - что-то, что есть в памяти, но состояние можно
  забрать

Как с ними работает `decltype`?

```cpp
int x = 0;
decltype(x += 10);      // T&
decltype(x + 10);       // T
decltype(std::move(x)); // T&&
```

Ещё прикол в том, что для полей класса `decltype(s.x);` является `decltype` от
имени, не от expression.

## `decltype` в возвращаемом типе функций

Допустим, у нас есть какой-то код для контейнера, мы хотим получить элемент по
индексу. Следующий код не сработает, мы вернём копию:

```cpp
template <typename C>
auto get(C& cont, std::size_t i) {
  return cont[i];
}

std::deque<int> deq{1, 2, 3};
const int& val = get(deq, 0);
deq[0] = 1000;
std::cout << val << '\n'; // 1, значение не поменялось
```

Хотим как-то использовать `decltype`. Неудачный способ:

```cpp
// это не компилится, point of declaration cont и i позже
template <typename C>
decltype(cont[i]) get(C& cont, std::size_t i) {
  return cont[i];
}
```

Рассмотрим, что можно сделать, чтобы с этим можно было работать.

### Trailing return type

Сделали вот такой способ писать возвращаемый тип функций:

```cpp
// это действительно вернёт ссылку на cont[i]
template <typename C>
auto get(C& cont, std::size_t i) -> decltype(cont[i]) {
  return cont[i];
}
```

То есть сначала пишется auto, а уже после аргументов - стрелочка и возвращаемый
тип.

Так мы можем использовать аргументы функции в `decltype`, т.к. их point of
declaration уже прошла.

### `decltype(auto)`, начиная с С++14

Если мы не хотим заморачиваться с выводом типа, но как-то хотим вернуть ссылку,
можно использовать `decltype(auto)`:

```cpp
// тоже вернёт ссылку на cont[i]
template <typename C>
decltype(auto) get(C& cont, std::size_t i) {
  return cont[i];
}

decltype(auto) y = (x += 3); // y - ссылка на x
```

Это очень мощная штука, но она прямо кричит о том, что ссылка может быть какая
угодно (или не быть вообще), и надо вчитываться в код, чтобы понять, что
происходит.

Совет: если нам без разницы, использовать `auto` или `decltype(auto)`, то лучше
использовать `auto`.

## Struct binding (C++17)

```cpp
std::pair<int, double> val = {1, 2};
auto p = val.first; // не хочется постоянно писать .first и .second

// ввели вот такую красоту
auto [key, value] = val;
std::cout << key << ' ' << value << '\n';
```

Наконец-то можно нормально итерироваться по `std::map`:

```cpp
std::map<int, std::string> map {
  {1, "One"},
  {2, "Two"},
  {3, "Three"}
};

// вот это уже реально красивый range-based for
for (const auto& [key, value] : map) {
  std::cout << key << ' ' << value << '\n';
}

// с C++26 также есть возможность игнорировать объекты при такой распаковке
// с помощью символа _
for (const auto& [_, value] : map) {
  std::cout << value << '\n';
}
```

Эта штука также работает для публичных полей структуры:

```cpp
struct S {
  int x = 0;
  int y = 1;
};

S s;
auto [x, y] = S;
```

Распаковка игнорирует приватные поля, раскрывая доступ только к публичным.
