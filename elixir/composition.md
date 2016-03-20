Composition
===========

# Modules

Modules are the best way to organize functions into a namespace. It is possible to nest modules in Elixir, allowing you to further namespace your functionality:

```elixir
defmodule Example.Greetings do
  def morning(name) do
    "Good morning #{name}."
  end

  def evening(name) do
    "Good night #{name}."
  end
end

iex> Example.Greetings.morning "Jimmy"
"Good morning Jimmy."
```

## Module attributes

Module attributes are most commonly used as constants in Elixir. There are some reserved attributes, including `moduledoc`, `doc`, `behaviour`...etc.

```elixir
defmodule Example do
  @greeting "Hello"

  def greeting(name) do
    ~s(#{@greeting} #{name}.)
  end
end
```

# Structs

Structs are special maps with a defined set of keys and default values. It must be defined within a module, which it takes its name from. It is common for a struct to be the only thing defined within a module.

```elixir
defmodule Example.User do
  defstruct name: "Jimmy", roles: []
end
```

To create some structs:

```elixir
iex> %Example.User{}
%Example.User{name: "Jimmy", roles: []}

iex> %Example.User{name: "Steve"}
%Example.User{name: "Steve", roles: []}

iex> %Example.User{name: "Steve", roles: [:admin, :owner]}
%Example.User{name: "Steve", roles: [:admin, :owner]}
```

We can update our struct just like we would a map:

```elixir
iex> steve = %Example.User{name: "Steve", roles: [:admin, :owner]}
%Example.User{name: "Steve", roles: [:admin, :owner]}
iex> jimmy = %{steve | name: "Jimmy"}
%Example.User{name: "Jimmy", roles: [:admin, :owner]}
```

Also, we can match structs against maps:

```elixir
iex> %{name: "Jimmy"} = jimmy
%Example.User{name: "Jimmy", roles: [:admin, :owner]}
```

# Composition

## alias

Allows us to alias module names, used quite frequently in Elixir code:

```elixir
defmodule Sayings.Greetings do
  def basic(name), do: "Hi, #{name}"
end

defmodule Example do
  alias Sayings.Greetings

  def greeting(name), do: Greetings.basic(name)
end

# Without alias
defmodule Example do
  def greeting(name), do: Saying.Greetings.basic(name)
end
```

To alias multiple modules at once, use `alias Sayings.{Greetings, Farewells}`.

We can use `:as` option to alias to a different name entirely:

```elixir
defmodule Example do
  alias Sayings.Greetings, as: Hi

  def print_message(name), do: Hi.basic(name)
end
```

## import

If we want to import functions and macros rather than aliasing the module we can use `import`:

```elixir
iex> last([1, 2, 3])
** (CompileError) iex:9: undefined function last/1
iex> import List
nil
iex> last([1, 2, 3])
3
```

By default all functions and macros are imported but we can filter them using the `:only` and `:except` options:

```elixir
iex> import List, only: [last: 1]   # last/1 function
iex> import List, except: [last: 1]
```

In addition to the **name/arty** pairs there are two special atoms, `:functions` and `:macros`, which import only functions and macros respectively:

```elixir
import List, only: :functions
import List, only: :macros
```

## require

Requiring a module ensures that it is compiled and loaded. This is most useful when we need to access a module’s macros:

```elixir
defmodule Example do
  require SuperMacros

  SuperMacros.do_stuff
end
```

## use

Uses the module in the current context. This is particularly useful when a module needs to perform some setup. By calling `use` we invoke the `__using__` hook within the module, providing the module an opportunity to modify our existing context:

```elixir
defmodule MyModule do
  defmacro __using__(opts) do
    quote do
      import MyModule.Foo
      import MyModule.Bar
      import MyModule.Baz

      alias MyModule.Repo
    end
  end
end
```

Generally speaking, the following module:

```elixir
defmodule Example do
  use Feature, option: :value
end
```

is compiled into

```elixir
defmodule Example do
  require Feature
  Feature.__using__(option: :value)
end
```
