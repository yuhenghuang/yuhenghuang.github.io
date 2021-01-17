---
title: Tuple in Clang
top: false
cover: false
toc: true
mathjax: false
date: 2021-01-17 16:16:41
password:
summary:
tags:
  - Data Structure
categories: cpp
---

`tuple` is a data structure referring to a container than holds data of different types. Its implementation is very straight forward for `Java` and `python` as those languages basically cope with objects that inherit a common class, `Object`, and all objects are saved in the heap, referred by address in function calls.

However, for languages like `cpp` with strict typing, the implementation is much more difficult. The principle is to use template. But how the data structure should look like remains unclear.

In the undergraduate I studied the implementation of `tuple` in *GNUC* version of `cpp`.

Recently I started using macos and thus clang became the complier with its *STL* libraries. That was the time when I found the implementation of `tuple` in *clang* is totally different from than in *GNUC*.

Eventually it took me several days to extract the core structure for the purpose of study and curiosity. Here in the post I'd like to summarize and record the findings.


### `tuple` in *GNUC*

In *GNUC*, `tuple` in implemented recursively.


```cpp

// where data live
template <size_t Idx, typename Head> struct Head_base;

// declaration
// this version is never used for the feature that 
// template ALWAYS choose the one most specific
template <size_t Idx, typename... Elements> struct tuple_impl;

// recursively construct Head_base
template <size_t Idx, typename Head, typename... Tail>
struct tuple_impl<Idx, Head, Tail...>
  : public tuple_impl<Idx+1, Tail...>,
    private Head_base<Idx, Head> 
{ 
  typedef tuple_impl<Idx+1, Tail...> Inherited;
  typedef Head_base<Idx, Head> Base;
};

// end condition of recursion
template <size_t Idx, typename Head>
struct tuple_impl<Idx, Head>
  : private Head_base<Idx, Head> 
{
  typedef Head_base<Idx, Head> Base;
};


template <typename... Elements>
class tuple : public tuple_impl<0, Elements...>
{
  typedef tuple_impl<0, Elements...> Inherited;
};

// specialization
template <> class tuple<> { };

```

In this implementation, every `Head_base` is actually a base class of `tuple`. Thus it can be accessed by type casting fairly easily.



### `tuple` in *clang*

`tuple` is implemented quite different. It involves many helper functions/classes and the structure is more like a n-ary tree.

In the post some implementations of helpers are different from *clang*. This is because some *type traits* utilise compiler-specific built-in functions or classes in `type traits` header. For the test purpose on all platform, those compiler-specific functions are rewritten by me.

Most of those original *type traits* were implemented under the principle of **SFINAE**.

core structures are

```cpp
template <size_t... Is> struct tuple_indices { };

// where data live
template <size_t I, typename Tp> struct tuple_leaf;

template <typename Is, typename... Types>
struct tuple_impl;

// inherit all leaves at the same time
template <size_t... Is, typename... Types>
struct tuple_impl<tuple_indices<Is...>, Types...>
  : public tuple_leaf<Is, Types>... { };


// make_tuple_indices<N>::type is
// tuple_indices<0, ..., N-1>
template <typename... Types>
class tuple {
  typedef tuple_impl<typename make_tuple_indices<sizeof...(Types)>::type, Types...> BaseType;

  protected:
    BaseType base;
};

template <> class tuple<> { };
```

Let's start from helper classes.

#### Declarations

```cpp
template <typename... Types> class tuple;

template <typename... Types> struct tuple_types;

template <size_t I, typename Tp> struct tuple_element;

template <size_t I, class... Tp> inline
typename tuple_element<I, tuple<Tp...>>::type& get(tuple<Tp...>&);

template <size_t I, class... Tp> inline
const typename tuple_element<I, tuple<Tp...>>::type& get(const tuple<Tp...>&);

template <size_t I, class... Tp> inline
typename tuple_element<I, tuple<Tp...>>::type&& get(tuple<Tp...>&&);

template <size_t I, class... Tp> inline
const typename tuple_element<I, tuple<Tp...>>::type&& get(const tuple<Tp...>&&);
```


