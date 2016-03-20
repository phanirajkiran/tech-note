Enum
====

Enum is a set of algorithms for enumerating over collections.

# at

```elixir
iex> Enum.at(["foo", "bar", "hello"], 1)
"bar"
iex> Enum.at(["foo", "bar", "hello"], 3, "null")
"null"
```

# all?

```elixir
iex> Enum.all?(["foo", "bar", "hello"], fn(s) -> String.length(s) > 1 end)
true
```

# any?

```elixir
iex> Enum.any?(["foo", "bar", "hello"], fn(s) -> String.length(s) == 5 end)
true
```

# chunk

```elixir
iex> Enum.chunk([1, 2, 3, 4, 5, 6], 2)
[[1, 2], [3, 4], [5, 6]]
iex> Enum.chunk([1, 2, 3, 4, 5, 6], 4)
[[1, 2, 3, 4]]
```

# chunk_by

```elixir
iex> Enum.chunk_by(["one", "two", "three", "four", "five"], fn(x) -> String.length(x) end)
[["one", "two"], ["three"], ["four", "five"]]
```

# each

```elixir
iex> Enum.each(["one", "two", "three"], fn(s) -> IO.puts(s) end)
one
two
three
```

# max/min

```elixir
iex> Enum.max([5, 3, 0, -1])
5
iex> Enum.min([5, 3, 0, -1])
-1
```

# map

```elixir
iex> Enum.map([0, 1, 2, 3], fn(x) -> x - 1 end)
[-1, 0, 1, 2]
```

# reduce

```elixir
iex> Enum.reduce([1, 2, 3], fn(x, acc) -> x + acc end)
6
iex> Enum.reduce([1, 2, 3], 10, fn(x, acc) -> x + acc end)
16
```

# sort

```elixir
iex> Enum.sort([5, 6, 1, 3, -1, 4])
[-1, 1, 3, 4, 5, 6]
iex> Enum.sort([5, 6, 1, 3, -1, 4], fn(x, y) -> x > y end)
[6, 5, 4, 3, 1, -1]
```

# uniq

We can use `uniq` to remove duplicates from our collections:

```elixir
iex> Enum.uniq([1, 2, 2, 3, 3, 3, 4, 4, 4, 4])
[1, 2, 3, 4]
```

# Eager vs Lazy

All the functions in the `Enum` module are eager. This means that when performing multiple operations with `Enum`, each operation is going to generate an intermediate list until we reach the result.

As an alternative to `Enum`, Elixir provides the `Stream` module which supports lazy operations:

```elixir
iex> 1..100_000 |> Stream.map(&(&1 * 3)) |> Stream.filter(odd?) |> Enum.sum
7500000000
```

In the example above, `1..100_000 |> Stream.map(&(&1 * 3))` returns a data type, an actual stream, that represents the map computation over the range 1..100_000:

```elixir
iex> 1..100_000 |> Stream.map(&(&1 * 3))
#Stream<[enum: 1..100000, funs: [#Function<34.16982430/1 in Stream.map/2>]]>
```

Instead of generating intermediate lists, streams build a series of computations that are invoked only when we pass the underlying stream to the `Enum` module. `Streams` are useful when working with large, *possibly infinite*, collections.

# Comprehensions

In Elixir, it is common to loop over an Enumerable, often filtering out some results and mapping values into another list. Comprehensions are **syntactic sugar** for such constructs: they group those common tasks into the `for` special form:

```elixir
iex> for n <- [1, 2, 3, 4], do: n * n
[1, 4, 9, 16]
iex> for n <- 1..4, do: n * n
[1, 4, 9, 16]
iex> for i <- [:a, :b, :c], j <- [1, 2], do:  {i, j}
[a: 1, a: 2, b: 1, b: 2, c: 1, c: 2]

iex> values = [good: 1, good: 2, bad: 3, good: 4]
iex> for {:good, n} <- values, do: n * n
[1, 4, 16]
```

The result of a comprehension can be inserted into different data structures by passing the `:into` option to the comprehension:

```elixir
iex> for {key, val} <- %{"a" => 1, "b" => 2}, into: %{}, do: {key, val * val}
%{"a" => 1, "b" => 4}
```
