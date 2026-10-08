---
title: Writing your own random-access iterator
section: Standard library containers, algorithms, and iterators
section_href: /#standard-library-containers-algorithms-and-iterators
---

Back in chapter 1, [Enabling range-based for on your own types](/core-language/range-for-custom-types/) made a custom type iterable by giving it a small nested iterator. That iterator was deliberately minimal  three operations and no more: `operator*` to read the current element, prefix `operator++` to advance, and `operator!=` to tell when the end had been reached. That trio is the entire protocol range-based `for` asks for.

What is worth noticing now is everything that iterator could *not* do. You could not hand it to `std::sort`, `std::accumulate`, or any other standard algorithm, because it was not a model of any standard iterator category. It could not be copy-constructed and assigned the way the library requires, and it could not be incremented in any form beyond the single prefix `++`  no post-increment, no stepping backwards, no jumping *n* positions at once, no subscript, and none of the member type aliases the algorithms inspect. Range-based `for` is forgiving; the algorithms are not. To write an iterator the whole library will accept, you have to satisfy the full set of requirements for the category you are targeting.

This page builds the most capable of those categories  the *random-access iterator*, the one that can move to any element in constant time starting from an empty container. Before diving in, it helps to have the requirements in front of you: a compact overview of every iterator category and the operations each one is obliged to provide is at [cplusplus.com/reference/iterator](https://www.cplusplus.com/reference/iterator/).

## A container to iterate over

First we need something to iterate over. `dummy_array` is a thin wrapper around a fixed-size C array — just enough of a container to be worth giving an iterator, and nothing that would distract from the iterator itself:

```cpp
template <typename Type, size_t const SIZE>
class dummy_array
{
    Type data[SIZE] = {};

public:
    Type& operator[](size_t const index)
    {
        if (index < SIZE)
            return data[index];

        throw std::out_of_range("index out of range");
    }

    Type const& operator[](size_t const index) const
    {
        if (index < SIZE)
            return data[index];

        throw std::out_of_range("index out of range");
    }

    size_t size() const { return SIZE; }
};
```

The element count is a template parameter, so the size is baked into the type and the array lives inline with no allocation. Two subscript operators give read-write access to a `dummy_array` and read-only access to a `const dummy_array`, each bounds-checked so an out-of-range index throws rather than reads past the storage, and `size()` reports the fixed length. What it does *not* have yet is any `begin()` or `end()` — so at this point it works with neither range-based `for` nor a single algorithm. Supplying that is the whole job of the iterator we are about to write.

## The iterator's type aliases

To provide a mutable and constant random-access iterator for the `dummy_array` class shown in the previous section, write one class template and let a `bool` decide which of the two it is. Add the following members:

```cpp
template <typename T, size_t const Size, bool is_const>
class dummy_array_iterator
{
public:
    using self_type = dummy_array_iterator;
    using value_type = T;
    using reference = std::conditional_t<is_const, T const&, T&>;
    using pointer = std::conditional_t<is_const, T const*, T*>;
    using iterator_concept = std::random_access_iterator_tag;
    using iterator_category = std::random_access_iterator_tag;
    using difference_type = ptrdiff_t;
};
```

`value_type` stays bare `T`, as the traits require — it is `reference` and `pointer` that pick up `const` when `is_const` is `true`; instantiate the mutable form (`is_const == false`) and they collapse back to exactly `T&` and `T*`. `difference_type` must be a *signed* type, since subtracting two iterators can run negative, and `self_type` simply saves respelling the full template-id in the operations to come.

## The iterator's state

An iterator has to remember two things: where the data is, and where in it we are. The iterator class needs a random-access pointer to the array of data and the current index into that array:

```cpp
private:
    pointer ptr = nullptr;   // points at the array's data
    size_t index = 0;        // current position within it
```

`ptr` is the *base* pointer — it stays fixed at the start of the array while every move (`++`, `+= n`, subscript) works by changing `index` instead, so dereferencing is just `ptr[index]`.

## An explicit constructor

The container builds an iterator by handing it those two pieces of state — the base pointer and a starting index. An explicit constructor stores them in the iterator's members:

```cpp
public:
    explicit dummy_array_iterator(pointer ptr,
                                  size_t const index)
        : ptr(ptr), index(index)
    { }
```

The constructor name must match the class name: this page uses `dummy_array_iterator`; a class named `data_array_iterator` would use that name instead. In the member initializer list, `ptr(ptr)` initializes the member `ptr` from the parameter `ptr`, and `index(index)` does the same for `index`. Writing `index_index(index)` would try to initialize a member named `index_index`, which this class does not have.

With an `iterator` alias for the class, `begin()` can return `iterator{data, 0}` and `end()` can return `iterator{data, Size}` — two iterators over the same array that differ only in their index. These are direct-list-initializations, which can call an explicit constructor. A bare `return {data, 0};` uses copy-list-initialization and cannot select this explicit constructor. Both arguments are required regardless of `explicit`; a pointer alone is already insufficient.

Declaring any constructor suppresses the compiler-generated default one, so also add `dummy_array_iterator() = default;`, as shown below, since a random-access iterator must be default-initializable. The `const` on the by-value `index` parameter prevents changing that parameter inside the constructor; it does not make the iterator's stored index const.

## Iterator class members

Our random-access iterator must be default-constructible, copy-constructible, copy-assignable, destructible, swappable, and equality-comparable. It also needs prefix and postfix increment, dereference, and, for convenient member access, an arrow operator. The type aliases above support iterator traits and algorithm dispatch; C++20 concepts check the operations and their semantics as well, so aliases alone do not establish conformance.

Add these public members to `dummy_array_iterator`. Post-increment delegates to pre-increment to avoid code duplication. Include `<memory>` for `std::addressof` and `<stdexcept>` for the bounds checks:

```cpp
public:
    dummy_array_iterator() = default;
    dummy_array_iterator(dummy_array_iterator const& other) = default;
    dummy_array_iterator& operator=(dummy_array_iterator const& other) = default;
    ~dummy_array_iterator() = default;

    self_type& operator++()
    {
        if (ptr == nullptr || index >= Size)
            throw std::out_of_range("Iterator can not be incremented "
                                    "past the end of its range.");
        ++index;
        return *this;
    }

    self_type operator++(int)
    {
        self_type tmp = *this;
        ++(*this);
        return tmp;
    }

    reference operator*() const
    {
        if (ptr == nullptr || index >= Size)
            throw std::out_of_range("Iterator dereference out of range.");
        return ptr[index];
    }

    pointer operator->() const
    {
        return std::addressof(**this);
    }

    bool operator==(self_type const& other) const
    {
        return compatible(other) && index == other.index;
    }

    bool operator!=(self_type const& other) const
    {
        return !(*this == other);
    }
```

Use `= default;` for the special members and `++(*this)` to advance the iterator object. Two adjacent string literals keep the exception message valid across source lines. Advancing the last element to the end position is allowed; incrementing an iterator already at the end throws.

Dereference returns `ptr[index]` after rejecting a null pointer or an end position. The arrow operator reuses that check through `**this`; `std::addressof` obtains the real address even when the element type overloads `operator&`. Equality compares both the base pointer, through `compatible()` below, and the index. C++20 can rewrite `!=` from `==`, but the explicit definition shows the relationship.

These pointer-and-index iterators can use `std::swap` without a custom swap function; include `<utility>` when calling it. The defaulted copy operations can also accept rvalues. This is still only part of a random-access iterator: decrement, offset arithmetic, iterator subtraction, subscripting, and ordering remain to be added before the class satisfies `std::random_access_iterator`. The `iterator_concept` and `iterator_category` tags describe the intended completed iterator.

## Checking iterator compatibility

Add a private method to check whether two iterators refer to the same array data. Follow the existing `self_type` alias and `ptr` member names:

```cpp
private:
    bool compatible(self_type const& other) const
    {
        return ptr == other.ptr;
    }
```

The return statement ends with a semicolon, not a colon. `other` is passed by const reference to avoid copying it, and the trailing `const` lets the check run without modifying this iterator.

Because advancing changes `index` while leaving `ptr` fixed, iterators at different positions in the same array are compatible. Two iterators at index zero in different arrays are not. Compatibility therefore checks the underlying array, not equality of positions: an equality operator must also compare the indices. The ordering and difference operators can use this helper to check that both positions belong to the same array before comparing or subtracting their indices.