#### `tuple_indices`

The class that contains `size_t` indices in template parameters.

`make_tuple_indices<N, L=0>` defines `tuple_indices<L, ..., N-1>`

```cpp
template <size_t... Is>
struct tuple_indices { };


// never used declaration
template <size_t... Is>
struct make_tuple_indices_impl;

// end condition
template <size_t L, size_t... Is>
struct make_tuple_indices_impl<L, L, Is...> {
  typedef tuple_indices<Is...> type;
};

// expand recursively
template <size_t L, size_t N, size_t... Is>
struct make_tuple_indices_impl<L, N, Is...> 
  : public make_tuple_indices_impl<L, N-1, N-1, Is...> { };

template <size_t N, size_t L=0>
struct make_tuple_indices
  : public make_tuple_indices_impl<L, N>{ };
```

#### `type_pack_element`

This is a clang built-in function which extracts i-th type in a pack of types.

Recursively implemented.

```cpp
template <size_t I, typename... Types>
struct type_pack_element;

template <typename Head, typename... Tails>
struct type_pack_element<0, Head, Tails...> {
  typedef Head type;
};

template <size_t I, typename Head, typename... Tails>
struct type_pack_element<I, Head, Tails...>
  : public type_pack_element<I-1, Tails...>{ };
```


#### `tuple_size`

Count how many elements are there in a `tuple`.

```cpp
// implement integral_constant
template <typename Tp, Tp v>
struct integral_constant {
  static constexpr const Tp value = v;
};

// in recent version or higher (c++14)
// using syntax is more convenient
/*
template <size_t N>
using index_constant = integral_constant<size_t, N>;
*/
template <size_t N>
struct index_constant
  : public integral_constant<size_t, N>{ };


template <typename Tuple> struct tuple_size;

template <typename... Types>
struct tuple_size<tuple<Types...>> 
  : public index_constant<sizeof...(Types)> { };

template <typename... Types>
struct tuple_size<tuple_types<Types...>>
  : public index_constant<sizeof...(Types)> { };

// specialization that works for const and/or volatile types
template <class Tuple>
struct tuple_size<const Tuple>
  : public tuple_size<Tuple> { };

template <class Tuple>
struct tuple_size<volatile Tuple>
  : public tuple_size<Tuple> { };

template <class Tuple>
struct tuple_size<const volatile Tuple>
  : public tuple_size<Tuple> { };
```

#### `is_same` and `all`

* `is_same<Tp0, Tp1>` predicates whether `Tp0==Tp1`
* `all<bool...>` predicates if all `bool`s are true


```cpp
typedef integral_constant<bool, true> true_type;
typedef integral_constant<bool, false> false_type;


template <class Tp, class Up>
struct is_same
  : public false_type { };

template <class Tp>
struct is_same<Tp, Tp>
  : public true_type { };


template <bool... Preds>
struct all_dummy { };

// logical all
template <bool... Preds>
struct all
  : public is_same<all_dummy<Preds...>, all_dummy<((void)Preds, true)...>> { };
```

#### `apply_cv_t`

This class copes with the feature that data inside `tuple` should *inherit* `const`, `volatile` and `reference` statuses from outside.

The usage will be exhibited later.

```cpp
template <bool ApplyLV, bool ApplyConst, bool ApplyVolatile>
struct apply_cv_mf;

template <>
struct apply_cv_mf<false, false, false> {
  template <class T> using apply = T;
};

template <>
struct apply_cv_mf<true, false, false> {
  template <class T> using apply = T&;
};

template <>
struct apply_cv_mf<true, true, false> {
  template <class T> using apply = const T&;
};

template <>
struct apply_cv_mf<true, false, true> {
  template <class T> using apply = volatile T&;
};

template <>
struct apply_cv_mf<true, true, true> {
  template <class T> using apply = const volatile T&;
};


template <>
struct apply_cv_mf<false, true, false> {
  template <class T> using apply = const T;
};

template <>
struct apply_cv_mf<false, false, true> {
  template <class T> using apply = volatile T;
};

template <>
struct apply_cv_mf<false, true, true> {
  template <class T> using apply = const volatile T;
};


template <class T, class Raw = typename std::remove_reference<T>::type>
using apply_cv_t = apply_cv_mf<
                     std::is_lvalue_reference<T>::value,
                     std::is_const<Raw>::value,
                     std::is_volatile<Raw>::value
                   >;
```


