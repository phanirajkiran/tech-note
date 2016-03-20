Functions
=========

# Anonymous functions

```elixir
iex> sum = fn (a, b) -> a + b end
iex> sum.(2, 3)
5
```

There is also a shorthand for doing so:

```elixir
iex> sum = &(&1 + &2)
iex> sum.(2, 3)
5
```

A dot (`.`) between the variable and parenthesis is required to invoke an anonymous function.

# Pattern matching

Pattern matching can be also applied to function signatures:

```elixir
iex> handle_result = fn
...>   {:ok, result} -> IO.puts "Handling result..."
...>   {:error} -> IO.puts "An error has occurred!"
...> end

iex> some_result = 1
iex> handle_result.({:ok, some_result})
Handling result...

iex> handle_result.({:error})
An error has occurred!
```

# Named functions

Named functions defined with `def` should be defined within a module, and they are available to other modules for use, this is a particularly useful building block in Elixir. `defp` is used to define **private functions** which cannot be accessed by other modules.

If we want a default value for an argument we use the `argument \\ value` syntax.

```elixir
defmodule Greeter do
  def hello(name, country \\ "en") do
    phrase(country) <> name
  end
  defp phrase("en"), do: "Hello, "
  defp phrase("zh"), do: "你好, "
end

iex> Greeter.hello("Jimmy")
"Hello, Sean"
iex> Greeter.hello("Jimmy", "zh")
"你好, Sean"
iex> Greeter.phrase("en")
** (UndefinedFunctionError) undefined function Greeter.phrase/1
    Greeter.phrase("en")
```

# Loops through recursion

Due to **immutability**, loops in Elixir (as in any functional programming language) are written differently from imperative languages. Instead, functional languages rely on recursion, and it can be implemented by using named function with pattern matching:

```elixir
defmodule Recursion do
  def print_multiple_times(msg, n) when n <= 1 do
    IO.puts msg
  end

  def print_multiple_times(msg, n) do
    IO.puts msg
    print_multiple_times(msg, n - 1)
  end
end

Recursion.print_multiple_times("Hello!", 3)
# Hello!
# Hello!
# Hello!
```

And another example to implement map and reduce:

```elixir
defmodule Math do
  def sum_list([head|tail], accumulator) do
    sum_list(tail, head + accumulator)
  end

  def sum_list([], accumulator) do
    accumulator
  end
end

IO.puts Math.sum_list([1, 2, 3], 0) #=> 6
```

# Guards

We can have many fumctions with the same signature and rely on **guards** to determine which to use based on the argument’s type:

```elixir
defmodule Greeter do
  def hello(names) when is_list(names) do
    names
    |> Enum.join(", ")
    |> hello
  end

  def hello(name) when is_binary(name) do
    phrase <> name
  end

  defp phrase, do: "Hello, "
end

iex> Greeter.hello ["Jimmy", "Steve"]
"Hello, Jimmy, Steve"
```

However, Elixir doesn’t like default arguments in multiple matching functions, it can be confusing. To handle this we add a function head with our default arguments:

```elixir
defmodule Greeter do
  def hello(names, country \\ "en")   # function header
  def hello(names, country) when is_list(names) do
    names
    |> Enum.join(", ")
    |> hello(country)
  end

  def hello(name, country) when is_binary(name) do
    phrase(country) <> name
  end

  defp phrase("en"), do: "Hello, "
  defp phrase("zh"), do: "你好, "
end

iex> Greeter.hello ["Jimmy", "Steve"]
"Hello, Jimmy, Steve"

iex> Greeter.hello ["Jimmy", "Steve"], "zh"
"你好, Jimmy, Steve"
```

Moreover, errors in guards do not leak but simply make the guard fail. If none of the clauses match, an error is raised.

# Protocols

Protocols are a mechanism to achieve **polymorphism** in Elixir. Dispatching on a protocol is available to any data type as long as it implements the protocol.

```elixir
defprotocol Blank do
  @doc "Returns true if data is considered blank/empty"
  def blank?(data)
end
```

The protocol expects a function called `blank?` that receives one argument to be implemented. We can implement this protocol for different Elixir data types as follows:

```elixir
# Integers are never blank
defimpl Blank, for: Integer do
  def blank?(_), do: false
end

# Just empty list is blank
defimpl Blank, for: List do
  def blank?([]), do: true
  def blank?(_),  do: false
end

# Just empty map is blank
defimpl Blank, for: Map do
  def blank?(map), do: map_size(map) == 0
end

# Just the atoms false and nil are blank
defimpl Blank, for: Atom do
  def blank?(false), do: true
  def blank?(nil),   do: true
  def blank?(_),     do: false
end
```

# Typespecs

## Function specifications

```elixir
@spec round(number) :: integer
def round(number), do: # implementation...
```

`::` means that the function on the left side *returns* a value whose type is what’s on the right side. Function specs are written with the `@spec` directive, placed right before the function definition.

## Defining custom types

While Elixir provides a lot of useful built-in types, it’s convenient to define custom types when appropriate. This can be done when defining modules through the `@type` directive.

```elixir
defmodule LousyCalculator do
  @typedoc """
  Just a number followed by a string.
  """
  @type number_with_remark :: {number, String.t}

  @spec add(number, number) :: number_with_remark
  def add(x, y), do: {x + y, "You need a calculator to do that?"}

  @spec multiply(number, number) :: number_with_remark
  def multiply(x, y), do: {x * y, "It is like addition on steroids."}
end
```
