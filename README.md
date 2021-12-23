Here's a collection of useful [Phoenix LiveView](https://github.com/phoenixframework/phoenix_live_view) tidbits. I thought that the format and easy grok-ability of asking questions would be valuable. [`LiveView`'s documentation](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html) is fantastic and thorough, but you got to read all of it to get all of it. And we all know we don't read every line of published documentation :)

The first draft was aggregated from ~10 days of Elixir Slack's `#liveview` channel. Thanks to everyone who asks questions and helps out!

## Contents 📖

- [Why is `mount/3` being called twice?](#why-is-mount3-being-called-twice)
- [I'm fetching the `current_user` in `mount` and it's being fetched... twice 😤😤](#im-fetching-the-current_user-in-mount-and-its-being-fetched-twice-)
- [I want my `LiveView` to be rendered once and only once](#i-want-my-liveview-to-be-rendered-once-and-only-once)
- [I want to upload a file!](#i-want-to-upload-a-file)
- [My form isn't tracking changes! 😠](#my-form-isnt-tracking-changes-)
- [Nothing is tracking changes!!! 😡😡😡](#nothing-is-tracking-changes-)
- [Is pagination a thing?](#is-pagination-a-thing)
- [I'm trying to redirect in a callback but it's being ignored 🤔](#im-trying-to-redirect-in-a-callback-but-its-being-ignored-)
- [No, really, my redirect isn't working in a callback 🤔🤔](#no-really-my-redirect-isnt-working-in-a-callback-)
- [Can I style live flash messages?](#can-i-style-live-flash-messages)
- [But what does my `LiveView` process state _really_ look like?](#but-what-does-my-liveview-process-state-_really_-look-like)
- [I'm using `live_action`s and my modules are getting large and unwieldly 🙁](#im-using-live_actions-and-my-modules-are-getting-large-and-unwieldly-)
- [Why can't I send a message to my `LiveComponent` process?](#why-cant-i-send-a-message-to-my-livecomponent-process)
- [Why does Phoenix 1.5 generate a `root.html.leex` if it doesn't track changes?](#why-does-phoenix-15-generate-a-roothtmlleex-if-it-doesnt-track-changes)
- [Is the `user_id` in my `socket.assigns` secure? Can it be tampered with? 👮](#is-the-user_id-in-my-socketassigns-secure-can-it-be-tampered-with-)
- [Where are my `LiveView` routes?](#where-are-my-liveview-routes)
- [My stateless `LiveComponent` is sending all my `assigns` over the wire!!](#my-stateless-livecomponent-is-sending-all-my-assigns-over-the-wire)
- [My entire user is in `Plug.Session` so I don't have to make database calls in `LiveView`. This is a good thing, right?](#my-entire-user-is-in-plugsession-so-i-dont-have-to-make-database-calls-in-liveview-this-is-a-good-thing-right)
- [Add css class to navbar item in phoenix liveview](#add-css-class-to-navbar-in-phoenix-liveview)
- [phoenix_pubsub: Broadcast from iex not trigger handle_info in liveview](https://github.com/dfang/phoenix-live-view-tips#phoenix_pubsub-broadcast-from-iex-not-trigger-handle_info-in-liveview)
- [How to remove unused dependencies from mix](https://github.com/dfang#phoenix-live-view-tips#how-to-remove-unused-dependencies-from-mix)
- [Got anything else? 🥺](#got-anything-else-)

## Why is `mount/3` being called twice?

Straight from [the docs themselves](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#c:mount/3):

> For each LiveView in the root of a template, mount/3 is invoked twice: once to do the initial page load and again to establish the live socket.

The important phrase here being "the root of the template". In this case:

```elixir
scope "/", AppWeb do
  live "/", PageLive
end
```

`PageLive`'s `mount/3` is called twice on the first navigation to `"/"`. Once you navigate to another `LiveView` with [`live_redirect`](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.Helpers.html#live_redirect/2), however, `mount/3` is called once.

Note that if you're performing some expensive operation in `mount/3`, take a look at [`connected?`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#connected?/1). A check to see if the socket is connected to defer the work is fine when it's needed.

## I'm fetching the `current_user` in `mount` and it's being fetched... twice 😤😤

Which is expected, see above :D If you are fetching it in a plug you're actually fetching it three times... the horror! To solve this, reach for [`assign_new/3`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#assign_new/3). This allows you a few niceties such as sharing connection assigns on the initial HTTP request and only setting an assign if it's not available to children components.

This, admittedly, took me a while to wrap my head around. Nothing helps like a (contrived) example!

```elixir
# in a plug or controller
conn
  |> assign(:foo, "BAR")

defmodule AppWeb.PageLive do
  use AppWeb, :live_view

  def render(assigns) do
    ~L"""
      <div>assign foo is <%= @foo %></div>
    """
  end

  def mount(_params, _session, socket) do
    {:ok, assign_new(socket, :foo, fn -> "bar" end)}
  end
end
```

Here's the logic of the `:foo` assign:

- On the first disconnected mount, `PageLive` will render `"BAR"` in place of `@foo` because the key is already available in the `conn`
- On the second connected mount, it will render `"bar"` because there are no parent assigns to pull from

Now, let's add and render a stateful `LiveComponent` and see how `assign_new` behaves:

```elixir
# in a plug or controller
conn
  |> assign(:foo, "BAR")

defmodule AppWeb.PageLive do
  use AppWeb, :live_view

  def render(assigns) do
    ~L"""
      <div>assign foo is <%= @foo %></div>
      <%= live_component(@socket, AppWeb.PageLiveChild, id: "page-live-child", bar: @bar) %>
    """
  end

  def mount(_params, _session, socket) do
    socket =
      |> assign_new(:foo, fn -> "bar" end)
      |> assign(:bar, "FOO")
    {:ok, socket}
  end
end

defmodule AppWeb.PageLiveChild do
  use Phoenix.LiveComponent

  def render(assigns) do
    ~L"""
      <div>
        <div>assign bar is <%= @bar %></div>
      </div>
    """
  end

  def mount(socket) do
    socket =
      socket
      |> assign_new(:bar, fn -> "wee" end)

    {:ok, socket}
  end
end
```

Let's go through the rendering logic for `PageLiveChild`:

- On the first disconnected mount, `PageLiveChild` will render `"FOO"` in place of `@bar` because the key is already available in socket coming in from the parent, `PageLive`
- On the second connected mount, it will render exactly same thing! That's because `:bar` is still set from the socket assign in `PageLive`

## I want my `LiveView` to be rendered once and only once

Seeing `mount/3` being called on every `live_redirect` but want it to just be called once? Render your component in `root.html.leex` like this:

```elixir
<%= live_render(@conn, AppWeb.PageLive) %>
```

Now `PageLive` will survive redirects.

## I want to upload a file!

File uploads aren't supported by `LiveView` just yet. Check out [Jon Rowe's](https://github.com/JonRowe) library [Phoenix Live View Dropzone](https://github.com/JonRowe/phoenix_live_view_dropzone).

## My form isn't tracking changes! 😠

`LiveView` can't compute diffs instead of anonymous functions, so [`form_for/4`](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#form_for/4) doesn't work. Make sure you are using [`form_for/3`](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#form_for/3).

## Nothing is tracking changes!!! 😡😡😡

Make sure you're writing code in a `html.leex` file, not `html.eex` 😇

## Is pagination a thing?

Not out of the box, but check out [joshchernoff's](https://gist.github.com/joshchernoff) very helpful [gist](https://gist.github.com/joshchernoff/e8473ed01f31e01a2153c9b621ee529f).

## I'm trying to redirect in a callback but it's being ignored 🤔

Just like [`assign/3`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#assign/3), [`redirect/2`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#redirect/2) annotates and _returns_ the updated socket. Meaning, this won't work:

```elixir
def handle_info("annotate_redirect", _, socket) do
  redirect(socket, to: Routes.some_path())
  {:noreply, socket}
end
```

But this will:

```elixir
def handle_info("annotate_redirect", _, socket) do
  socket = redirect(socket, to: Routes.some_path())
  {:noreply, socket}
end
```

Extra credit if you noticed this is also an option:

```elixir
def handle_info("annotate_redirect", _, socket) do
  {:noreply, redirect(socket, to: Routes.some_path())}
end
```

Remember: data is immutable in Elixir!

## No, really, my redirect isn't working in a callback 🤔🤔

Note that [`redirect/2`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#redirect/2) requires you take action on the provided redirect location. If you want to redirect right from the server, use [`push_patch/2`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#push_patch/2) or [`push_redirect/2`](https://hexdocs.pm/phoenix_live_view/0.12.1/Phoenix.LiveView.html#push_redirect/2).

## Can I style live flash messages?

Absolutely! In a fresh Phoenix app generated with the `--live` flag, you can see this in `templates/layout/live.html.leex`:

```elixir
<p class="alert alert-info" role="alert"
    phx-click="lv:clear-flash"
    phx-value-key="info"><%= live_flash(@flash, :info) %></p>
```

If you'd like to conditionally display it and wrap it in more complicated markup, try:

```elixir
<%= if message = live_flash(@flash, :info) do %>
  <div class="alert alert-info" role="alert" phx-click="lv:clear-flash" phx-value-key="info">
    <div class="some-class">
      <div class="another-class">
        <%= message %>
      </div>
    </div>
  </div>
<% end %>
```

## But what does my `LiveView` process state _really_ look like?

Check out [toranb's](https://github.com/toranb) quick blog post on [inspecting `LiveView` process state](https://toranbillups.com/blog/archive/2020/05/09/phoenix-liveview-process-state/).

## I'm using `live_action`s and my modules are getting large and unwieldly 🙁

Instead of separating logic via `live_action`s in the router like this:

```elixir
scope "/", AppWeb do
  live "/", PageLive, :index
  live "/edit", PageLive, :edit
  live "/foo", PageLive, :foo
end
```

Pull them out into separate, name-spaced modules:

```elixir
scope "/", AppWeb do
  live "/", PageLive.Index
  live "/edit", PageLive.Edit
  live "/foo", PageLive.Foo
end
```

## Why can't I send a message to my `LiveComponent` process?

`LiveComponent`s aren't in their own process, only `LiveView`s are. If you call `self()` in a `LiveComponent` you'll get back that `PID` of the `LiveView` parent.

## Why does Phoenix 1.5 generate a `root.html.leex` if it doesn't track changes?

It can be used to track changes later if you want to render a `LiveView` inside of it. Mainly, it's to reduce confusion 😁

## Is the `user_id` in my `socket.assigns` secure? Can it be tampered with? 👮

Is is secure. External clients have no access to it as long as your signing secrets are safe.

## Where are my `LiveView` routes?

Given:

```elixir
live "/foo/new", FooLive.New, :new
resources "/foo", FooController, only: [:index, :create, :show]
```

It would be easy to assume that `Routes.foo_path(@conn, :new)` would generate a link that would bring us to our `LiveView`. However, that's not the case. The full module name, namespace and all, will be taken to account so the path you'd want to use is actually `Routes.foo_new_path(@conn, :new)`. Remember to check `mix phx.routes` if you're having issues finding paths! All live and dead routes will be listed.

## My stateless `LiveComponent` is sending all my `assigns` over the wire!!

`update/2` merges assigns into the socket, then `render/1` is called with _all_ assigns. Thus, no change tracking occurs. If you've got a stateless `LiveComponent` with a lot of assigns, consider:

- Making it stateful by passing an `:id` to the component
- Abstract the most updated assign into a stateful component and past the rest of the assigns to a stateless component

## My entire user is in `Plug.Session` so I don't have to make database calls in `LiveView`. This is a good thing, right?

No :) It's unlikely you want the whole `%User{}` struct. There could be other metadata added to it (see [`pow`](https://github.com/danschultzer/pow)), cache busting gets thrown out the door, and you're subject to cookie overflow.

## Add css class to navbar in Phoenix Liveview
https://elixirforum.com/t/conn-request-path-doesnt-change-in-header/44424/5?u=dfang

## phoenix_pubsub: Broadcast from iex not trigger handle_info in liveview
https://elixirforum.com/t/broadcast-from-iex-not-working/43273/4?u=dfang

## How to remove unused dependencies from mix
https://bartoszgorka.com/clear-mix-lock

## Got anything else? 🥺

Check out [Awesome Phoenix Liveview](https://github.com/beam-community/awesome-phoenix-liveview).
