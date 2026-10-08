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

The container builds an iterator by handing it those two pieces of state  the base pointer and a starting index. A single constructor stores them, marked `explicit` so a bare `pointer` can never silently convert into an iterator:

```cpp
public:
    explicit dummy_array_iterator(pointer ptr, size_t const index)
        : ptr(ptr), index(index)
    { }
```

`begin()` will call it as `{ data, 0 }` and `end()` as `{ data, Size }` — two iterators over the same array that differ only in their index. One thing to note: declaring any constructor suppresses the compiler-generated default one, so you will also want `dummy_array_iterator() = default;`, since the random-access-iterator concept requires an iterator be default-initializable.

## Iterator class members

The iterator must be copy-constructible, copy-assignable, destructible, and incrementable. Add these public members to `dummy_array_iterator`. Post-increment delegates to pre-increment to avoid code duplication:

```cpp
public:
    dummy_array_iterator() = default;
    dummy_array_iterator(dummy_array_iterator const& other) = default;
    dummy_array_iterator& operator=(dummy_array_iterator const& other) = default;
    ~dummy_array_iterator() = default;

    self_type& operator++()
    {
        if (index >= Size)
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
```

Use `= default;` for the special members and `++(*this)` to advance the iterator object. Two adjacent string literals keep the exception message valid across source lines. Advancing the last element to the end position is allowed; incrementing an iterator already at the end throws.

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
