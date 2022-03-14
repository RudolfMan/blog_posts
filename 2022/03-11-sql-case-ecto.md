%{
  title: "SQL CASE clause with Ecto",
  description: "We will explore the ways to send `SQL CASE` conditional expression to Postgres using Ecto. Also we will touch elixir macros and get familiar with `unquote_splicing/1`."
}
---
**TL;DR** jump straight to the implementation of specialized variant of [`sql_case/2` macro](#sql_case/2).

[`Ecto`](https://hexdocs.pm/ecto/Ecto.html) is one of the best things happened to elixir. It provides us with tools for language integrated query and if a query is not possible to represent using standard Ecto's syntax, there is [`Ecto.Query.fragment/1`](https://hexdocs.pm/ecto/Ecto.Query.API.html#fragment/1) to send the expression to the DB.

`SQL CASE` is one of those expressions that's not available by default in Ecto so let's see how we could implement it:

The basic SQL syntax looks like:
```sql
CASE WHEN condition THEN result
     [WHEN ...]
     [ELSE result]
END
```

Suppose the simple scenario. We have a table `movies` and, say, we would like to query the titles and types of movies based on the length requirements set by American Film Institute and British Film Institute.
```table
 title         | length
---------------+--------
 The Big Shave |      6
 Goodfellas    |    146
```
According to those organizations, motion pictures with running time less than 40 minutes are considered short films otherwise it can be considered as feature-length film.
```sql
SELECT title,
       CASE WHEN length > 40 THEN 'feature film'
            ELSE 'short film'
       END
    FROM movies;
```
With `Ecto.Query.fragment` formatted with `mix format` elixir code could look like:
```elixir
import Ecto.Query

from(m in "movies",
  select:
    {m.title,
     fragment(
       """
       CASE WHEN ? > 40 THEN 'feature film'
            ELSE 'short film'
       END
       """,
       m.length
     )}
)
```

Pretty straight forward, isn't it?

However, later we learn that the Screen Actors Guild has a different requirement. Instead of 40 minutes, they require the film to be at least 60 minutes long.

To be able to satisfy all those organizations we can make the query a little bit more dynamic. Both conditional expression and values could also be bound on fragment building, as:
```elixir
required_length = 60

fragment(
  """
  CASE WHEN ? THEN ?
       ELSE ?
  END
  """,
  m.length > ^required_length,
  "feature film",
  "short film"
)
```

Nice! Thought it could be even nicer!

### Macros

As described in the docs for [`fragment/1`](https://hexdocs.pm/ecto/Ecto.Query.API.html#fragment/1-defining-custom-functions-using-macros-and-fragment) we can define a custom function using macros!

```elixir
defmodule CustomFunctions do
  defmacro case_when(condition, then, otherwise) do
    quote do
      fragment(
        """
        CASE WHEN ? THEN ?
             ELSE ?
        END
        """,
        unquote(then),
        unquote(otherwise)
      )
    end
  end
end
```

And could be used as:
```elixir
import Ecto.Query
import CustomFunctions

from(m in "movies",
  select:
    {m.title, case_when(m.length > 60, "feature film", "short film")}
)
```

> #### Note {: .note}
>
> I'm not encouraging to prefer macros whenever possible. Overall **in elixir macros should only be used as a last resort.**
>
> Custom functions for Ecto is one of the exceptional cases when I think macros are more acceptable.

Now, let's say, for each film we also have the rating symbol according to Motion Picture Association, but we want to print out the name of the rating rather than a symbol.

```table
 title                       | rating
-----------------------------+--------
 The Hunchback of Notre Dame |      G
 Coraline                    |     PG
 Black Swan                  |      R
 Pulp Fiction                |  NC-17
```
In SQL it would be something like:
```sql
SELECT title,
       CASE WHEN rating == 'G' THEN 'General Audiences'
            WHEN rating == 'R' THEN 'Restricted'
            ...
       END
    FROM movies;
```

The idea for our custom function is to generate the "template" sql expression with `?` in places where values would be bound. Bounded values could belong to either "`WHEN`-condition", "`THEN`-result" or "`ELSE`-result"

Keywords seem to be a perfect fit for such exercise.

Suppose a function:
```elixir
sql_case(when: "R", then: "Restricted", else: "General Audience")
```
and the list can go on and on
```elixir
sql_case(
  when: rating == "R", then: "Restricted",
  when: rating == "G", then: "General Audience",
  when: rating == "PG", then: "Parental Guidance Suggested",
  when: rating == "NC-17", then: "Clearly Adult"
)
```
We could take keys and iterate over the list to build a template like:
```elixir
template =
  params
  |> Keyword.keys()
  |> Enum.reduce("CASE ", fn
    :when, template -> template <> "WHEN ? "
    :then, template -> template <> "THEN ? "
    :else, template -> template <> "ELSE ? "
  end) <> "END"
```
That would build a "template" string of the query for the fragment to "`unquote`":
```elixir
"CASE WHEN ? THEN ? WHEN ? THEN ? ... ELSE ? END"
```

And the arguments to bind would be the list of values
```elixir
args = Keyword.values(params)
```
The signature for fragment is
```elixir
fragment(template, [arg1, [arg2, [arg3, ..]]])
```
So at compile time we need to know the number of arguments to bind. Luckily special form [`unquote_splicing/1`](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#unquote_splicing/1) can help us to achieve that:
```elixir
quote do
  fragment(unquote(template), unquote_splicing(args))
end
```

The resulting macro would look like:
```elixir
defmacro sql_case(params) do
  template =
    params
    |> Keyword.keys()
    |> Enum.reduce("CASE ", fn
      :when, template -> template <> "WHEN ? "
      :then, template -> template <> "THEN ? "
      :else, template -> template <> "ELSE ? "
    end) <> "END"

  args = Keyword.values(params)

  quote do
    fragment(unquote(template), unquote_splicing(args))
  end
end
```
And we could use it as:
```elixir
sql_case(
  when: rating == "R", then: "Restricted",
  when: rating == "G", then: "General Audience",
  when: rating == "PG", then: "Parental Guidance Suggested",
  when: rating == "NC-17", then: "Clearly Adult",
)
```
_Notice how that `SQL CASE` expression looks like `cond do` in elixir._  Each conditional expression is just comparing `rating` with the symbol. In `SQL` for these types of a situation there is a specialized variant of the general form:
```sql
CASE expression
    WHEN value THEN result
    [WHEN ...]
    [ELSE result]
END
```
It's more like elixir's `case do`.
With that variant our query could look like:
```sql
CASE rating
  WHEN 'G' THEN 'General Audiences'
  WHEN 'R' THEN 'Restricted'
    ...
END
```

We could have the same: as
```elixir
#                 ↓↓↓↓↓
defmacro sql_case(value, params) do
  template =            #     ↓↓
    Enum.reduce(params, "CASE ? ", fn
      :when, template -> template <> "WHEN ? "
      :then, template -> template <> "THEN ? "
      :else, template -> template <> "ELSE ? "
    end) <> "END"

  #       ↓↓↓↓ 
  args = [value | Keyword.values(params)]

  quote do
    fragment(unquote(template), unquote_splicing(args))
  end
end

# use
sql_case(rating,
  when: "R", then: "Restricted",
  when: "G", then: "General Audience",
  when: "PG", then: "Parental Guidance Suggested",
  when: "NC-17", then: "Clearly Adult",
)
```

### Formatting

Once we run `mix format` - it quickly transforms the code into:
```elixir
sql_case(rating,
  when: "R",
  then: "Restricted",
  when: "G",
  then: "General Audience",
  when: "PG",
  then: "Parental Guidance Suggested",
  when: "NC-17",
  then: "Clearly Adult",
)
```
Which is less readable than I wanted. However, if we had pairs of `when-then` wrapped into tuples or lists formatter would have left them on the same line:

```elixir
sql_case(m.rating, [
  [when: "G", then: "General Audiences"],
  [when: "R", then: "Restricted"],
  [when: "PG", then: "Parental Guidance Suggested"],
  [when: "NC-17", then: "Clearly Adult"]
])
```

While we could flatten this list before building the template sql query string, reducing over this type of a list could provide us with some "validation" that every `:when` is followed by `:then`. Otherwise, it will not compile!
```elixir
template =
  Enum.reduce(params, "CASE ? ", fn
    [when: _condition, then: _result], template -> template <> "WHEN ? THEN ? "
    [else: _result], template -> template <> "ELSE ? "
  end) <> "END"
```
And for arguments we could use a comprehension to extract nested values:
```elixir
args = for pair <- params, {_, arg} <- pair, do: arg
args = [value | args]
```
<span id="sql_case/2"></span>
and this will give us the macro:

```elixir
defmodule CustomFunctions do
  defmacro sql_case(value, params) do
    template =
      Enum.reduce(params, "CASE ? ", fn
        [when: _condition, then: _result], template -> template <> "WHEN ? THEN ? "
        [else: _result], template -> template <> "ELSE ? "
      end) <> "END"

    args = for pair <- params, {_, arg} <- pair, do: arg
    args = [value | args]

    quote do
      fragment(unquote(template), unquote_splicing(args))
    end
  end
end
```
That could be used as:
```elixir
import Ecto.Query
import CustomFunctions

from(m in "movies",
  select:
    {m.title,
     sql_case(m.rating, [
       [when: "G", then: "General Audiences"],
       [when: "R", then: "Restricted"],
       [when: "PG", then: "Parental Guidance Suggested"],
       [when: "NC-17", then: "Clearly Adult"],
       [else: m.rating]
     ])}
```

Awesome! Now this macro can also be used in combination with other functions and macros from Ecto DSL, for example with `update:`
```elixir
  from(m in "movies",
    update: [
      set: [
        rating_descritpion:
          sql_case(m.rating, [
            [when: "G", then: "General Audiences"],
            [when: "R", then: "Restricted"],
            ...
          ])
      ]
    ]
  )
```

-----
For more ideas and inspiration also check out this amazing blog post ["I can do all things through PostgreSQL"](https://supersimple.org/blog/all-things) by [Todd Resudek](https://twitter.com/sprsmpl) _(if you haven't yet)_.
