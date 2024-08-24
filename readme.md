# API proposal for ParadeDB extension for Ecto
My aim here's to figure out what an API for ParadeDB in Ecto should look like, and determine what it'd take to implement.

#### What `0.1` should cover:
* pg_search queries via ParadeQL strings.
* pg_search queries via the "advanced" query objects.
* pg_search index mapping.
* Dense vector search, sparse vector search, hybrid search.

#### What `0.1` shouldn't cover:
* `pg_analytics` is still WIP. The foreign data wrappers are generally created in a few lines, and are otherwise queried like normal postgres tables.
* `pgvector` is already implemented via `pgvector-elixir`.
* `PostGIS` is already implemented via `geo_postgis`.
* Creating the search indicies in migrations (leave it manual for now).

## Creating search indicies in migrations
For a `0.1` version I'd propose foregoing any migration DSL, and just using raw SQL. This could be supported later on should the package prove popular.

Here's an example migration. "calls" herein represent police scanner chatter:
```elixir
defmodule Scanner.Repo.Migrations.CreateCallsTableAndSearchIndex do
  use Ecto.Migration

  def change do
    # nothing parade-specific here, just a table.
    create table(:calls) do
      add :filename, :text, null: false
      add :system_id, references(:systems, type: :text), null: false
      add :call_length, :integer, null: false
      add :start_time, :naive_datetime, null: false
      add :stop_time, :naive_datetime, null: false
      add :talk_group_id, references(:talk_groups)
      add :transcript, :text
    end

    create index(:calls, :start_time)

    # create the index via plain SQL for v0.1
    execute("""
    CALL paradedb.create_bm25(
      index_name => 'calls_search_idx',
      table_name => 'calls',
      key_field => 'id',
      text_fields => paradedb.field(
        name => 'transcript',
        tokenizer => paradedb.tokenizer('stem', language => 'English')
      ) || paradedb.field(
        name => 'system_id',
        tokenizer => paradedb.tokenizer('raw')
      ),
      numeric_fields => paradedb.field('call_length') || paradedb.field('talk_group_id'),
      datetime_fields => paradedb.field('start_time') || paradedb.field('stop_time')
    )
    """)
  end
end
```

## The Schema
There's some important things to establish in the ecto schema itself:
- The schema has a search index that can be queried.
- Queries on that index return that schema (and thus can be mapped and preloaded accordingly).
- What fields are available in the search index.

```elixir
defmodule Scanner.Call do
  use Ecto.Schema
  use ParadeDB.Schema # new

  alias Scanner.System
  alias Scanner.TalkGroup

  schema "calls" do
    field :filename, :string
    field :call_length, :integer
    field :start_time, :naive_datetime
    field :stop_time, :naive_datetime
    field :transcript, :string

    belongs_to :system, System, type: :string
    belongs_to :talk_group, TalkGroup

    # new
    pdb_index "calls_search_idx", [
      :transcript,
      :system_id,
      :call_length,
      :talk_group_id,
      :start_time,
      :stop_time
    ]
  end
end
```

The index name could also be omitted, defaulting to `"#{schema_name}_search_idx"`.

## The FROM Expression
ParadeDB's query syntax essentially embeds (*perhaps hijacks*) in the FROM expression of a SQL query. Here's some not necessarily practical examples for sake of illustration:

```sql
SELECT * FROM calls WHERE transcript ilike '%walking%' AND call_length > 3;
```
A simple workalike to the above using ParadeQL would be:
```sql
SELECT * FROM calls_search_idx.search('transcript:walking') WHERE call_length > 3;

-- equivalent to:
SELECT * FROM calls_search_idx.search(query => paradedb.parse('transcript:walking')) WHERE call_length > 3;
```

pg_search's "Advanced" queries nest within:
```sql
SELECT * FROM calls_search_idx.search(query => paradedb.boolean(
  must => ARRAY[
    paradedb.parse('transcript:walking'),
    paradedb.range(field => 'call_length', range => '[3,)'::int4range)
  ]
)) AS i JOIN talk_groups AS ii ON ii.id = i.talk_group_id;
```

For the sake of it, a more maximal example:
```sql
SELECT * FROM calls_search_idx.search(query => paradedb.boolean(
  must => ARRAY[
    paradedb.parse('transcript:walking'),
    paradedb.phrase(field => 'transcript', phrases => ARRAY['on', 'sunshine'], slop => 1),
    paradedb.range(field => 'call_length', range => '[3,)'::int4range)
  ]
)) AS i
JOIN talk_groups_search_idx.search(query => paradedb.parse('alpha_tag:MCPD')) AS ii ON ii.id = i.talk_group_id
ORDER BY i.call_length DESC;
```
*Help wanted: make this query even more complex.*

### Expressing this in Ecto
First, the plain query:
```elixir
from(
  c in Calls,
  where: ilike(c.transcript, "%walking%"),
  where: c.call_length > 3,
)
```

The next one could be expressed as so, if we know we want a text search ahead of time:
```elixir
pdb_from(
  c in Calls,
  search: pdb_parse(c, "transcript:walking"),
  where: c.call_length > 3
)
```

However, that'd require the user to know they want to query the search index from the outset. A more composable example could be:
```elixir
query = from(c in Calls, where: c.call_length > 3)

query =
  if some_condition do
    search(query, [c], parse(c, "transcript:walking"))
  else
    query
  end
```

Finally, the maximal example:
```elixir
from(
  c in Call,
  join: tg in assoc(c, :talk_group),
  preload: [talk_group: tg],
  search: parse(c, "transcript:walking"),
  search: phrase(c.transcript, ["on", "sunshine"], slop: 1),
  search: c.call_length >= 3,
  search: parse(tg, "alpha_tag:MCPD"),
  order_by: {:desc, c.call_length}
)
```
The above assumes `search:` would work like Ecto's `where:` macro, defaulting to a logical "AND" for each expression. This would have to resolve to `paradedb.boolean` at the top level like so:

```sql
search_idx.search(query => paradedb.boolean(must => ARRAY[...]))
```

The maximal query also illustrates the need for search operators to have bindings, in order to disambiguate their targets.

## Top-level search options:
pg_search provides the following as options for the top-level search:
* `limit_rows` - The maximum number of rows to return.
* `offset_rows` - The number of rows to skip before starting to return rows.
* `stable_sort` - A boolean specifying whether ParadeDB should stabilize the order of equally-scored results, at the cost of performance.

```elixir
from(
  c in Call,
  join: tg in assoc(c, :talk_group),
  preload: [talk_group: tg],
  search: parse(c, "transcript:walking"),
  search: limit(c, 20),
  search: offset(c, 3),
  search: stable_sort(c, true)
  search: limit(tg, 50),
  order_by: {:desc, c.call_length}
)
```

## Misc. Notes
* ParadeQL strings are expressly **aren't** wrapped, leaving them as a sort of escape hatch. They would instead be passed as parameters.
* Parameterization appears like it'd be relatively straightforward.
* Optional parameters could be provided as a keyword list (maybe).
* It's somewhat verbose prepending everything with `search:`, but it helps draw a clean line between normal ecto expressions and ParadeDB specific ones.

## Open Questions
* Figure out what the boolean operators should look like.
* Figure out what facets should look like.
* Determine if a `search_fragment` macro in order?
* Figure out what where an ideal implementation would need to start.