#### `tuple_types`

Similar to `tuple_indices`, though it holds types in the template parameters.

One may find that `make_tuple_types` constructs types after applying `const`, `volatile` and `reference` of `tuple`.


```cpp
template <typename... Types>
struct tuple_types { };


template <typename Tp, typename Is>
struct make_tuple_types_flat;

template <typename... Types, size_t... Is>
struct make_tuple_types_flat<tuple<Types...>, tuple_indices<Is...>> {
  template <class Tuple, class ApplyFn = apply_cv_t<Tuple>>
  using apply_quals = tuple_types<
    typename ApplyFn::template apply<
      typename type_pack_element<Is, Types...>::type
    >...
  >;
};

/* 
  make_tuple_type
    apply const volatile and lvalue reference to types in tuple!

    applying lvalue reference to avoid unexpected move construction and assignment of non-reference members.
*/

template <typename Tp, size_t N = tuple_size<typename std::remove_reference<Tp>::type>::value, 
          size_t L = 0, 
          bool Total = (N == tuple_size<typename std::remove_reference<Tp>::type>::value)>
struct make_tuple_types {
  static_assert(L <= N, "make_tuple input error");
  typedef typename std::remove_cv<typename std::remove_reference<Tp>::type>::type Raw;
  typedef make_tuple_types_flat<Raw, typename make_tuple_indices<N, L>::type> Maker;
  using type = typename Maker::template apply_quals<Tp>;
};


// specialization for efficiency
template <typename... Types, size_t N>
struct make_tuple_types<tuple<Types...>, N, 0, true> {
  typedef tuple_types<Types...> type;
};


template <typename... Types, size_t N>
struct make_tuple_types<tuple_types<Types...>, N, 0, true> {
  typedef tuple_types<Types...> type;
};
```


#### `tuple_like`

Whether a type is `tuple` or `tuple_types`.

```cpp
template <class Tp>
struct tuple_like
  : public false_type { };

// strip const and volatile
template <class Tp>
struct tuple_like<const Tp>
  : public tuple_like<Tp> { };

template <class Tp>
struct tuple_like<volatile Tp>
  : public tuple_like<Tp> { };

template <class Tp>
struct tuple_like<const volatile Tp>
  : public tuple_like<Tp> { };


// true specialization
template <class... Types>
struct tuple_like<tuple<Types...>>
  : public true_type { };

template <class... Types>
struct tuple_like<tuple_types<Types...>>
  : public true_type { };
```


#### `tuple_element`

Extract i-th type of tuple and apply `const` and `volatile` if necessary.

```cpp
template <size_t I, typename Tp>
struct tuple_element;

template <size_t I, typename... Types>
struct tuple_element<I, tuple_types<Types...>> {
  static_assert(I < sizeof...(Types), "index out of bound");
  typedef typename type_pack_element<I, Types...>::type type;
};

template <size_t I, typename... Types>
struct tuple_element<I, tuple<Types...>> {
  typedef typename tuple_element<I, tuple_types<Types...>>::type type;
};

// apply const and volatile
template <size_t I, typename Tuple>
struct tuple_element<I, const Tuple> { 
  typedef const typename tuple_element<I, Tuple>::type type;
};

template <size_t I, typename Tuple>
struct tuple_element<I, volatile Tuple> {
  typedef volatile typename tuple_element<I, Tuple>::type type;
};

template <size_t I, typename Tuple>
struct tuple_element<I, const volatile Tuple> { 
  typedef const volatile typename tuple_element<I, Tuple>::type type;
};
```


#### `tuple_leaf`

Where the data is stored.

To be more specific, for empty and non-final type the implementation is different. That specialization is skipped in the post as it is the minor issue.

