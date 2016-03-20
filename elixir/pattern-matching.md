Pattern Matching
================

# Match operator

In Elixir, the `=` operator is actually our match operator.

```elixir
iex> x = 1
1
iex> 1 = x
1
iex> 2 = x
** (MatchError) no match of right hand side value: 1
```

The match operator handles assignment when the left side of the match includes a variable.

```elixir
# Lists
iex> [1|tail] = list
[1, 2, 3]
iex> tail
[2, 3]

# Tuples
iex> {:ok, value} = {:ok, "Successful!"}
{:ok, "Successful!"}
iex> value
"Successful!"
```

# Pin operator

When we pin a variable we match on the existing value rather than rebinding to a new one.

```elixir
iex> x = 1
1
iex> ^x = 2
** (MatchError) no match of right hand side value: 2
iex> {x, ^x} = {2, 1}
** (MatchError) no match of right hand side value: {2, 1}
```

For pinning in a function clause:

```elixir
iex> greeting = "Hello"
"Hello"
iex> greet = fn
...>   (^greeting, name) -> "Hi #{name}"
...>   (greeting, name) -> "#{greeting}, #{name}"
...> end
iex> greet.("Hello", "Jimmy")
"Hi Jimmy"
iex> greet.("Morning", "Jimmy")
"Morning, Jimmy"
```
