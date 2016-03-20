Testing
=======

# ExUnit

Before we can run our tests we need to start ExUnit with `ExUnit.start()`, this is most commonly done in `test/test_helper.exs`.

`refute` is to `assert` as `unless` is to if. Use `refute` when you want to ensure a statement is always false.

```elixir
defmodule ExampleTest do
  use ExUnit.Case
  doctest Example

  test "the truth" do
    assert 1 + 1 == 2
  end
end
```

We can run our project’s tests with `mix test`.

# Test Setup

In some instances it may be necessary to perform setup before our tests. To accomplish this we can use the `setup` and `setup_all` macros. `setup` will be run before each test and `setup_all` once before the suite. It is expected that they will return a tuple of `{:ok, state}`, the state will be available to our tests.

```elixir
defmodule ExampleTest do
  use ExUnit.Case
  doctest Example

  setup_all do
    {:ok, number: 2}
  end

  test "the truth", state do
    assert 1 + 1 == state[:number]
  end
end
```