```cpp
template <size_t I, typename Tp>
struct tuple_leaf {
  protected:
    Tp data;

  public:
    tuple_leaf() = default;

    template <typename Up>
    tuple_leaf(Up&& val): 
      data(std::forward<Up>(val)) { }

    tuple_leaf(const tuple_leaf&) = default;
    tuple_leaf(tuple_leaf&&) = default;

    template <typename Up>
    tuple_leaf& operator=(Up&& t) {
      data = std::forward<Up>(t);
      return *this;
    }

    Tp& get() { return data; }
    const Tp& get() const { return data; }
};
```


#### `tuple_impl`

The implementation class of `tuple_impl`. Details of type traits for exceptions are deleted.

* `tuple_leaf`s are not inherited recursively but simultaneously.
* Constructor supports only initializing first *n* data given value.


```cpp
// for compatibility before c++17
template <typename... Types>
void swallow(Types&& ...) {}


template <typename Is, typename... Types>
struct tuple_impl;

template <size_t... Is, typename... Types>
struct tuple_impl<tuple_indices<Is...>, Types...>
  : public tuple_leaf<Is, Types>... {

  public:
    tuple_impl() = default;

    // d : default construction
    // p : passing parameters
    template <size_t... Id, class... Td,
              size_t... Ip, class... Tp, class... Ap>
    tuple_impl(tuple_indices<Id...>, tuple_types<Td...>,
               tuple_indices<Ip...>, tuple_types<Tp...>,
               Ap&&... Args):
      tuple_leaf<Id, Td>()...,
      tuple_leaf<Ip, Tp>(std::forward<Ap>(Args))... { }

    template <class Tuple>
    tuple_impl(Tuple&& t):
      tuple_leaf<Is, Types>(
        std::forward<
          typename tuple_element<
            Is, 
            typename make_tuple_types<Tuple>::type
          >::type
        >(get<Is>(t))
      )...
    { }

    tuple_impl(const tuple_impl&) = default;
    tuple_impl(tuple_impl&&) = default;

    template <class Tuple>
    tuple_impl& operator=(Tuple&& t) {
#if __cplusplus >= 201703L
      (
        tuple_leaf<Is, Types>::operator=(
          std::forward<
            typename tuple_element<
              Is,
              typename make_tuple_types<Tuple>::type
            >::type
          >(get<Is>(t))
        ), ...
      );
#else
      swallow(
        tuple_leaf<Is, Types>::operator=(
          std::forward<
            typename tuple_element<
              Is,
              typename make_tuple_types<Tuple>::type
            >::type
          >(get<Is>(t))
        ) ...
      );
#endif
      return *this;
    }

    tuple_impl& operator=(const tuple_impl& t) {
#if __cplusplus >= 201703L
      (
        tuple_leaf<Is, Types>::operator=(
          static_cast<const tuple_leaf<Is, Types>&>(t).get()
        ), ...
      );
#else
      swallow(
        tuple_leaf<Is, Types>::operator=(
          static_cast<const tuple_leaf<Is, Types>&>(t).get()
        ) ...
      );
#endif
      return *this;
    }

    tuple_impl& operator=(tuple_impl&& t) {
#if __cplusplus >= 201703L
      (
        tuple_leaf<Is, Types>::operator=(
          std::forward<Types>(
            static_cast<tuple_leaf<Is, Types>&>(t).get()
          )
        ), ...
      );
#else
      swallow(
        tuple_leaf<Is, Types>::operator=(
          std::forward<Types>(
            static_cast<tuple_leaf<Is, Types>&>(t).get()
          )
        ) ...
      );
#endif
      return *this;
    }
};
```


#### `tuple`

`tuple` has one `tuple_impl` inside by composition.

* Crucial function `get` that grants access to data to users is friend function of `tuple`

