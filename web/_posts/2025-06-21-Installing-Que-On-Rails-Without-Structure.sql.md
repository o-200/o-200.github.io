---
layout: post
author: Alex Abramov
title: Que On Rails Without structure.sql
---

# Предупреждение (!)

Если есть возможность - устанавливайся Que по инструкции. Это проверенный путь, который в будущем обеспечит надёжность. Установка не по инструкции приводит к нестабильной работе с миграциями/схемой в базе данных (конкретно речь про миграции и таблицы в бд от que и que-scheduler).

Чтобы это всё работало стабильно необходимо полностью переписывать миграции от que на rails migration dsl. Из-за того, что Que постоянно использует триггеры, необычные для rails типы данных - это практически невозможно.

# Resume

`Que` (не путать с `solid-queue`) имеет ограниченную поддержу rails. Для установки Que на Rails c ActiveRecord необходимо сменить `schema.rb` на `structure.sql`. В этом посте я расскажу как добавить Que и остаться со `schema.rb`. Я сохраню ваше время, чтобы вы не пытались различными костылями заставить Que работать и приведу различные решения этой задачи.

## Почему Que не работает?

Que имеет частичную поддержку, в README.md прямо говорится что

```
If you're using ActiveRecord to dump your database's schema, please set your schema_format to :sql so that Que's table structure is managed correctly.
```

То есть от нас требуют чтобы мы поменяли формат схемы и полностью всё отформатировали в `structure.sql`:

```ruby
config.active_record.schema_format :sql
```

## Почему ActiveRecord и `schema.rb` не поддерживаемы с `Que`

Модуль `Rails::Migration` и `schema.rb` поддерживают базовые и часто используемые структуры. Этим самым у нас обделяются такие структуры как view, trigger и functions. Que активно использует эти структуры у себя и по этому у нас появился один конфликт, который находится в схеме в таблице `que_jobs` используется constraint который обращается к функцие:

```ruby
create_table "que_jobs", comment: "7", force: :cascade do |t|
  t.integer "priority", limit: 2, default: 100, null: false
  ...
  t.check_constraint "jsonb_typeof(data) = 'object'::text AND (NOT data ? 'tags'::text OR jsonb_typeof(data -> 'tags'::text) = 'array'::text AND jsonb_array_length(data -> 'tags'::text) <= 5 AND que_validate_tags(data -> 'tags'::text))", name: "valid_data"
end
```

`que_validate_tags` является функцией которая создается в одной из миграций в que. Нужная "проблемная" миграция расположена на [четвёртой que миграции](https://github.com/que-rb/que/blob/master/lib/que/migrations/4/up.sql#L34):

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

## Но почему у нас появляется проблема когда мы производим все миграции от que?

Rails постоянно проверяет схема на валидность (c помощью rails db:schema:dump), во время этой проверки создаётся транзакция в которой теряются все непонятные для фреймворка структуры. В данном случае теряется та самая проблемная функция и мы [получаем невозможность полноценно работать с приложением](https://github.com/que-rb/que/issues/397).

`schema.rb` это структура которая исключительно работает с DSL от Rails. Миграция как процесс не такой простой как кажется на первый взгляд. Идёт речь про кучу процессов и транзакций в которых rails игнорирует неизвестные ему структуры.

Именно по этому создатели `Que` просят делать перенос `schema.rb -> structure.sql`. Суть в том, что в structure.sql загружаются все структуры, используя внутренние инструменты СУБД (в нашем случае это `pg_dump`) заместо db:schema:dump.

## Почему стоит избегать structure.sql

Работающий `structure.sql` имеет свои недостатки:

- Тяжело проверять целостность
- Тяжело решать конфликты
- Мы всё дальше отходим от экосистемы rails

## arfi

Но всё же есть возможность сохранить schema.rb и заставить Que работать.

[arfi](https://github.com/unurgunite/arfi)

```
ARFI gem предоставляет возможность создавать и поддерживать пользовательские
функции SQL для моделей ActiveRecord без переключения на structure.sql (схему на основе SQL).
Вы можете использовать собственные функции SQL в любой части проекта,
от миграций и моделей до всего остального.
```

Это ведь то что нам нужно! Как же оно работает?
Гем предзагружает функции перед каждым процессом, связанный с миграциями и делает так, чтобы rails видел и понимал что такое функция.

## Устанавливаем Que без переноса на structure.sql с использованием arfi

1. Добавляем 'que' гем

```
bundle add que
```

2. Добавляем 'arfi' гем

```
bundle add arfi
```

3. [Настраиваем arfi](https://github.com/unurgunite/arfi?tab=readme-ov-file#usage) и создаем проблемную функцию из [que's миграции](https://github.com/que-rb/que/tree/master/lib/que/migrations). В функциях заместо `CREATE FUNCTION` используется `CREATE OR REPLACE FUNCTION`

4. Создаем все миграции от que без упоминания функций [que's миграции](https://github.com/que-rb/que/tree/master/lib/que/migrations)

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

Ваши миграции будут похожи на:

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

5 Производим `rails db:migrate`

Вуаля! Que работает и вы увернулись от перехода на структурую
