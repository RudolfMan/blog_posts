%{
  title: "Doctests with side effects",
  description: "How to write doctests for public functions that have side effects."
}
---

> **TL;DR** jump to the [conclusion](#conclusion)

At [Simplebet](https://simplebet.io) while we are doing our best to have the systems covered with integration and unit tests, recently, we've also started focusing more on "Living Documentations" mostly using [`ExDoc`](https://hexdocs.pm/ex_doc/readme.html). Filled with mermaid charts, descriptions of the service architecture and data flows it provides a better onboarding experience.

That drew my attention to the docs of "context" modules. Often those top-level public functions, meant to be called from controllers, live views and other parts of the application, happen to be impure. They might try to reach out to remote HTTP services, RabbitMQ, the DB or to other resources.

In those cases, while sometimes the `@doc`s include usage examples, they intentionally lack `iex>` prefixes, so `doctests` don't pick them up and fail. However, with the code refactorings and the system evolutions we risk having the docs mismatch the reality. Although, arguably "outdated docs are still better than no docs" we don't have to compromise!

In this post I'll show one way of "doctesting" functions that have side effects.

> If you are not familiar with `doctest`s, please take a look at the official documentation of [`ExUnit.DocTest`](https://hexdocs.pm/ex_unit/ExUnit.DocTest.html) first.


### Closer to real world example

Let's take a look at the example of interactions with an external system. In regular scenarios we tend to mock or stub those with [`Mox`](https://hexdocs.pm/mox/Mox.html).

> If you haven't yet read ["Mocks and explicit contracts"](https://dashbit.co/blog/mocks-and-explicit-contracts) by José Valim I highly encourage you to do that!

The documentation of `Mox` provides with an example of testing the module that displays the weather information.

```elixir
# humanized_weather.ex

defmodule MyApp.HumanizedWeather do
  def display_temp({lat, long}) do
    {:ok, temp} = MyApp.WeatherAPI.temp({lat, long})
    "Current temperature is #{temp} degrees"
  end

  # ...
end
```

Let's add a `@doc` with a "doctestable" example:

```elixir
# humanized_weather.ex

# ...

@doc """
Displays current temperature at the given coordinates

## Examples

    iex> MyApp.HumanizedWeather.display_temp({50.06, 19.94})
    "Current temperature is 30 degrees"
"""
def display_temp({lat, long}) do
  {:ok, temp} = MyApp.WeatherAPI.temp({lat, long})
  "Current temperature is #{temp} degrees"
end
```

Now let's add the `doctest`:

```elixir
# humanized_weather_test.exs

defmodule MyApp.HumanizedWeatherTest do
  use ExUnit.Case, async: true

  doctest MyApp.HumanizedWeather
end
```

Essentially `doctest` macro extracts examples from `@moduledoc` and `@doc` attributes into separate tests. What we want here is to "setup" some expectations for `MyApp.MockWeatherAPI`:

```elixir
defmodule MyApp.HumanizedWeatherTest do
  use ExUnit.Case, async: true
  import Mox

  doctest MyApp.HumanizedWeather

  setup :verify_on_exit!

  setup do
    expect(MyApp.MockWeatherAPI, :temp, fn _ -> {:ok, 30} end)

    :ok
  end
end
```

This should be enough to make it pass! Basically that's the whole trick - `setup` the "context" to satisfy the test example.

Suppose we want to also show an example of error response:

```elixir
# humanized_weather.ex

# ...

@doc """
Displays current temperature at the given coordinates

## Examples

    iex> HumanizedWeather.display_temp({50.06, 19.94})
    "Current temperature is 30 degrees"

    iex> HumanizedWeather.display_temp({102.06, 19.94})
    ** (RuntimeError) latitude must be in the range of -90 to 90 degrees
"""
def display_temp({lat, long}) do
  case MyApp.WeatherAPI.temp({lat, long}) do
    {:ok, temp} -> "Current temperature is #{temp} degrees"
    {:error, reason} -> raise reason
  end
end
```

With `Mox` that can be easily achieved with either checking the arguments in the body of the function or pattern matching the arguments.


```elixir
defmodule MyApp.HumanizedWeatherTest do
  use ExUnit.Case, async: true
  import Mox
  alias MyApp.HumanizedWeather

  doctest MyApp.HumanizedWeather

  setup :verify_on_exit!

  setup do
    expect(MyApp.MockWeatherAPI, :temp, fn
      {lat, _long} when lat < -90 or lat > 90 ->
        {:error, "latitude must be in the range of -90 to 90 degrees"}

      {_lat, _long} ->
        {:ok, 30}
    end)

    :ok
  end
end
```

> _By the way, notice how `alias MyApp.HumanizedWeather` in the test file allows us to call `HumanizedWeather.display_temp/1` in the doctest example without common `MyApp` namespace._

### Separate tests with isolated contexts
Let's say we are satisfied with the test example and now we want to document the other function in the same module: `display_humidity/1`

```elixir
defmodule MyApp.HumanizedWeather do

  # ...
  # ...

  @doc """
  Displays current humidity at the given coordinates

  ## Examples

      iex> HumanizedWeather.display_humidity({50.06, 19.94})
      "Current humidity is 60%"
  """
  def display_humidity({lat, long}) do
    {:ok, humidity} = MyApp.WeatherAPI.humidity({lat, long})
    "Current humidity is #{humidity}%"
  end
end
```

If we add another expectation to the same setup as:

```elixir
defmodule MyApp.HumanizedWeatherTest do
  # ...
  doctest MyApp.HumanizedWeather

  setup :verify_on_exit!

  setup do
    expect(MyApp.MockWeatherAPI, :temp, fn
      # ...
    end)

    expect(MyApp.MockWeatherAPI, :humidity, fn
      {_lat, _long} -> {:ok, 60}
    end)

    :ok
  end
end
```

It will error while verifying mock for  `display_temp/1` with `* expected MyApp.MockWeatherAPI.temp/1 to be invoked once but it was invoked 0 times`

Indeed, we don't expect `WeatherAPI.temp/1` to be called when checking humidity. So we need to separate the setup context for these functions. That could be done with the combination of [`describe/2`](https://hexdocs.pm/ex_unit/ExUnit.Case.html#describe/2) macro and one of the options `:only` or `:except` for `doctest`:

```elixir
defmodule MyApp.HumanizedWeatherTest do
  # ...

  setup :verify_on_exit!

  describe "display_temp/1" do
    doctest MyApp.HumanizedWeather, only: [display_temp: 1]

    setup do
      expect(MyApp.MockWeatherAPI, :temp, fn
        # ...
      end)

      :ok
    end
  end

  describe "display_humidity/1" do
    doctest MyApp.HumanizedWeather, only: [display_humidity: 1]

    setup do
      expect(MyApp.MockWeatherAPI, :humidity, fn
        {_lat, _long} -> {:ok, 60}
      end)

      :ok
    end
  end
end
```

Voilà! 🎉

<span id="conclusion"></span>

> #### Note {: .note}
>
> There is a bug in Elixir v1.13.3 and older versions in `doctest` macro. `ExUnit` will just ignore it when no function from the list of given to the option `:only` is found. So if, say, later we rename the function but don't update it in the test file - the test case will still pass (though the number of successful doctests will be smaller)
>
> In other words this will always pass.
> ```elixir
> doctest MyApp.HumanizedWeather, only: [not_existing_function: 0]
> ```
>
> Luckily [the fix](https://github.com/elixir-lang/elixir/pull/11684/files) is already merged.

### Conclusion

To sum up, in order to "doctest" functions that have side effects we need to "setup" a context around testing examples and, if needed, isolate via `describe` macro and specify the functions to run the `doctest` using options `:only` and/or `:except`.
