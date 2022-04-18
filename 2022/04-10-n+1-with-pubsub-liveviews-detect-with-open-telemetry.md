%{
  title: "N+1 problem by misusing Phoenix PubSub and how to spot it with OpenTelemetry.",
  description: "Phoenix LiveView is awesome! Accompanied with Phoenix PubSub it provides the superpower for building interactive real-time UX. But \"with great power comes great responsibility\". In this article we'll take a look at the problem and how to detect it with OpenTelemetry."
}
---
> #### TL:DR {: .note}
>
> The problem may occur when each PubSub subscriber does an expensive operation based on payload once an event is received that could have been done prior to broadcasting the message. Instrumentation with tracing telemetry can help to detect those.

Lately I've been playing more with [Phoenix LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html) and [Surface-UI](https://surface-ui.org). I enjoy building rich UI with almost no custom JS and getting more used to thinking differently about interactivity, when the state is pushed to the user from the Back-End not upon the request from the client, but upon the data update. I noticed a potential place where some sort of a N+1 problem, that I haven't dealt with before, could appear if the developer is not careful (as with any types of N+1 problems).

During the development of the LiveView UI, to quickly test the whole result, sometimes I just open 2 browser windows on the same view, change the data and see the broadcasted message pushed down to clients that didn't interact with the UI but receive an update. And that's when the hidden problem can occur.

### A sample app with LiveView and PubSub

Let's take a look at the example. Say, we put together a service for "ACME" to track orders and their statuses. The order status is either "placed", "shipped" or "delivered". All connected users should see the update once it happens.

Let's create a phoenix app:
```shell
$ mix phx.new acme
```
and generate "Orders" context.
```shell
$ mix phx.gen.live Orders Order orders account_id:integer status:integer
```
That task would generate for us a context with liveview to do minimal CRUD operations. Notice that status is an `:integer`. We want to use `Ecto.Enum` to map the status code to the "name". Change the field type in the schema definition in "lib/acme/orders/order.ex"

```elixir
...
schema "orders" do
  field :account_id, :integer
  field :status, Ecto.Enum, values: [placed: 1, shipped: 2, delivered: 3]

  timestamps()
end
...
```
And let's change the generated form input box from type "number" to "select". In "lib/acme_web/live/order_live/form_component.html.heex" change the status field:

```elixir
...
<%= label f, :status %>
<%= select f, :status, Ecto.Enum.values(Orders.Order, :status), prompt: "Order status" %>
<%= error_tag f, :status %>
```
[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/17ad5cbc0ff68040b0d8f55fb70483dafe909ce3)

Now let's add some minimal `PubSub` to broadcast the updates from Orders context. Change function `update_order/2` in "/lib/acme/orders.ex" to:
```elixir
def update_order(%Order{} = order, attrs) do
  order_changeset = Order.changeset(order, attrs)

  with {:ok, order} <- Repo.update(order_changeset) do
    Phoenix.PubSub.broadcast(Acme.PubSub, "orders:#{order.id}", {:order_updated, order})
    {:ok, order}
  end
end
```
We also need to subscribe to that topic from `OrderLive.Show`. Inside `handle_params/3` callback add:
```elixir
if connected?(socket), do: Phoenix.PubSub.subscribe(Acme.PubSub, "orders:#{id}")
```
where `id` is order id.

And add `handle_info/2` callback to reassign new order once broadcasted:
```elixir
def handle_info({:order_updated, order}, socket) do
  {:noreply, assign(socket, :order, order)}
end
```
[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/3471f617a91a9a1a5a6c04595b8579a8c7532130)

Now when the order is updated all users connected to `OrderLive.Show` will see the changes.

### The tricky part.

It's important to keep in mind that LiveView spawns an erlang process for each connected client.

Therefore, since each of them receive the update, `handle_info/2` callback performs individually for each client. Hence, we should avoid any expensive operation, such as DB calls, remote service calls etc. in that callback.

Just don't do anything expensive in there. Sounds simple, right? However, in practice in larger systems when requirements change and multiple modules are already subscribed to particular updates - some of them eventually might require some extra info or side effects when update happens.

For example, imagine that in the "ACME" a new requirement, if an order is shipped - we want to load and show the information about the shipping agency that takes care of the particular delivery. And there could be a temptation to add that code to liveview, because on first glance it seems like we just want to "show that info to viewers".
```elixir
def handle_info({:order_updated, order}, socket) do
  order = preload_shipping_agency(order)

  {:noreply, assign(socket, :order, order)}
end
```

It might work fine and when testing locally. We won't notice any issue, but it will try to preload shipping agency `N` times, where `N` is the number of connected clients.

### How to detect the problem?

