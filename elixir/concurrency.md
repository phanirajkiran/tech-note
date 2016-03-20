Concurrency
===========

The concurrency model in Elixir relies on **Actors**, a contained process that communicates with other processes through message passing.

# Processes

Processes in the ErlangVM are lightweight and run across all CPUs. While they may seem like native threads they’re simpler and it’s not uncommon to have thousands of concurrent process in an Elixir application.

The easiest way to create a new process is `spawn` which takes either an anonymous or named function. When we create a new process it returns a *Process Identifier*, or PID, to uniquely identify it within our application.

```elixir
defmodule Example do
  def add(a, b) do
    IO.puts(a + b)
  end
end

iex> pid = spawn(Example, :add, [2, 3])
5
#PID<0.80.0>
iex> Process.alive?(pid)
false
```

`self()` can be used to retrieve the PID of the current process, which is useful in message passing.

## Message Passing

The `send/2` function allows us to send messages to PIDs. To listen we use `receive` to match messages, if no match is found the execution continues uninterrupted.

```elixir
defmodule Example do
  def listen do
    receive do
      {:ok, "hello"} -> IO.puts "World"
      listen   # infinite loop
    after    # optional timeout handler
      1_000 -> IO.puts "timeout"
    end
  end
end

iex> pid = spawn(Example, :listen, [])
#PID<0.108.0>

iex> send pid, {:ok, "hello"}
World
{:ok, "hello"}
```

## Process Linking

We can link our processes using `spawn_link`. Two linked processes will receive exit notifications from one another:

```elixir
defmodule Example do
  def explode, do: exit(:kaboom)
end

iex> spawn(Example, :explode, [])
#PID<0.66.0>

iex> spawn_link(Example, :explode, [])
** (EXIT from #PID<0.57.0>) :kaboom
```

When trapping exits they will be received as a tuple message: `{:EXIT, from_pid, reason}`:

```elixir
defmodule Example do
  def explode, do: exit(:kaboom)
  def run do
    Process.flag(:trap_exit, true)
    spawn_link(Example, :explode, [])

    receive do
      {:EXIT, from_pid, reason} -> IO.puts "Exit reason: #{reason}"
    end
  end
end

iex> Example.run
Exit reason: kaboom
:ok
```

## Process Monitoring

We can use `spawn_monitor` to kept two processes informed without linking them. When we monitor a process we get a message if the process crashes, without our current process crashing or needing to explicitly trap exits.

```elixir
defmodule Example do
  def explode, do: exit(:kaboom)
  def run do
    {pid, ref} = spawn_monitor(Example, :explode, [])

    receive do
      {:DOWN, ref, :process, from_pid, reason} -> IO.puts "Exit reason: #{reason}"
    end
  end
end

iex> Example.run
Exit reason: kaboom
:ok
```

# Agents

Agents are an abstraction around background processes maintaining state. We can access them from other processes within our application and node.

```elixir
iex> {:ok, agent} = Agent.start_link(fn -> [1, 2, 3] end)
{:ok, #PID<0.65.0>}

iex> Agent.update(agent, fn (state) -> state ++ [4, 5] end)
:ok

iex> Agent.get(agent, &(&1))
[1, 2, 3, 4, 5]
```

When we name an Agent we can refer to it by that instead of it’s PID:

```elixir
iex> Agent.start_link(fn -> [1, 2, 3] end, name: Numbers)
{:ok, #PID<0.74.0>}

iex> Agent.get(Numbers, &(&1))
[1, 2, 3]
```

# Tasks

Tasks provide a way to execute a function in the background and retrieve its return value later. They can be particularly useful when handling expensive operations without blocking the application execution.

```elixir
defmodule Example do
  def double(x) do
    :timer.sleep(2000)
    x * 2
  end
end

iex> task = Task.async(Example, :double, [2000])
%Task{pid: PID<0.111.0>, ref: Reference<0.0.8.200>}

# Do some work...

iex> Task.await(task)   # blocked until finished or timeout
4000
```
