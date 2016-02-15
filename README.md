# JaResource

A behaviour to reduce boilerplate in your JSON-API compliant Phoenix
controllers without sacrificing flexibility.

Exposing a resource becomes as simple as:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource
end
```

See [Usage](#usage) for more details on customizing.

## Rational

JaResource lets you focus on the data in your APIs, instead of worrying about
response status, rendering validation errors, and inserting changesets.

** DISCLAIMER: This is curretly pre-release software **

## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed by:

  1. Adding ja_resource to your list of dependencies in `mix.exs`:

        def deps do
          [{:ja_resource, "~> 0.0.1"}]
        end

  2. Ensuring ja_resource is started before your application:

        def application do
          [applications: [:ja_resource]]
        end

While not required, it is suggested to direct JaResource to what repo to use in
controllers:

    config :ja_resource,
      repo: MyApp.Repo


## Usage

For the most simplistic resources JaSerializer lets you replace hundreds of
lines of boilerplate with a simple use statement. JaResource simply defines
the standard restful controller actions for you, while providing many simple
callbacks you can optionally implement to customize behaviour.

To expose index, show, update, create, and delete of the `MyApp.Post` model
with no restrictions:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource
end
```

You can optionally prevent JaResource from implementing actions you wish to define yourself
(however there are better ways to tweak behavior):

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource, except: [:delete]

  def delete(conn, params) do
    # Custom delete logic
  end
end
```

And because JaResource is just implementing actions, you can still use plug
filters just like in normal Phoenix controllers:

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource

  plug MyApp.Authenticate when action in [:create, :update, :delete]
end
```

You are also free to define any custom actions in your controller, JaResource
will not interfere with them at all.

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource

  def publish(conn, params) do
   # Custom action logic
  end
end
```

### Changing the model exposed

By default JaResource parses the controller name to determine the model exposed
by the controller. `MyApp.UserController` will expose the `MyApp.User` model,
`MyApp.API.V1.CommentController' will expose the `MyApp.Comment` model.

This can easily be overridden by defining the `model/0` callback:

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource

  def model, do: MyApp.Models.BlogPost
end
```

### Customizing records returned

Many applications need to expose only subsets of a resource to a given user,
those they have access to or maybe just models that are not soft deleted.
JaResource allows you to define the `records/1` and `record/2`

`records/1` is used by index, show, update, and delete requests to get the base
query of records. Many/most controllers will override this:

```elixir
defmodule MyApp.V1.MyPostController do
  use MyApp.Web, :controller
  use JaResource

  def model, do: MyApp.Post
  def records(%Plug.Conn{assigns: %{user_id: user_id}}) do
    model
    |> where([p], p.author_id == ^user_id)
  end
end
```

`record/2` receives the results of `records/1` and the id param and returns a
single record for use in show, update, and delete. This is less common to
customize but may be useful if using non-id fields in the url:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def record(query, slug_as_id) do
    query
    |> MyApp.Repo.get_by(slug: slug_as_id)
  end
end
```

### 'Handle' Actions

Every action not excluded defines a default `handle_` variant which receives
pre-processed data and is expected to return an Ecto query or record. All of
the handle calls may also return a conn (including the result of a render
call).

An example of customizing the index and show actions (instead of customizing
`records/1` and `record/2`) would look something like this:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def handle_index(conn, _params) do
    case conn.assigns[:user] do
      nil -> where(Post, [p], p.is_published == true)
      u   -> Post # all posts
    end
  end

  def handle_show(conn, id) do
    Repo.get_by(Post, slug: id)
  end
end
```

### Creating and Updating

Like index and show, customizing creating and updating resources can be done
with the `handle_create/2` and `handle_update/3` actions, however if just
customizing what attributes to use, prefer `permitted_attributes/3`.

For example:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def permitted_attributes(conn, attrs, :create) do
    attrs
    |> Map.take(~w(title body type category_id))
    |> Map.merge("author_id", conn.assigns[:current_user])
  end

  def permitted_attributes(_conn, attrs, :update) do
    Map.take(attrs, ~w(title body type category_id))
  end
end
```

Note: The attributes map passed into `permitted_attributes` is a "flattened"
version including the values at `data/attributes`, `data/type` and any
relationship values in `data/relationships/[name]/data/id` as `name_id`.

#### Create

Customizing creation can be done with the `handle_create/2` function.

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def handle_create(conn, attributes) do
    Post.changeset(%Post{}, attributes, :create_and_publish)
  end
end
```

The attributes argument is the result of the `permitted_attributes` function.

If this function returns a changeset it will be inserted and errors rendered if
required. It may also return a model or validation errors for rendering
or a %Plug.Conn{} for total rendering control.

By default this will call `changeset/2` on the model defined by `model/0`.

#### Update

Customizing update can be done with the `handle_update/3` function.

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def handle_update(conn, post, attributes) do
    current_user_id = conn.assigns[:current_user].id
    case post.author_id do
      ^current_user_id -> {:error, author_id: "you can only edit your own posts"}
      _                -> Post.changeset(post, attributes, :update)
    end
  end
end
```

If this function returns a changeset it will be inserted and errors rendered if
required. It may also return a model or validation errors for rendering
or a %Plug.Conn{} for total rendering control.

The record argument (`post` in the above example) is the record found by the
`record/3` callback. If `record/3` can not find a record it will be nil.

The attributes argument is the result of the `permitted_attributes` function.

By default this will call `changeset/2` on the model defined by `model/0`.

#### Delete

Customizing delete can be done with the `handle_delete/2` function.

```elixir
def handle_delete(conn, post) do
  case conn.assigns[:user] do
    %{is_admin: true} -> super(conn, post)
    _                 -> send_resp(conn, 401, "nope")
  end
end
```

The record argument (`post` in the above example) is the record found by the
`record/2` callback. If `record/2` can not find a record it will be nil.

