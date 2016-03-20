Error Handling
==============

# raise

```elixir
iex> raise "Oh no!"
** (RuntimeError) Oh no!
iex> raise ArgumentError, message: "the argument value is invalid"
** (ArgumentError) the argument value is invalid
```

# try/rescue/after

```elixir
try do
  opts
  |> Keyword.fetch!(:source_file)
  |> File.read!
rescue
  e in KeyError -> IO.puts "missing :source_file option"
  e in File.Error -> IO.puts "unable to read source file"
after
  IO.puts "The end!"
end
```

In practice, however, Elixir developers rarely use the `try/rescue` construct.

Many functions in the standard library follow the pattern of having a counterpart that raises an exception instead of returning tuples to match against. The convention is to create a function (`foo`) which returns `{:ok, result}` or `{:error, reason}` tuples and another function (`foo!`, same name but with a trailing `!`) that takes the same arguments as foo but which raises an exception if there’s an error. `foo!` should return the result (not wrapped in a tuple) if everything goes fine.

```elixir
iex> case File.read "hello" do
...>   {:ok, body}      -> IO.puts "Success: #{body}"
...>   {:error, reason} -> IO.puts "Error: #{reason}"
...> end

iex> File.read! "unknown"
** (File.Error) could not read file unknown: no such file or directory
    (elixir) lib/file.ex:305: File.read!/1
```

# Custom Errors

```elixir
defmodule ExampleError do
  defexception message: "an example error has occurred"
end
```

# Exiting

Exit signals occur whenever a process dies and are an important part of the **fault tolerance** of Elixir.

To explicitly exit we can use `exit/1`:

```elixir
iex> spawn_link fn -> exit("oh no") end
** (EXIT from PID<0.101.0>) "oh no"
```
