Control Structures
==================

# if

```elixir
iex> x = 1
1
iex> if x > 1 do
...>   "x is greater than 1."
...> else
...>   "x is less than or equal to 1."
...> end
"x is less than or equal to 1."
```

Since in Elixir’s regular syntax, each argument is separated by comma, we can pass `else` using keywords:

```elixir
iex> if false, do: :this, else: :that
:that
```

# unless

```elixir
iex> unless is_integer("hello") do
...>   "Not an Int"
...> end
"Not an Int"
```

Actually `if/2` and `unless/2` are implemented as macros in `Kernel` module of Elixir.

# case

Consider `_` as the else that will match “everything else”.

```elixir
iex> case :even do
...>   :odd -> "Odd"
...> end
** (CaseClauseError) no case clause matching: :even

iex> case :even do
...>   :odd -> "Odd"
...>   _ -> "Not Odd"
...> end
"Not Odd"
```

Since `case` relies on **pattern matching**, all of the same rules and restrictions apply. If you intend to match against existing variables you must use the pin `^` operator:

```elixir
iex> pie = 3.41
3.41
iex> case "cherry pie" do
...>   ^pie -> "Not so tasty"
...>   pie -> "I bet #{pie} is tasty"
...> end
"I bet cherry pie is tasty"
```

`case` also supports guard clauses:

```elixir
iex> case {1, 2, 3} do
...>   {1, x, 3} when x > 0 ->
...>     "Will match"
...>   _ ->
...>     "Won't match"
...> end
"Will match"
```

# cond

`cond` is used to match conditions, it's akin to `else if` or `elsif` from other languages. To handle no match, we can define a condition set to `true`.

```elixir
iex> cond do
...>   2 + 2 == 5 ->
...>     "This will not be true"
...>   2 * 2 == 3 ->
...>     "Nor this"
...>   1 + 1 == 2 ->
...>     "But this will"
...>   true ->
...>     "Catch all"
...> end
"But this will"
```
