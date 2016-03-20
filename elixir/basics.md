Basics
======

# Basic Types

Some basic types are:

```elixir
iex> 1          # integer
iex> 0x1F       # integer
iex> 1.0        # float
iex> true       # boolean
iex> :atom      # atom / symbol
iex> "elixir"   # string
iex> [1, 2, 3]  # list
iex> {1, 2, 3}  # tuple
```

Type checking functions including `is_integer/1`, `is_float/1` and `is_number/1` can be used to check, respectively, if an argument is an integer, a float or either.

## Atoms

- An atom is a constant whose name is their value
- Booleans `true` and `false` are also the atoms `:true` and `:false` respectively

## Binaries

In Elixir, you can define a binary using `<<>>`:

```elixir
iex> <<0, 1, 2, 3>>
<<0, 1, 2, 3>>
iex> byte_size(<<0, 1, 2, 3>>)
4
```

## Type Information

`i/1` can be used to retrieve information about the type of the value:

```elixir
iex> i 'hello'
Term
  'hello'
Data type
  List
Description
  This is a list of integers that is printed as a sequence of characters
  delimited by single quotes because all the integers in it represent valid
  ASCII characters. Conventionally, such lists of integers are referred to as
  "char lists" (more precisely, a char list is a list of Unicode codepoints,
  and ASCII is a subset of Unicode).
Raw representation
  [104, 101, 108, 108, 111]
Reference modules
  List
```

# Basic Operations

## Arithmetic

- operator `/` will always return a **float**
- `round`: get the closest integer to a given float
- `trunc`: get the integer part of a float
- division or the division remainder
  ```elixir
  iex> div(10, 3)
  3
  iex> rem(10, 3)
  1
  ```

## Boolean

Except `||`, `&&` and `!`, there are three additional operators (`and`, `or`, and `not`) whose first argument must be a boolean (`true` and `false`):

```elixir
iex> true and 42
42
iex> false or true
true
iex> not false
true
iex> 42 and true
** (ArgumentError) argument error: 42
iex> not 42
** (ArgumentError) argument error
```

## Comparison

- strict comparison of integers and floats use `===`
  ```elixir
  iex> 2 == 2.0
  true
  iex> 2 === 2.0
  false
  ```
- any two types can be compared
  ```
  number < atom < reference < functions < port < pid < tuple < maps < list < bitstring
  ```

## String Interpolation

```elixir
iex> name = "Jimmy"
iex> "Hello #{name}"
"Hello Jimmy"
```

## String Concatenation

```elixir
iex> name = "Jimmy"
iex> "Hello " <> name
"Hello Jimmy"
```
