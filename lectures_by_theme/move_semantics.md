# Move semantics <!-- omit from toc -->

- [Основная идея move semantics](#основная-идея-move-semantics)
  - [Мотивация](#мотивация)
  - [C++11: Move semantics](#c11-move-semantics)
- [Большой пример: `String`](#большой-пример-string)
- [Подробнее про rvalue references](#подробнее-про-rvalue-references)
  - [Lvalue refrences](#lvalue-refrences)
  - [Rvalue references на временные объекты](#rvalue-references-на-временные-объекты)
  - [Памятка: что lvalue, а что rvalue](#памятка-что-lvalue-а-что-rvalue)
- [Зачем нужны rvalue references на практике?](#зачем-нужны-rvalue-references-на-практике)
  - [Про конструкторы](#про-конструкторы)
- [Первичное устройство std::move](#первичное-устройство-stdmove)
- [Rule of five](#rule-of-five)
  - [Правила генерации move конструктора и assignment](#правила-генерации-move-конструктора-и-assignment)
- [Универсальные ссылки](#универсальные-ссылки)
- [`std::forward`](#stdforward)
  - [Мотивация `std::forward`](#мотивация-stdforward)
  - [Идея решения](#идея-решения)
  - [Реализация `std::move` и `std::forward`](#реализация-stdmove-и-stdforward)
- [Практический пример: `Allocator::construct()`](#практический-пример-allocatorconstruct)

## Основная идея move semantics

### Мотивация

Представим ситуацию, когда есть структура:

```cpp
struct S {
  std::vector<int> vec;
};
```

Мы создали вектор, чтобы передать его в структуру, но дальше он нам оказался
не нужен.

```cpp
vector<int> v;
v.push_back(1);
... // тут напушили кучу всего
S s;
s.vec = v;
// дальше v не используется
```

Проблема: вектор полностью копируется, хотя сам объект больше не нужен. Хотелось
бы уметь как-то передавать владение ресурсами объекта.

До С++11 в основном костылили.

### C++11: Move semantics

Ввели rvalue reference:

```cpp
class C {
  C() = default;

  C(C&& c) { // здесь && означает rvalue reference на c
    ptr = c.ptr;
    c.ptr = nullptr;
  }

  int* ptr = new int[10];
};
```

Такая запись означает, что мы в конструкторе можем забрать ресурсы объекта.

Чтобы привести к этому && и вызвать такой конструктор, пишем (пока волшебный)
`std::move`:

```cpp
C c1;
// c2 забирает поля c1 за O(1), не выделяя память лишний раз
C c2 = std::move(c1);
```

Похожую вещь можно сделать и для `operator=`, не только конструктора. И ещё для
кучи вещей.

## Большой пример: `String`

```cpp
struct String {
  String() = default;

  ~String() {
    delete[] data_;
  }

  // Copy constructor
  String(const String& other)
      : size_(other.size_)
      , cap_(other.cap_),
      , data_(new char[cap_]) {
    for(std::size_t i = 0; i < size_; ++i) {
      data_[i] = other.data_[i];
    }
  }

  // Move constructor
  String(String&& other) {
    // забрали поля у другого объекта
    data_ = other.data_;
    size_ = other.size_;
    cap_ = other.cap_;

    // похерили поля у other, чтобы не вызвать каких-то неожиданных приколов
    other.data_ = nullptr;
    other.size_ = 0;
    other.cap_ = 0;
  }

  std::size_t size_ = 0;
  std::size_t cap_ = 0;
  char* data_ = nullptr;
};
```

## Подробнее про rvalue references

Что мы умели раньше (до С++11):

### Lvalue refrences

Была ссылка на переменные:

```cpp
int x = 3;
int& y = x;
```

Также был костыль с константными ссылками и временными объектами (продление
жизни):

```cpp
const int& z = 3;
```

### Rvalue references на временные объекты

Для временных объектов в С++11 ввели новый синтаксис:

```cpp
int&& t = 4;
const int&& ct = 10;
```

### Памятка: что lvalue, а что rvalue

|lvalue           | rvalue           |
| --------------- | ---------------- |
| `x;`            | `3; 2.5;`        |
| `"abc"`         | `'a';`           |
| `x += ...`      | `x + ...`        |
| func returns T& | func returns T   |
|                 | func returns T&& |

Последние 2 строки аналогично работают для кастов.

## Зачем нужны rvalue references на практике?

Заводить ссылки на временные инты - скучно. На практике rvalue references
используется для перегрузок:

```cpp
int foo(int& x) { return 0; }
int foo(const int& x) { return 1; }
int foo(int&& x) { return 2; } // эта перегрузка забирает все временные объекты
int foo(const int&& x) { return 3; }
int glob = 10;

int bar() { return 0; }
int& baz() { return glob; }

int main() {
  int x = 0;
  const int y = 0;
  const int&& z = 0;

  foo(x);     // 0
  foo(10);    // 2
  foo(bar()); // 2
  foo(baz()); // 0
  foo(y);     // 1

  foo(z);     // 1 hehehe
  foo(std::move(z)); // 3
  foo(z + 3); // 2
}
```

Три последних вызова `foo()` требуют пояснения:

- тип z - `const int&&`, но expression category - lvalue.
- тип передаваемого объекта - `const int&&`, но expression category - rvalue.
- тип - `int`, expression category - rvalue.

То есть по сути нужно смотреть на expression category для понимания, в какую
перегрузку по & или && мы попадём.

### Про конструкторы

Раньше (с С++11 до С++17) тут везде был вызов move конструктора:

```cpp
C(C&& other) { ... }

C c1 = C(3, 2);
C c2 = std::move(c1);
```

Начиная с C++17, некоторые вещи изменились в лучшую сторону, но об этом потом.

## Первичное устройство std::move

Приведём пару начальных версий и их проблемы:

```cpp
// тут проблема в копировании, дорого
template <typename T>
T my_move(T& value) {
  return value;
}

// static_cast умеет приводить lvalue к rvalue
// нюанс: не сработает C c1 = my_move(C());
template <typename T>
T&& my_move(T& value) {
  return static_cast<T&&>(value);
}
```

Мы ещё допишем эту функцию, чтобы всё работало, но чуть позже. Основная суть не
поменяется - move это просто крутой `static_cast`.

## Rule of five

Теперь, помимо деструктора, копирующего конструктора и присваивания, у нас есть:

```cpp
class C {
  // [cv] - const и volatile в любой комбинации
  C([cv] C&& other); // move конструктор
  C& operator=([cv] C&& other); // move assignment
};

C c1;
C c = c1; // copy constructor
c = c1;   // copy assignment
C c = std::move(c1); // move constructor
c1 = std::move(c);   // move assignment
```

### Правила генерации move конструктора и assignment

Обозначения:

- X_c - copy constructor
- Y_c - copy assignment
- X_m - move constructor
- Y_m - move assignment

Сами правила:

1. Если определили X_c или Y_c, не генерируются автоматически X_m и Y_m.
2. Если определили X_m, не генерируются автоматически X_c, Y_c, Y_m.
3. Если определили Y_m, не генерируются автоматически X_c, Y_c, X_m.
4. Если определили деструктор, не генерируются автоматически X_m, Y_m.

## Универсальные ссылки

Введём новую сущность - `T&&`. Прикол: это не всегда rvalue reference.

```cpp
template <typename T>
foo(T&& x) { ... }
```

Это, фактически, костыль в языке. Называется forwarding reference. Она может
принять буквально **что угодно**:

```cpp
int x = 0;
const int y = x;
int&& t = 5;

// Здесь запись T -> *вывод Т* ; *итоговый принимаемый тип foo*
foo(x);   // T -> int& ; int& && -> int&
foo(y);   // T -> const int& ; const int& && -> const int&
foo(t);   // T -> int& ; int& && -> int&
foo(10);  // T -> int ; int && -> int&&
```

Здесь сработали т.н. правила склеивания ссылок (reference collapsing):

- &  + &  = &&
- &  + && = &
- && + &  = &
- && + && = &&

## `std::forward`

### Мотивация `std::forward`

Представим ситуацию:

```cpp
void foo(int&&) { ... }
void foo(int&) { ... }

template <typename T>
void bar(T t) {
  f(t); // вызовется foo(int&)
}

int x = 0;
bar(std::move(x));
```

В коде выше странность: мы муваем x, но при этом вызывается версия `foo(int&)`,
хотя из контекста было бы логичнее вызывать `foo(int&&)`.

### Идея решения

Мы хотим передавать универсальную ссылку и каким-то образом определять, к какому
типу её приводить при передаче (forwarding, отсюда название ссылок) дальше.

То есть нам хотелось бы иметь какое-то приведение типа или if:

```cpp
template <typename T>
void bar(T&& t) {
  // псевдокод
  if (T = int&) foo(int&);
  else foo(int&&);
}
```

Этот if - `std::forward`:

```cpp
template <typename T>
void bar(T&& t) {
  f(std::forward<T>(t)); // приводит t к нужному типу, в зависимости от вывода T
}
```

### Реализация `std::move` и `std::forward`

Допишем наш старый `std::move`, чтобы он умел принимать rvalue expression
аргументы:

```cpp
template <typename T>
[[nodiscard]] std::remove_reference_t<T>&& move(T&& value) {
  return static_cast<std::remove_reference_t<T>&&>(value);
}
```

Насколько я помню, в реализацию выше стоит ещё тыкнуть `constexpr` и `noexcept`.

А теперь напишем `std::forward`:

```cpp
// Вот это пока принимает только lvalue references...
template <typename T>
T&& forward(std::remove_reference_t<T>& value) {
  return static_cast<T&&>(value);
}

// ...так что делаем перегрузку для rvalue references
template <typename T>
T&& forward(std::remove_reference_t<T>&& value) {
  return static_cast<T&&>(value);
}
```

## Практический пример: `Allocator::construct()`

Теперь мы умеем по-умному конструировать объекты с помощью аллокатора:

```cpp
template <typename T>
struct Allocator {
  // Это не работает так, как мы хотим
  // T&& здесь не универсальная ссылка, т.к. это не шаблон функции,
  // и вывода типов для неё нет
  void construct(T* ptr, T&&) { ... }

  // Правильно:
  template <typename... Args>
  void construct(T* ptr, Args&&... args) {
    new(ptr) T(std::forward<Args>(args)...);
  }
};
```