```cpp
template <typename... Types>
class tuple {
  typedef tuple_impl<typename make_tuple_indices<sizeof...(Types)>::type, Types...> BaseType;

  protected:
    BaseType base;

  private:

    template <size_t I, class... Tp> friend 
    typename tuple_element<I, tuple<Tp...>>::type &get(tuple<Tp...>&);

    template <size_t I, class... Tp> friend 
    const typename tuple_element<I, tuple<Tp...>>::type &get(const tuple<Tp...>&);

    template <size_t I, class... Tp> friend 
    typename tuple_element<I, tuple<Tp...>>::type &&get(tuple<Tp...>&&);

    template <size_t I, class... Tp> friend 
    const typename tuple_element<I, tuple<Tp...>>::type &&get(const tuple<Tp...>&&);

  public:

    tuple() = default;

    template <typename... Ap>
    tuple(Ap&&... Args):
      base(typename make_tuple_indices<sizeof...(Types), sizeof...(Ap)>::type(),
           typename make_tuple_types<tuple, sizeof...(Types), sizeof...(Ap)>::type(),
           typename make_tuple_indices<sizeof...(Ap)>::type(),
           typename make_tuple_types<tuple, sizeof...(Ap)>::type(),
           std::forward<Ap>(Args)...) { }

    template <class Tuple>
    tuple(Tuple&& t): base(std::forward<Tuple>(t)) { }

    tuple(const tuple&) = default;
    tuple(tuple&&) = default;


    tuple& operator=(const tuple& t) {
      base.operator=(t.base);
      return *this;
    }

    tuple& operator=(tuple&& t) {
      base.operator=(static_cast<BaseType&&>(t.base));
      return *this;
    }

};

template <> class tuple<> { };
```

#### `get` by index

* get const/non-const and lvalue/rvalue reference
* use cast to access `tuple_leaf` properly
* return type is determined by `tuple_element`

```cpp
template <size_t I, class... Tp> inline
typename tuple_element<I, tuple<Tp...>>::type& 
get(tuple<Tp...>& t) {
  typedef typename tuple_element<I, tuple<Tp...>>::type type;
  return static_cast<tuple_leaf<I, type>&>(t.base).get();
}

template <size_t I, class... Tp> inline
const typename tuple_element<I, tuple<Tp...>>::type& 
get(const tuple<Tp...>& t) {
  typedef typename tuple_element<I, tuple<Tp...>>::type type;
  return static_cast<const tuple_leaf<I, type>&>(t.base).get();
}

template <size_t I, class... Tp> inline
typename tuple_element<I, tuple<Tp...>>::type&& 
get(tuple<Tp...>&& t) {
  typedef typename tuple_element<I, tuple<Tp...>>::type type;
  return static_cast<type&&>(
            static_cast<tuple_leaf<I, type>&&>(t.base).get());
}

template <size_t I, class... Tp> inline
const typename tuple_element<I, tuple<Tp...>>::type&& 
get(const tuple<Tp...>&& t) {
  typedef typename tuple_element<I, tuple<Tp...>>::type type;
  return static_cast<const type&&>(
            static_cast<const tuple_leaf<I, type>&&>(t.base).get());
}
```


#### `get` by type

* only well-defined when there is exactly one member of the type
* first find the index of the type and use `get` by index to access the data

```cpp
static constexpr size_t NOT_FOUND = -1;
static constexpr size_t AMBIGUOUS = -2;

inline constexpr size_t find_idx_return(size_t i, size_t res, bool match) {
  return !match ? res : (res==NOT_FOUND ? i : AMBIGUOUS);
}

template <size_t N>
inline constexpr size_t find_idx(size_t i, const bool (&matches)[N]) {
  return i==N ? NOT_FOUND : find_idx_return(i, find_idx(i+1, matches), matches[i]);
}


template <typename Tp, typename... Types>
struct find_exactly_one_checked {
  static constexpr bool matches[sizeof...(Types)] = {is_same<Tp, Types>::value...};
  static constexpr size_t value = find_idx(0, matches);
  static_assert(value != NOT_FOUND, "type not found");
  static_assert(value != AMBIGUOUS, "type found more than once");
};

template <typename Tp>
struct find_exactly_one_checked<Tp> {
  static_assert(!is_same<Tp, Tp>::value, "type not in empty list");
};


template <typename Tp, typename... Types>
struct find_exactly_one
  : public find_exactly_one_checked<Tp, Types...> { };


template <typename T, class... Tp> inline
constexpr T& get(tuple<Tp...>& t) {
  return get<find_exactly_one<T, Tp...>::value>(t);
}

template <typename T, class... Tp> inline
constexpr const T& get(const tuple<Tp...>& t) {
  return get<find_exactly_one<T, Tp...>::value>(t);
}

template <typename T, class... Tp> inline
constexpr T&& get(tuple<Tp...>&& t) {
  return get<find_exactly_one<T, Tp...>::value>(std::move(t));
}

template <typename T, class... Tp> inline
constexpr const T&& get(const tuple<Tp...>&& t) {
  return get<find_exactly_one<T, Tp...>::value>(std::move(t));
}
```


