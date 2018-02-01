# Proposal for C++ variadic operators

# What is this?

This is a [**proposal**](https://github.com/ombre5733/cpp_variadic_operators_proposal/blob/master/proposal.md)
to bring variadic operators to C++.

# What can I do with it?

## Efficient string concatenation

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

## Efficient vector math

```cpp
template <typename... T>
Vector operator+(const Vector& a, const Vector& b, T... rest)
{
    Vector r(a.size());
    for (int i = 0; i < a.size(); ++i)         // loop only once over the elements
        r[i] = a[i] + b[i] + (rest[i] + ...);  // sum the components of all vectors
    return r;
}
```

## Python-like comparison chains:

```cpp
template <typename... T>
bool operator<(Number a, Number b, T... rest)
{
    if constexpr (sizeof...(rest) == 0)
        return int(a) < int(b);
    else
        return int(a) < int(b) && operator<(b, rest...);
}

if (Number(1) < Number(10) < Number(50) < Number(70))  // ... Number(1) < Number(10) && Number(10) < Number(50)
    std::cout << "ordered" << std::endl;               //       && Number(50) < Number(70)
```

## Locked streaming

```cpp
struct locked_stream
{
    template <typename T, typename... TRest>
    locked_stream& operator<<(const T& t, const TRest&... rest)
    {
        std::lock_guard<std::mutex> lock(m_mutex);  // acquire the lock once
        m_out << t;
        (void(m_out << rest), ...);                 // and stream all arguments
        return *this;
    }

    std::mutex m_mutex;
    std::ostream& m_out;
};
```

# What is the status?

Currently, this is a rough draft to present the idea and to collect feedback. No standardese has been written yet.

# Will it come to C++20?

Depends on the feedback from the community and [your input](https://github.com/ombre5733/cpp_variadic_operators_proposal/issues/new).

