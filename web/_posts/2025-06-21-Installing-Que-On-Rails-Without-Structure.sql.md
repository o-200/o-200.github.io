---
layout: post
author: Alex Abramov
title: Que On Rails Without structure.sql
---

# Resume

`Que` (\*not confuse with `solid-queue`) have some specific way to implement it to rails due of `PostgreSQL` specific structures (especially functions). Que forces to switch `schema.rb` to `structure.sql` . At this article i'll describe how to implement Que and stay with `schema.rb` . Also, at this article i will describe my pain and save your time to solve this interesting task.

## Problem?

Que has only condition supports. It forces to switch `schema.rb` to `structure.sql`

```ruby
config.active_record.schema_format :sql
```

Why is `schema.rb` isn't compatible with `Que`???

`Rails::Migration` and especially `schema.rb` doesn't supports database specific structures, like views, triggers and functions. `Que` using functions and use it at table constraints:

```ruby
create_table "que_jobs", comment: "7", force: :cascade do |t|
  t.integer "priority", limit: 2, default: 100, null: false
  ...
  t.check_constraint "jsonb_typeof(data) = 'object'::text AND (NOT data ? 'tags'::text OR jsonb_typeof(data -> 'tags'::text) = 'array'::text AND jsonb_array_length(data -> 'tags'::text) <= 5 AND que_validate_tags(data -> 'tags'::text))", name: "valid_data"
end
```

`que_validate_tags` is a function which defines at one of `Que's` migrations. They are places at [4th que migration](https://github.com/que-rb/que/blob/master/lib/que/migrations/4/up.sql#L34):

but why we just cant create this function inside on migration?

```ruby
class CreateQueSchemaV4 < ActiveRecord::Migration[7.1]
  def up
    execute <<~SQL
      CREATE FUNCTION que_validate_tags(tags_array jsonb) RETURNS boolean AS $$
        SELECT bool_and(
          jsonb_typeof(value) = 'string'
          AND
          char_length(value::text) <= 100
        )
        FROM jsonb_array_elements(tags_array)
      $$
      LANGUAGE SQL;
    SQL
  end

  def down
    execute "DROP FUNCTION que_validate_tags(jsonb);"
  end
end
```

## Why its doesnt solved my problem?

When Rails checks the schema (db:schema:dump), it does so in a new transaction. Functions are missing and we [got that problem](https://github.com/que-rb/que/issues/397).

`schema.rb` is a structure which created only for Rails DSL. Rails migrations have a lot of transactions/processes which can forgot your function from database at any process.

So, for that creators of `Que` asking for switching `schema.rb -> structure.sql`. It's using `pg_dump` instead of `rails db:schema:dump` and includes all objects.

## structure.sql is a endless headache.

While `structure.sql` works, it comes with drawbacks:

- Harder to review in version control
- More prone to merge conflicts
- Loses Rails' abstraction over database specifics

## arfi

I found a solution - [arfi](https://github.com/unurgunite/arfi)

```
The ARFI gem provides the ability to create and maintain custom
SQL functions for ActiveRecord models without switching to structure.sql (an SQL-based schema).
You can use your own SQL functions in any part of the project,
from migrations and models to everything else.
```

This is what we need! But how it works?
That gem loads functions therefore and guarantee will be before of any migrations and schema validations.

## Installing Que without switching to structure.sql

1. Add 'que' gem

```
bundle add que
```

2. Add 'arfi' gem

```
bundle add arfi
```

3. [Configure arfi](https://github.com/unurgunite/arfi?tab=readme-ov-file#usage) and create functions from [que's migrations](https://github.com/que-rb/que/tree/master/lib/que/migrations). Instead of `CREATE FUNCTION` better use `CREATE OR REPLACE FUNCTION`

4. Create list of que's migrations [que's migrations](https://github.com/que-rb/que/tree/master/lib/que/migrations)

```
db/migrate/
  20250605112406_create_que_schema_v1.rb
  20250605112407_create_que_schema_v2.rb
  20250605112408_create_que_schema_v3.rb
  20250605112409_create_que_schema_v4.rb
  20250605112410_create_que_schema_v5.rb
  20250605112411_create_que_schema_v6.rb
  20250605112412_create_que_schema_v7.rb
```

Copy sql code and put it similar to:

```
class CreateQueSchemaV1 < ActiveRecord::Migration[7.1]
  def up
    execute <<-SQL
      CREATE TABLE que_jobs
      (
        priority    integer     NOT NULL DEFAULT 1,
        run_at      timestamptz NOT NULL DEFAULT now(),
        job_id      bigserial   NOT NULL,
        job_class   text        NOT NULL,
        args        json        NOT NULL DEFAULT '[]'::json,
        error_count integer     NOT NULL DEFAULT 0,
        last_error  text,

        CONSTRAINT que_jobs_pkey PRIMARY KEY (priority, run_at, job_id)
      );
    SQL
  end

  def down
    execute "DROP TABLE que_jobs;"
  end
end
```

p.s.: don't forget to remove functions

Execute migrations. Voila! Que is works! And you avoid switching to bulky `structure.sql`
