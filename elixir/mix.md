Mix
===

# New Projects

`mix new` command can generate our project’s folder structure and necessary boilerplate.

```shell
$ mix new hello
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/hello.ex
* creating test
* creating test/test_helper.exs
* creating test/hello_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd hello
    mix test

Run "mix help" for more commands.
```

# Compilation

To compile a mix project, run `mix compile` in our base directory.

When we compile a project mix creates a `_build` directory for our artifacts. If we look inside `_build` we will see our compiled application: `hello.app`.

# Interactive

With our application compiled we can start a new iex session by running `iex -S mix`. Starting iex in this way will load your application and dependencies into the current runtime.

# Manage Dependencies

To add a new dependency we need to first add it to our `mix.exs` in the `deps` section:

```elixir
def deps do
  [{:phoenix, "~> 0.16"},
   {:phoenix_html, "~> 2.1"},
   {:cowboy, "~> 1.0", only: [:dev, :test]},   # dev/test only
   {:slim_fast, ">= 0.6.0"}]
end
```

To fetch the dependencies, run `mix deps.get`.

# Environments

Out of the box mix works with three environments:

- `:dev`: The default environment
- `:test`: Used by `mix test`
- `:prod`: Used when we ship our application to production

The current environment can be accessed using `Mix.env`, and the environment can be changed via the `MIX_ENV` environment variable:

```shell
$ MIX_ENV=prod mix compile
```