Overall it's a good practice to have the project well instrumented and setup with observability. Similar to any kind of N+1 problems, this can be spotted relatively easily with tracing.

Let's see how we can instrument it with [OpenTelemetry](https://opentelemetry.io/) and, for, at least, local development, observe traces in [OpenZipkin](https://zipkin.io/).

In a separate terminal tab let's start zipkin server:
```shell
$ docker run -d -p 9411:9411 openzipkin/zipkin
```
Now let's define required dependencies in "mix.exs":
```elixir
defp deps do
  [
    ...
    {:opentelemetry, "~> 1.0"},
    {:opentelemetry_api, "~> 1.0"},
    {:opentelemetry_ecto, "~> 1.0"}
    {:opentelemetry_phoenix, "~> 1.0"},
    {:opentelemetry_zipkin, "~> 1.0"},
  ]
```
setup default tracers for phoenix and ecto in "application.ex":
```elixir
def start(_type, _args) do
  OpentelemetryPhoenix.setup()
  OpentelemetryEcto.setup([:acme, :repo])
  ...
end
```
And let's also configure exporter to zipkin in either "config/dev.exs" or "config/config.exs" add:
```elixir
config :opentelemetry, :processors,
  otel_batch_processor: %{
    exporter: {:opentelemetry_zipkin, %{address: 'http://localhost:9411/api/v2/spans'}}
  }
```

[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/e83443875926b1a79a4c0af406ea5db1ff3e28d8)

Now let's create traces.

First we need to require `OpenTelemetry.Tracer` as it provides with macros to create spans.
```elixir
defmodule AcmeWeb.OrderLive.Show do
  use AcmeWeb, :live_view

  require OpenTelemetry.Tracer

  ...

```
and update `handle_info/2` with:
```elixir
def handle_info({:order_updated, order}, socket) do
  span_opts = %{attributes: %{user: inspect(self())}}

  OpenTelemetry.Tracer.with_span "order_live.show:order_updated", span_opts do
    # expensive operation like DB call, service call.. etc.
    Process.sleep(70)

    {:noreply, assign(socket, :order, order)}
  end
end
```
For the purpose of the example we pretend that each viewer-user is represented by its live_view pid. We add `:user` attribute to differentiate between spans created for each user.

[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/efb4cea542578c8e32fd6d6fcb4ebe47b0bb0446)

Set the env variable `OTEL_SERVICE_NAME` and start the server like:
```shell
OTEL_SERVICE_NAME=acme iex -S mix phx.server
```

Now if we try to trigger a broadcast to multiple connected windows - we should see in Zipkin UI multiple spans created.

However, as you notice, spans are not part of the same trace, because the span context by default is local for the erlang process, while, as mentioned earlier, each LiveView connection lives in its own process. In order to overcome this we could start a span when an update happens, and pass its context as part of the broadcasted event message. In `Acme.Orders` update `update_order/2` to be like:
```elixir
def update_order(%Order{} = order, attrs) do
  OpenTelemetry.Tracer.with_span "orders:update_order" do
    order_changeset = Order.changeset(order, attrs)

    with {:ok, order} <- Repo.update(order_changeset) do
      ctx = OpenTelemetry.Tracer.current_span_ctx()
      Phoenix.PubSub.broadcast(Acme.PubSub, "orders:#{order.id}", {:order_updated, order, ctx})

      {:ok, order}
    end
  end
end
```
And in the `OrderLive.Show.handle_info/2` set the span context:
```elixir
  def handle_info({:order_updated, order, ctx}, socket) do
    OpenTelemetry.Tracer.set_current_span(ctx)

    opts = %{attributes: %{user: inspect(self())}}

    OpenTelemetry.Tracer.with_span "order_live.show:order_updated", opts do
      # expensive operation like DB call, service call.. etc.
      Process.sleep(70)

      {:noreply, assign(socket, :order, order)}
    end
  end
```
[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/8f26d9687d56ec7a308c596b7520cf075a2fe43b)

Now when we trigger broadcast, a single trace with multiple spans occurs in Zipkin UI.

![N+1 problem Phoenix LiveView PubSub OpenTelemetry](/images/zipkin_n_1.png)

That's it in a nutshell!

The code with calls to OpenTelemetry everywhere looks boilerplaty, but this should be enough to make it work.

### Thoughts on how to make it beautiful.

