---
title: "std::at : Range-checked accesses to arbitrary collections"
document: PxxxxR0
date: today
audience: Evolution
author:
  - name: Andre Kostur
    email: andre@kostur.net
toc: true
---

# Introduction

Lately there has been a stronger push towards code safety, in particular
performing range-checked accesses to collections.  It would be nice to
have a consistent and generic syntax for performing such accesses into
arbitrary collections.

# Motivation and Scope

A number of existing collections have both an `operator[]` for unchecked
accesses, as well as an `at()` member function for range-checked accesses
which may throw an `std::out_of_range` exception.  However, this is not
universally true.  Some collections may not have an `at()` member through
design oversight.  Plain arrays could also benefit through having a free
function `std::at()` apply to them.

This issue was recognized when dicussing the addition of an `at()`
member function to `std::ranges::view_interface` [@P3052R1].

# Design Overview

Create a set of free-function `std::at()` which may be used instead of
calling a member-function `at()`.  This is a similar mechanism that is
already used for `std::begin()`.

Create additional overloads of `std::at()` to cover additional use-cases:

1. Collection types which do not have an `at()` member function, but do have
   an `operator[]` and `size()`.
2. Normal arrays

Allow for user-declared overloads of `at()` so that users could provide the
range-checked behaviour to classes that do not conform to the above cases.

# Reference implementation

```cpp
#ifndef STDAT_H
#define STDAT_H

namespace std {

template <typename T, typename U>
concept _has_at = requires(T t, U u) { t.at(u); };

template<typename T>
concept _has_index = requires(T t) { t[0]; };

template<typename T>
concept _has_size = requires(T t) { t.size(); };

template<typename T, typename U>
concept _can_range_check_index = _has_index<T> and _has_size<T> and not _has_at<T, U>;

// Arbirary types that have an at() member function 
template <typename T, typename U> requires _has_at<T, U>
constexpr decltype(auto) at(T & coll, U && idx)
{
    return coll.at(std::forward<U>(idx));
}

// Arbitrary types that have an indexing operator and a size member function 
template <typename T>
decltype(auto) at(T & coll, size_t idx) requires _can_range_check_index<T, size_t>
{
    if (coll.size() <= idx)
    {
        throw std::out_of_range("array");
    }

    return coll[idx];
}

// Plain arrays
template<typename T, size_t N>
constexpr decltype(auto) at(T (&coll)[N], size_t idx)
{
    if (N <= idx)
    {
        throw std::out_of_range("array");
    }

    return coll[idx];
}

}

#endif //STDAT_H
```




