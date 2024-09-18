---
title: "std::at : Range-checked accesses to arbitrary containers"
document: DxxxxR0
date: today
audience: LEWG, SG23 (Safety and Security), SG9 (Ranges)
author:
  - name: Andre Kostur
    email: <andre@kostur.net>
toc: true
---

# Abstract

This proposal is to add a set of free-function overloads for `std::at()`
which will forward to containers which have a member function `at()`.
In addition this will also apply to arrays.

# Motivation 

Lately there has been a stronger push towards code safety, in particular
performing range-checked accesses to containers.  A number of existing
containers have both an `operator[]` for unchecked accesses, as well as
an `at()` member function for range-checked accesses which may throw an
`std::out_of_range` exception.  It would be useful to be able to have
an `at()` for user-declared types which may not have provded their own
as well as having range-checked accesses to arrays.  This facilitates
generic programming with containers in the same manner that `std::begin()`
does.

# Revision History

## R0

Initial revision.

# Design

Define a set of templated `std::at()` free-functions which may be used
instead of calling a member-function `at()`.  This is a similar mechanism
that which is already used for `std::begin()`.  

`std::mdspan` and `std::mdarray` are interesting types for consideration.
They support multidimensional access, but do not yet have an `at()`
member function.  We should anticipate the addition of `std::mdspan::at()`
and `std::mdarray::at()` that will have similar function signatures as
`std::mdspan::operator[]` and `std::mdarray::operator[]` and thus `std::at()`
should take a variadic number of indices.

An additional consideration for `std::at()` is that the member function `at()`
may have multiple overloads in the container taking differing types and
numbers of arguments.

There are three cases that this proposal needs to consider:

1. Containers which have a member-function `at()`.  `std::at()` will
   call the member function, forwarding all of the additional arguments.
2. Arrays cannot have member functions.  `std::at()` will test the
   index to verify that it is in range, and will throw `std::out_of_range`
   when the range is violated.
3. User declared free-function `at()`, presumably in the same namespace
   as the container so as to take advantage of ADL to choose this function
   over the templated function that will be provided in the standard
   library.

As `at()` functions use exceptions as the error reporting mechanism, this is
not applicable to freestanding and should be explicitly marked as
freestanding-deleted, just as `span::at()` is.

# Discarded Case

Containers which do not have an `at()` member function, but do have
`operator[]` and `size()` member functions, or a custom overload of `size()`.
And overload of `std::at()` could do a range-check using the result of
calling `size()`, and then either returning the result of `operator[]`, or
throw an exception. However, attempting to address this case introduces a
number of concerns:

1. There is no assurance that any of the overlaods of `operator[]` on the
   container take an integral parameter.
2. There is no assurance that the valid range of values for the integral
   index necessarily go from 0..`size()`.
3. If this hypothetical overload existed, it would need to be templated
   on the index type.  Which would mean that a user likely need to declare
   their own custom overload also templated on the index type.  Otherwise
   if the custom overload takes a `size_t` as the index, but is passed an
   `int` literal, this would match the template provided by the standard
   library instead of implicitly converting to `size_t` and invoking the
   custom overload.

# Proposed Wording

Add the following clauses to [iterator.range]{.sref}:
 
:::add
    template <class C, class... _Idxs>
    constexpr auto at(C& coll, _Idxs&&... idxs)
        -> decltype(coll.at(std::forward<_Idxs>(idxs)...));
        
        Returns: coll.at(idxs...)

    template <class C, size_t N>
    constexpr auto at(C (&array)[N], size_t idx) -> C&;

        Returns: array[idx]
        Throws: std::out_of_range if idx >= N        
:::

# Feature Test Macro

```
#define __cpp_lib_at xxxxxxL
```

# Example implementation

## Case 1: Containers with `at()` member function(s)

```cpp
namespace std
{
    template <typename _Tp, typename... _Idxs>
    constexpr decltype(auto) at(_Tp& coll, _Idxs&&... idxs)
        requires requires(_Tp coll, _Idxs&&... idxs)
            { coll.at(std::forward<_Idxs>(idxs)...); }
    {
        return coll.at(std::forward<_Idxs>(idxs)...);
    }
}
```

## Case 2: Arrays

```cpp
namespace std
{
    template <typename _Tp, size_t N>
    constexpr auto at(_Tp (&array)[N], size_t idx) -> _Tp&
    {
        if (N <= idx)
        {
            throw std::out_of_range("generic at");
        }

        return array[idx];
    }
}
```

## Case 3: Custom overload

There is no example implementation as this is user-supplied.

# Questions

Are we about to cause problems by introducing a `std::at()`
identifier like we did when we added `std::byte`?  We should be
able to introduce new identifiers in std:: namespace without
worrying about this as it would greatly hobble the committee's
ability to add new things.  Additionaly the problem with byte was
that for at least one platform, they had defined a macro "byte"
which caused the issue, and they resolved that issue.

# Acknowledgements

Thank-you to Herb Sutter for the initial inspiration.