#### `tuple_cat`

* this function concatenate multiple `tuples` to one.
* the implementation of the function is one of the largest scales in all helper functions of `tuple`.


1. Determine the return type


```cpp
// type trait that remove const, volatile and reference
// the name of the type trait is different in GNUC and clang
template <class Tp>
struct uncvref {
  typedef typename std::remove_cv<typename std::remove_reference<Tp>::type>::type type;
};


template <class Tp, class Up>
struct tuple_cat_type;

template <class... Tps, class... Ups>
struct tuple_cat_type<tuple<Tps...>, tuple_types<Ups...>> {
  typedef tuple<Tps..., Ups...> type;
};


template <class ResTuple, bool IsTupleLike, class... Tuples>
struct tuple_cat_return_impl;

template <class... Tps, class Tuple0>
struct tuple_cat_return_impl<tuple<Tps...>, true, Tuple0> {
  typedef typename tuple_cat_type<
    tuple<Tps...>,
    typename make_tuple_types<
      typename uncvref<Tuple0>::type // such that make_tuple_types does not apply those qualities to types in Tuple0
    >::type
  >::type type;
};

template <class... Tps, class Tuple0, class Tuple1, class... Tuples>
struct tuple_cat_return_impl<tuple<Tps...>, true, Tuple0, Tuple1, Tuples...>
  : public tuple_cat_return_impl<
      typename tuple_cat_type<
        tuple<Tps...>,
        typename make_tuple_types<
          typename uncvref<Tuple0>::type
        >::type
      >::type,
      tuple_like<typename std::remove_reference<Tuple1>::type>::value,
      Tuple1,
      Tuples...
    >
{ };


template <class... Tuples> 
struct tuple_cat_return;

template <class Tuple0, class... Tuples>
struct tuple_cat_return<Tuple0, Tuples...> 
  : public tuple_cat_return_impl<
      tuple<>,
      tuple_like<typename std::remove_reference<Tuple0>::type>::value,
      Tuple0, Tuples...
    >
{ };


template <>
struct tuple_cat_return<> {
  typedef tuple<> type;
};
```


2. Determine the reference type of intermediate tuple

```cpp
template <class Tp, class Up>
struct apply_cv {
  using type = typename apply_cv_t<Tp>::template apply<Up>;
};


template <class RefTuple, class Is, class... Tuples>
struct tuple_cat_return_ref_impl;


template <class... Tps, size_t... Is, class Tuple0>
struct tuple_cat_return_ref_impl<tuple<Tps...>, tuple_indices<Is...>, Tuple0> {
  typedef typename std::remove_reference<Tuple0>::type T0;
  typedef tuple<
    Tps...,
    typename apply_cv<
      Tuple0,
      typename tuple_element<Is, T0>::type
    >::type&& ... // the rvalue reference work in the same way as perfect forwarding
  > type;
};


template <class... Tps, size_t... Is, class Tuple0, class Tuple1, class... Tuples>
struct tuple_cat_return_ref_impl<tuple<Tps...>, tuple_indices<Is...>, Tuple0, Tuple1, Tuples...>
  : public tuple_cat_return_ref_impl<
      tuple<
        Tps...,
        typename apply_cv<
          Tuple0,
          typename tuple_element<
            Is,
            typename std::remove_reference<Tuple0>::type // no specialization of tuple_element that handles reference
          >::type
        >::type&& ...
      >,
      typename make_tuple_indices<
        tuple_size<
          typename std::remove_reference<Tuple1>::type // no specialization of tuple_size that handles reference
        >::value
      >::type,
      Tuple1, Tuples...
    >
{ };


template <class Tuple0, class... Tuples>
struct tuple_cat_return_ref 
  : public tuple_cat_return_ref_impl<
      tuple<>,
      typename make_tuple_indices<
        tuple_size<
          typename std::remove_reference<Tuple0>::type
        >::value
      >::type,
      Tuple0, Tuples...
    >
{ };
```


