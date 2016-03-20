Executables
===========

To build executables in Elixir we will be using escript. Escript produces an executable that can be run on any system with Erlang installed.

# Getting Started

We’ll start by creating a module to serve as the entry point (`main/1`) to our executable:

```elixir
defmodule ExampleApp.CLI do
  def main(args \\ []) do
    # Do stuff
  end
end
```

Next we need to update our Mixfile to include the `:escript` option for our project along with specifying our `:main_module`:

```elixir
defmodule ExampleApp.Mixfile do
  def project do
    [app: :example_app,
     version: "0.0.1",
     escript: escript]
  end

  def escript do
    [main_module: ExampleApp.CLI]
  end
end
```

To build, simply run `mix escript.build`.

# Parsing Args

```elixir
defmodule ExampleApp.CLI do
  def main(args \\ []) do
    args
    |> parse_args
    |> response
    |> IO.puts
  end

  defp parse_args(args) do
    {opts, word, _} =
      args
      |> OptionParser.parse(switches: [upcase: :boolean])

    {opts, List.to_string(word)}
  end

  defp response({opts, "Hello"}), do: response({opts, "World"})
  defp response({opts, word}) do
    if opts[:upcase], do: word = String.upcase(word)
    word
  end
end
```
