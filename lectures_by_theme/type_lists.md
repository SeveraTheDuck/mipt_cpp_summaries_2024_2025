# Type lists

## Определение

**Type list** - это список типов. Да, мы просто перевели название.

Они работают из-за ленивого инстанцирования шаблонов. Структуры, которые мы не
используем, не инстанцируются.

## Реализация

Рассмотрим несколько простых примеров.

### Starred

Starred - на n-ой итерации вернёт n-ый указатель на T:

```cpp
template <typename T>
struct Starred {
  using Head = T*;
  using Tail = Starred<T*>;
};

static_assert(std::is_same_v<
  typename Starred<int>::Tail::Tail::Head,
  int***
>);
```

Если мы хотим ограничить рост TypeList, заведём пустой список:

struct Nil {};

struct EmptyTypeList : Nil {};

### Обработчики TypeList

Для конкатенации листов можно делать вот так:

```cpp
template <typename T, typename TL>
struct Concat {
  using Result = TypeList<T, TL>
};
```

Можно создать TypeList из TypeTuple:

```cpp
template <typename... T>
struct TypeTuple {};

template <typename TT> // TypeTuple
struct FromTuple;

template <typename Head, typename... Tail>
struct FromTuple<TypleTuple<Head, Tail...>> {
  using Result = TypeList<Head, typename FromTuple<TypeTuple<Tail...>>::Result>;
};

template <typename HeadT>
struct FromType<TypeTuple<HeadT>> {
  using Result = TypeList<HeadT, Nil>;
};

static_assert(std::is_same_v<
  typename FromTuple<TypeTuple<int, double, int>>::Result,
  Concat<int,
    typename Concat<double>,
      typename Concat<int, Nil>::Result
    >::Result
  >::Result
>);
```

Хотим получить тип из TypeList по индексу:

```cpp
template <typename TL, std::size_t I>
struct GetImpl {
  using Result = GetImpl<typename TL::Tail, I - 1>::Result;
};

template <typename TL>
struct GetImpl<TL, 0> {
  using Result = TL::Head;
};

template <std::size_t I>
struct GetImpl<Nil, I> {
  using Result = Nil;
};

template <>
struct GetImpl<Nil, 0> {
  using Result = Nil;
};

template <std::size_t I, typename TL>
using Get = GetImpl<TL, I>::Result;
```

Теперь хотим уметь заменять первое вхождение типа T на тип U:

```cpp
template <typename T, typename U, typename TL>
struct ReplaceImpl {
  using Result = TypeList<typename TL::Head, typename ReplaceImpl<T, U, typename TL::Tail>;
};

template <typename T, typename U, typename TLTail>
struct ReplaceImpl<T, U, TypeList<T, TLTail>> {
  using Result
};

template <typename T, typename U, typename TL>
using Replace = ReplaceImpl<T, U, TL>::Result;
```
