Documentation
=============

# Annotation

Elixir treats documentation as a *first-class citizen*, provides us with many different attributes to annotate a codebase:

- `#`: For inline documentation
- `@moduledoc`: For module-level documentation
- `@doc`: For function-level documentation

## Documenting Modules

```elixir
defmodule Greeter do
  @moduledoc """
  Provides a function `hello/1` to greet a human
  """

  def hello(name) do
    "Hello, " <> name
  end
end
```

We (or others) can access this module documentation using the `h` helper function within IEx.

## Documenting Functions

```elixir
defmodule Greeter do
  @moduledoc """
  ...
  """

  @doc """
  Prints a hello message

  ## Parameters

    - name: String that represents the name of the person.

  ## Examples

      iex> Greeter.hello("Sean")
      "Hello, Sean"

      iex> Greeter.hello("pete")
      "Hello, pete"

  """
  @spec hello(String.t) :: String.t
  def hello(name) do
    "Hello, " <> name
  end
end
```

# ExDoc

ExDoc is an official Elixir project that **produces HTML (HyperText Markup Language and online documentation for Elixir projects** that can be found on [GitHub](https://github.com/elixir-lang/ex_doc).

## Installing

```elixir
def deps do
  [{:earmark, "~> 0.1", only: :dev},
  {:ex_doc, "~> 0.11", only: :dev}]
end
```

## Generating Documentation

```shell
$ mix deps.get   # gets ExDoc + Earmark.
$ mix docs   # makes the documentation.

Docs successfully generated.
View them at "doc/index.html".
```

# Best Practice

- Always document a module.
- If you do not intend to document a module, do not leave it blank. Consider annotating `@moduledoc false`.
- When referring to functions within module documentation, use backticks like `hello/1`.
- Use markdown within functions that will make it easier to read either via IEx or ExDoc.
