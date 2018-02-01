# Variadic operators for C++

<a name="introduction"></a>
## Introduction

This paper is a proposal to make the binary C++ operators variadic, so that they can
accept two or more arguments.


<a name="table_of_contents"></a>
## Table of Contents

- [Motivation](#motivation)
- [Design Goal](#design-goal)
- [Declaration](#declaration)
- [Lookup](#lookup)


<a name="motivation"></a>
## Motivation

Naturally a binary operator only 'sees' its immediate neighboring arguments. This can lead
to inefficient code in some circumstances. A prominent example is the concatenation of long
strings with ``operator+()``:

```cpp
std::string a = some_long_string();
std::string b = another_long_string();
std::string c = yet_another_long_string();
std::string z = a + b + c;
```

The creation of ``z`` is actually done in two steps similar to

```cpp
std::string temp = a + b;
std::string z = std::move(temp) + c;
```

Although the temporary ``temp`` can be moved, it is unlikely that the memory allocated for
``(a + b)`` is sufficiently large to also hold ``c``. So memory needs to be allocated twice
and in total 5 strings are copied (``a`` and ``b`` are copied twice, ``c`` once). The only
chance is that the compiler is smart enough to merge the memory allocations into a single
allocation and to eliminate the redundant copy operations.

This weakness of binary operators in C++ is well known.
The solution is to use a variadic function like Abseil's ``StrCat()`` [Abseil]:

```cpp
std::string z = absl::StrCat(a, b, c);
```

Since the function sees all strings at once, it can sum up their sizes, allocate the total
memory once and copy one string after another to the output.

Another possibility is to implement ``operator+()`` with [Expression Templates].
Instead of directly returning a ``std::string``, the operator would return a proxy, which
only stores references to the source strings. Chains of operator calls build up a tree
of such lightweight proxy objects. Only on the final assignment to a ``std::string`` the
real work is done.

While Expression Templates are a powerful means to defer the intended operations,
they can be quite challenging to implement. Another drawback is that they do not work well
with ``auto``. For the following assignment

```cpp
auto z = a + b + c;
```

one would intuitively assume that ``decltype(z) == std::string``, but if ``operator+()`` for
strings would use Expression Templates, the type might be ``string_cat_proxy``.


<a name="design_goal"></a>
## Design goal

The aim of this proposal is to extend the binary C++ operators to accept more than just
two parameters. A straightforward implementation to concatenate an arbitary number of
strings with only a single memory allocation and a minimum number of copies would be:

```cpp
template <typename... T>
std::string operator+(const std::string& a, const std::string& b, T... rest)
{
    // compute the final size first
    auto total_size = a.size() + b.size() + (rest.size() + ...);
    if (total_size == 0)
        return std::string();

    // allocate enough space for the result
    std::string result(total_size, '\0');
    // and copy the input strings
    auto iter = result.begin();
    iter = std::copy(a.cbegin(), a.cend(), iter);
    iter = std::copy(b.cbegin(), b.cend(), iter);
    (void(iter = std::copy(rest.cbegin(), rest.cend(), iter)), ...);

    return result;
}
```

Note that a production-ready implemenation would take care of constraining the types in the
parameter pack in order to prevent matching against ``int``, for example. Furthermore,
the implementation should be adapted to deal with ``const char*`` efficiently.


<a name="declaration"></a>
## Declaration

The declaration of variadic operators is just an extension to the declaration of binary operators

```cpp
string operator+(string, string, string).  // concatenation operator with arity 3
string operator+(string, string, string, string).  // concatenation operator with arity 4
template <typename... T>
string operator+(string, string, T... rest).  // concatenation operator with arity n
```

It is an error to declare an n-ary operator, which can also be nullary or unary.

```cpp
template <typename... T>
string operator(T... strings); // error: can be nullary, unary or n-ary

template <typename... T>
string operator(string, T... strings); // error: can be unary or n-ary
```


<a name="lookup"></a>
## Lookup

The operator lookup rules are changed to incorporate n-ary operators. Operators with higher arity are
prefered, so they are selected over operators with lower arity. The interaction between unary and binary
operators is not changed.

In the following it will be convenient to have a common notation for free operators and member operators.
We simply write ``operator@(a, b)`` and mean both a free operator or the member operator ``a.operator@(b)``.


The C++ grammar only deals with binary operators. So an statement

```cpp
z = a + b + c + d;
```

will naturally lead to the abstract syntax tree

```cpp
operator=(z, operator+(operator+(operator+(a, b), c), d))
```

or graphically

```
         (=)
          |
       .--+--.
       |     |
       z    (+)
             |
          .--+--.
          |     |
         (+)    d
          |
       .--+--.
       |     |
      (+)    c
       |
    .--+--.
    |     |
    a     b
```

In order to support operators with arity ``n``, ``n >= 2``, an additional binary-to-n-ary conversion
step is added before the lookup of operators is done. This conversion linearizes expression trees
by lifting the nested calls to the same operator. The conversion is done iteratively until
no further changes to the expression tree are possible.

In each iteration step, walk over all n-ary operators in the expression tree in post-order.
Note that in the first iteration all n-ary operators are actually binary.
Let ``@`` be the currently visited operator. If the operator is left-associative and its left-most
operand is also the n-ary operator ``@``, lift the arguments from the nested operator to the current
operator and concatenate them on the left of the current argument list.

For the previous example, the conversion would look like:
```
         (=)                (=)                (=)
          |                  |                  |
       .--+--.            .--+--.            .--+--.
       |     |            |     |            |     |
       z    (+)           z    (+)           z    (+)
             |                  |                  |
          .--+--.   -->      .--+--.   -->      .--+--+--.
          |     |            |     |            |  |  |  |
         (+)    d           (+)    d            a  b  c  d
          |                  |
       .--+--.            .--+--.
       |     |            |  |  |
      (+)    c            a  b  c
       |
    .--+--.
    |     |
    a     b
```

So after n-ary conversion, the assignment expression becomes

```cpp
operator=(z, operator+(a, b, c, d))
```

Now the compiler performs the name lookup. If none of the arguments to an operator is
of class or enumeration type, the lookup for that operator is skipped and the operator
behaves as it would have been written with binary operators, i.e. the operator is folded
back to its original form.

Otherwise, the lookup following the ordinary lookup rules. The
``operator+()`` with arity 4 will match either a free operator

```cpp
operator+(a, b, c, d)
```

or a member operator

```cpp
a.operator+(b, c, d)
```

If no suitable match is found, the right-most argument is split off from the operator and
the lookup continues with ``operator+(a, b, c)``. If that also fails, the binary operator
``operator+(a, b)`` is tried. Only if the compiler cannot find a match for a binary operator,
it will emit an error.

The search is greedy. If an overload for ``operator+(a, b, c)`` is found, the compiler does
not consider ``operator+(a, b)`` anymore. This might imply that it cannot find a matching
overload for ``operator+(t, d)``, with ``t = operator+(a, b, c)`` later on. It emits an
error in this case.


Summing up, the following variants are tried by the compiler

```cpp
    operator=(z, operator+(a, b, c, d))
    operator=(z, operator+(operator+(a, b, c), d))
    operator=(z, operator+(operator+(operator+(a, b), c), d))
```

<a name="references"></a>
## References

[Abseil] https://abseil.io/tips/3

[Expression Templates] https://de.wikipedia.org/wiki/Expression_Templates
