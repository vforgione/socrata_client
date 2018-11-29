# Elixir Socrata Client

This library is intended to be as transparent a wrapper as possible for
the Socrata SODA 2.1+ API.

## Installation

This library is available via Hex. To add it as a dependency to your
application, add the following to your `mix.exs` file:

```elixir
defp deps do
  [
    {:socrata, ">= 0.0.0"}
  ]
end
```

## Configuration

There are two optional configuration values that you can supply via your
application's `config/config.exs` file:

```elixir
config :socrata,
  domain: "example.com",
  app_token: "blah blah blah"
```

Using the `domain` config sets a default Socrata domain for all of your
requests. This can be overwritten when calling `Socrata.Client.new/3` if
you need a one off connection to another Socrata deployment.

Using the `app_token` add the `X-App-Token` header to all of your requests.
Having a token greatly increases your rate limit. For more information about
tokens, see the
[Socrata App Tokens docs](https://dev.socrata.com/docs/app-tokens.html).

## Reading Data from the API

There are two endpoints that the `Socrata.Client` module will work with:

- the _views_ endpoint to get data set metadata
- the _resources_ endpoint to get data set records

### Metadata Client Example

```elixir
alias Socrata.Client

%HTTPoison.Response{body: body} =
  Client.new("data.cityofchicago.org")
  |> Client.get_view("yama-9had")

details = Jason.decode!(body)
Map.keys(details)

# ["oid", "publicationAppendEnabled", "category", "numberOfComments",
#  "createdAt", "attribution", "hideFromDataJson", "query", "id", "tableAuthor"
#  "rights", "tableId", "attributionLink", "owner", "viewCount", "grants",
#  "downloadCount", "flags", "publicationGroup", "name", "averageRating",
#  "publicationDate", "hideFromCatalog", "provenance", "totalTimesRated",
#  "description", "metadata", "viewLastModified", "rowsUpdatedAt", "rowsUpdatedBy",
#  "viewType", "newBackend", "publicationStage", "tags", "columns"]
```

### Getting Records as JSON

By default, calling `Socrata.Client.get_records/4` will use the `.json` API
syntax and the response body will be encoded JSON.

```elixir
alias Socrata.{Client, Query}

client = Client.new("data.cityofchicago.org")

query =
  Query.new("yama-9had")
  |> Query.limit(2)

%HTTPoison.Response{body: body} = Client.get_records(client, query)
records = Jason.decode!(body)
length(records)

# 2
```

### Getting Records as CSV

You can provide a third argument to `Socrata.Client.get_records/4` to specify
the API response syntax. In this case, we'll set it to `"csv"` and get back
an encoded CSV document in the response body.

```elixir
alias Socrata.{Client, Query}

client = Client.new("data.cityofchicago.org")

query =
  Query.new("yama-9had")
  |> Query.limit(2)

%HTTPoison.Response{body: body} = Client.get_records(client, query, "csv")
{:ok, stream} = StringIO.open(body)

records =
  IO.binstream(stream, :line)
  |> CSV.decode!(headers: true)
  |> Enum.map(& &1)

length(records)
# 2
```

### Getting Records as TSV

You can provide a third argument to `Socrata.Client.get_records/4` to specify
the API response syntax. In this case, we'll set it to `"tsv"` and get back
an encoded TSV document in the response body.

```elixir
alias Socrata.{Client, Query}

client = Client.new("data.cityofchicago.org")

query =
  Query.new("yama-9had")
  |> Query.limit(2)

%HTTPoison.Response{body: body} = Client.get_records(client, query, "tsv")
{:ok, stream} = StringIO.open(body)

records =
  IO.binstream(stream, :line)
  |> CSV.decode!(separator: ?\\t, headers: true)
  |> Enum.map(& &1)

length(records)
# 2
```

### Getting Records as GeoJSON

You can provide a third argument to `Socrata.Client.get_records/4` to specify
the API response syntax. In this case, we'll set it to `"geojson"` and get
back an encoded GeoJSON document in the response body.

```elixir
alias Socrata.{Client, Query}

client = Client.new("data.cityofchicago.org")

query =
  Query.new("yama-9had")
  |> Query.limit(2)

%HTTPoison.Response{body: body} = Client.get_records(client, query, "geojson")

%{"crs" => _, "type" => "FeatureCollection", "features" => records} =
  Jason.decode!(body)

length(records)
# 2
```

### Passing HTTPoison Options

The fourth parameter of `Socrata.Client.get_records/4` is a keyword list of
options that are directly dumped to the call to `HTTPoison.get!/3` under the
hood.

By doing this, the library hands over full control of the request/response
life cycle to you. By default it sends the request as a standard, synchronous
blocking call that gets a complete response object.

```elixir
alias Socrata.{Client, Query}

client = Client.new("data.cityofchicago.org")

query =
  Query.new("yama-9had")
  |> Query.limit(2)

%HTTPoison.AsyncResponse{id: id} = Client.get_records(client, query, "json", stream_to: self())
is_reference(id)

# true
```
