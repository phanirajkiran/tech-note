Erlang Interoperability
=======================

# Standard Library

Erlang’s extensive standard library can be accessed from any Elixir code in our application. Erlang modules are represented by lowercase atoms such as `:os` and `:timer`.

```elixir
defmodule Example do
  def timed(fun, args) do
    {time, result} = :timer.tc(fun, args)
    IO.puts "Time: #{time}ms"
    IO.puts "Result: #{result}"
  end
end

iex> Example.timed(fn (n) -> (n * n) * n end, [100])
Time: 8ms
Result: 1000000
```

# Erlang Packages

```elixir
def deps do
  [{:png, github: "yuce/png"}]
end
```