3. `tuple_cat_impl`

```cpp
template <class... Tps> inline
constexpr tuple<Tps&&...> forward_as_tuple(Tps&&... t) {
  return tuple<Tps&&...>(std::forward<Tps>(t)...);
}


template <class Tuple, class Is, class Js>
struct tuple_cat_impl;

// Is is indices for tuple of references
// Js is indices for next tuple to be concatenated
template <class... Tps, size_t... Is, size_t... Js>
struct tuple_cat_impl<tuple<Tps...>, tuple_indices<Is...>, tuple_indices<Js...>> {

  template <class Tuple0>
  typename tuple_cat_return_ref<tuple<Tps...>&&, Tuple0&&>::type
  operator()(tuple<Tps...> t, Tuple0&& t0) {
    return ::forward_as_tuple(
      std::forward<Tps>(get<Is>(t))...,
      get<Js>(std::forward<Tuple0>(t0))...
    );
  }

  // the first argument is non-reference type because
  // it will be returned to tuple_cat
  // it only consists of references, thus costs little resource
  template <class Tuple0, class Tuple1, class... Tuples>
  typename tuple_cat_return_ref<tuple<Tps...>&&, Tuple0&&, Tuple1&&, Tuples&&...>::type
  operator()(tuple<Tps...> t, Tuple0&& t0, Tuple1&& t1, Tuples&&... ts) {
    typedef typename std::remove_reference<Tuple0>::type T0;
    typedef typename std::remove_reference<Tuple1>::type T1;

    return tuple_cat_impl<
      tuple<
        Tps...,
        typename apply_cv<
          Tuple0,
          typename tuple_element<Js, T0>::type
        >::type&&...
      >,
      typename make_tuple_indices<sizeof...(Tps) + tuple_size<T0>::value>::type,
      typename make_tuple_indices<tuple_size<T1>::value>::type
    >()(
      ::forward_as_tuple(
        std::forward<Tps>(get<Is>(t))...,
        get<Js>(std::forward<Tuple0>(t0))...
      ),
      std::forward<Tuple1>(t1),
      std::forward<Tuples>(ts)...
    );
  }

};
```

4. `tuple_cat` where tuple of references implicitly construct the returned tuple

```cpp
template <class Tuple0, class... Tuples>
typename tuple_cat_return<Tuple0, Tuples...>::type
tuple_cat(Tuple0&& t0, Tuples&&... ts) {
  typedef typename std::remove_reference<Tuple0>::type T0;

  return tuple_cat_impl<
    tuple<>,
    tuple_indices<>,
    typename make_tuple_indices<tuple_size<T0>::value>::type
  >()(
    tuple<>(),
    std::forward<Tuple0>(t0),
    std::forward<Tuples>(ts)...
  );
}
```

To summarize the algorithm of `tuple_cat`

Without the loss of generality, given `tuple<Tp...>` and `tuple<Up...>`


* firstly, `tuple<Tp...>` + `tuple<Up...>` -> `tuple<Tp&..., Up&...>`. For other specific cases

- `tuple<Tp...>&&` -> `tuple<Tp&&...>`
- `const volatile tuple<Tp...>&` -> `tuple<const volatile Tp&...>`


* at last, `tuple<Tp&..., Up&...>` -> `tuple<Tp..., Up...>` by implicit construction

The implementation goes all this way to handle `const`, `volatile` and `reference` properly and explicitly (by constructing return and reference types).

The carefulness can also be found in the constructors of `tuple` and `tuple_impl`.