One thing comes to mind is to create a helper for broadcasting, that would wrap the event into a struct where it would also pass the OpenTelemetry context.
Let's create a module `Acme.PubSub`:
```elixir
defmodule Acme.PubSub do
  defmodule Event do
    defstruct [:message, :span_ctx]
  end

  def broadcast(topic, message) do
    require OpenTelemetry.Tracer

    OpenTelemetry.Tracer.with_span "acme.pubsub:broadcast" do
      event = %Event{message: message, span_ctx: OpenTelemetry.Tracer.current_span_ctx()}
      Phoenix.PubSub.broadcast(__MODULE__, topic, event)
    end
  end
end
```
New let's update `Orders.update_order/2` to use our new custom broadcast function:
```elixir
def update_order(%Order{} = order, attrs) do
  order_changeset = Order.changeset(order, attrs)

  with {:ok, order} <- Repo.update(order_changeset) do
    Acme.PubSub.broadcast("orders:#{order.id}", {:order_updated, order})

    {:ok, order}
  end
end
```

Now, for the subscriber side we would want to "unwrap" that struct, set a context and pass the message down to the module to handle it normally. That could be done via code injection:

Let's add to `Acme.PubSub` macro `__using__`:

```elixir
defmodule Acme.PubSub do
  ...

  defmacro __using__(_opts) do
    quote do
      def handle_info(%Acme.PubSub.Event{} = event, socket) do
        OpenTelemetry.Tracer.set_current_span(event.span_ctx)
        handle_info(event.message, socket)
      end
    end
  end
end
```

and `use Acme.PubSub` in `OrderLive.Show`

```elixir
defmodule AcmeWeb.OrderLive.Show do
  use AcmeWeb, :live_view
  use Acme.PubSub
  ...
end
```
Now we could remove all OpenTelemetry relate boilerplate from `handle_info/2`:
```elixir
defmodule AcmeWeb.OrderLive.Show do
  ...

  @impl true
  def handle_info({:order_updated, order}, socket) do
    # expensive operation like DB call, service call.. etc.
    # for the example we'll do a function call that results in DB query, which is already instrumented via OpenTelemetryEcto
    order = Orders.get_order!(order.id)

    {:noreply, assign(socket, :order, order)}
  end

  ...
end
```
[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/bb4b16f56ac8b1db84d42c5757754f44a7110e3c)

Much cleaner, isn't it? However, some more information about what module causes N+1 and where broadcast was made from would be helpful to find out the source of the problem.

We could "inject" that info into spans and still have relatively clean code using macros:

Let's convert our `Acme.PubSub.broadcast/2` into macro:
```elixir
defmodule Acme.PubSub do
  ...

  defmacro broadcast(topic, message) do
    quote do
      current_function = Acme.PubSub.current_function(__ENV__)
      Acme.PubSub.broadcast_from_function(unquote(topic), unquote(message), current_function)
    end
  end

  def broadcast_from_function(topic, message, function_name) do
    require OpenTelemetry.Tracer

    opts = %{attributes: %{broadcaster: function_name}}

    OpenTelemetry.Tracer.with_span "acme.pubsub:broadcast", opts do
      event = %Event{message: message, otel_ctx: OpenTelemetry.Tracer.current_span_ctx()}
      Phoenix.PubSub.broadcast(__MODULE__, topic, event)
    end
  end

  def current_function(env) do
    {fun, arity} = env.function
    "#{inspect(env.module)}.#{fun}/#{arity}"
  end
end
```

This macro `broadcast/3` gets the current function name from where it's used, and calls `Acme.PubSub.broadcast_from_function/3` which does the same as the previous `broadcast/2` function did but also sets the `:broadcaster` attribute.

Because `Acme.PubSub.broadcast/2` now is a macro we just need to `require` the `Acme.PubSub` in `Acme.Orders` before using it:

```elixir
defmodule Acme.Orders do
  ...
  require Acme.PubSub
  ...
end
```

Similarly we could add info about the module that handles the broadcasted event. Let's add a span with `:handler` attribute to injecting `handle_info/2` that would suggest the name of the handler process:

```elixir
defmodule Acme.PubSub do
  ...
  defmacro __using__(_opts) do
    quote do
      def handle_info(%Acme.PubSub.Event{} = event, socket) do
        require OpenTelemetry.Tracer

        OpenTelemetry.Tracer.set_current_span(event.span_ctx)
        opts = %{attributes: %{handler: inspect(__ENV__.module)}}

        OpenTelemetry.Tracer.with_span "acme:handle_event", opts do
          handle_info(event.message, socket)
        end
      end
    end
  end
  ...
end
```
[See diffs.](https://github.com/RudolfMan/acme_liveview_pubsub_opentelemetry/commit/0bd9edbf5dae1c3e9774f48ebfde29407dd9bb2d)

![N+1 problem Phoenix LiveView PubSub OpenTelemetry Trace With Attributes](/images/zipkin_n_1_with_attrs.png)

Voilà! 🎉

Thank you for reading!
