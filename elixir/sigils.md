Sigils
======

Elixir provides an alternative syntax for representing and working with literals. A sigil will start with a tilde `~` followed by a character. A list of built-in sigils include:

- `~C`: Generates a character list **with no** escaping or interpolation
- `~c`: Generates a character list **with** escaping and interpolation
- `~R`: Generates a regular expression **with no** escaping or interpolation
- `~r`: Generates a regular expression **with** escaping and interpolation
- `~S`: Generates strings **with no** escaping or interpolation
- `~s`: Generates string **with** escaping and interpolation
- `~W`: Generates a list **with no** escaping or interpolation
- `~w`: Generates a list **with** escaping and interpolation

# Char List

```elixir
iex> ~c/2 + 7 = #{2 + 7}/
'2 + 7 = 9'

iex> ~C/2 + 7 = #{2 + 7}/
'2 + 7 = #{2 + 7}'
```

# Regular Expressions

```elixir
iex> re = ~r/.*xir/
~r/.*xir/
iex> "Elixir" =~ re
true
iex> "elixir" =~ re
true
iex> "Erlang" =~ re
false
```

Elixir supports Perl Compatible Regular Expressions (PCRE), we can append `i` to the end of our sigil to turn on case sensitivity.

# Word List

```elixir
iex> ~w/i love elixir school/
["i", "love", "elixir", "school"]
```

# Creating Sigils

As an extensible programming language, Elixir allows you to easily create your own custom sigils.

```elixir
iex> defmodule MySigils do
...>   def sigil_u(string, []), do: String.upcase(string)
...> end

iex> import MySigils
nil

iex> ~u/elixir school/
ELIXIR SCHOOL
```
