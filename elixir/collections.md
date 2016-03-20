Collections
===========

# Lists

Elixir implements list as linked lists. This means accessing the list length is an `O(n)` operation. For this reason, it is typically faster to prepend than append:

```elixir
iex> list = [3.41, :pie, "Apple"]
[3.41, :pie, "Apple"]
iex> ["π"] ++ list
["π", 3.41, :pie, "Apple"]
iex> list ++ ["Cherry"]
[3.41, :pie, "Apple", "Cherry"]
```

## List Concatenation

```elixir
iex> [1, 2] ++ [3, 4, 1]
[1, 2, 3, 4, 1]
```

## List Subtraction

It’s safe to subtract a missing value:

```elixir
iex> ["foo", :bar, 42] -- [42, "bar"]
["foo", :bar]
```

## Head / Tail

```elixir
iex> hd [3.41, :pie, "Apple"]
3.41
iex> tl [3.41, :pie, "Apple"]
[:pie, "Apple"]
```

# Tuples

Tuples are similar to lists but are stored contiguously in memory. This makes accessing their length fast but modification expensive; the new tuple must be copied entirely to memory.

```elixir
iex> tuple = {:ok, "hello"}
{:ok, "hello"}
iex> elem(tuple, 1)
"hello"
iex> tuple_size(tuple)
2
iex> put_elem(tuple, 1, "world")
{:ok, "world"}
```

# Keyword lists

Keywords and maps are the associative collections of Elixir; both implement the `Dict` module. In Elixir, a keyword list is a special list of tuples whose first element is an atom; they share performance with lists:

```elixir
iex> [foo: "bar", hello: "world"]
[foo: "bar", hello: "world"]
iex> [{:foo, "bar"}, {:hello, "world"}]
[foo: "bar", hello: "world"]
```

- Keys are atoms
- Keys are ordered
- Keys are not unique

Since keyword lists are simply lists, they are used in Elixir mainly as **options**. If you need to store many items or guarantee one-key associates with at maximum one-value, you should use maps instead.

# Maps

Unlike keyword lists, maps allow keys of any type and they do not follow ordering.

```elixir
iex> map = %{:foo => "bar", "hello" => :world}
%{:foo => "bar", "hello" => :world}
iex> map[:foo]
"bar"
iex> map["hello"]
:world
iex> %{map | "hello" => :elixir}
%{:foo => "bar", "hello" => :elixir}
iex> %{map | "world" => :hello}
** (KeyError) key "world" not found in: %{:foo => "bar", "hello" => :world}
```

If a duplicate is added to a map, it will replace the former value:

```elixir
iex> %{:foo => "bar", :foo => "hello world"}
%{foo: "hello world"}
```

The `Map` module provides a set of APIs to manipulate maps:

```elixir
iex> Map.put(%{:a => 1}, 2, :b)
%{2 => :b, :a => 1}
iex> Map.get(%{:a => 1, 2 => :b}, :a)
1
iex> Map.to_list(%{:a => 1, 2 => :b})
[{2, :b}, {:a, 1}]
```

When all the keys in a map are atoms, you can use the keyword syntax for convenience:

```elixir
ex> %{foo: "bar", hello: "world"}
%{foo: "bar", hello: "world"}

iex> %{foo: "bar", hello: "world"} == %{:foo => "bar", :hello => "world"}
true
```
