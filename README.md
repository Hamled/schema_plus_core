[![Gem Version](https://badge.fury.io/rb/schema_plus_core.svg)](http://badge.fury.io/rb/schema_plus_core)
[![Build Status](https://secure.travis-ci.org/SchemaPlus/schema_plus_core.svg)](http://travis-ci.org/SchemaPlus/schema_plus_core)
[![Coverage Status](https://img.shields.io/coveralls/SchemaPlus/schema_plus_core.svg)](https://coveralls.io/r/SchemaPlus/schema_plus_core)
[![Dependency Status](https://gemnasium.com/lomba/schema_plus_core.svg)](https://gemnasium.com/SchemaPlus/schema_plus_core)

# SchemaPlus::Core

SchemaPlus::Core creates an internal extension API to ActiveRecord.  The idea is that:

* ShemaPlus::Core does the monkey-patching so clients don't have to know too much about the internal of ActiveRecord.

* SchemaPlus::Core's extension API is consistent across the various connection adapters, so clients don't have to figure out how to extend each connection adapter independently.

* SchemPlus::Core's extension API intends to remain reasonably stable even as ActiveRecord changes.

By itself, SchemaPlus::Core does not change any behavior or add any external features to ActiveRecord.  It just makes the API available to clients.

SchemaPlus::Core is a client of [schema_monkey](https://github.com/SchemaPlus/schema_monkey), using [modware](https://github.com/ronen/modware) to define middleware callback stacks.


## Compatibility

SchemaPlus::Core is tested on:

<!-- SCHEMA_DEV: MATRIX - begin -->
<!-- These lines are auto-generated by schema_dev based on schema_dev.yml -->
* ruby **2.1.5** with activerecord **4.2**, using **mysql2**, **sqlite3** or **postgresql**

<!-- SCHEMA_DEV: MATRIX - end -->


## Installation

<!-- SCHEMA_DEV: TEMPLATE INSTALLATION - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
As usual:

```ruby
gem "schema_plus_core"                # in a Gemfile
gem.add_dependency "schema_plus_core" # in a .gemspec
```

To use with a rails app, also include

```ruby
gem "schema_monkey_rails"
```

which creates a Railtie to that will insert SchemaPlus::Core appropriately into the rails stack. To use with Padrino, see [schema_monkey_padrino](https://github.com/SchemaPlus/schema_monkey_padrino).

<!-- SCHEMA_DEV: TEMPLATE INSTALLATION - end -->


## Usage

The API is in the form of a collection of [modware](https://github.com/ronen/modware) middleware callback stacks.  A client of the API uses [schema_monkey](https://github.com/SchemaPlus/schema_monkey) to insert middleware modules into the stacks.  As per [schema_monkey](https://github.com/SchemaPlus/schema_monkey), the typical module structure looks like:

```ruby
require "schema_plus/core"

module MyClient
  module Middleware
    #
    # Middleware modules to insert in SchemaPlus::Core API stacks
    #
  end
  module ActiveRecord
    #
    # direct ActiveRecord enhancements, should your client need any.
    #
  end
end

SchemaMonkey.register MyClient
```

For example, a client could use the `Migration::Index` stack to automatically make an index unique if any column starts with 'u':

```ruby
require "schema_plus/core"

module AutoUniquify
  module Middleware

    module Migration
      module Index
        def before(env)
          env.options[:unique] = true if env.column_names.grep(/^u/).any?
        end
      end
    end

  end
end

SchemaMonkey.register AutoUniquify
```

Ideally most clients will not need to define direct ActiveRecord enhancements, other than perhaps to create new methods on public classes.  If you have a client that needs more complex monkey-patching, that could be a sign that SchemaPlus::Core's API is missing some useful functionality -- consider submitting a PR to SchemaPlus::Core add it!

## API details

For organizational clarity, the SchemaPlus::Core stacks are grouped into modules based on broad categories.  In the Env field tables below, Initialized

### Schema

Stacks for general operations queries pertaining to the entire database schema:

* `Schema::Define`

  Wrapper around the `ActiveRecord::Schema.define` method loads a dumped schema file (`schema.rb`).
  
    Env Field    | Description | Initial value
    --- | --- | ---
    `:info` | Schema information hash | *args*
    `:block` | The proc containing the schema definition statements | *args*

  The base implementation calls the block to define the schema.

* `Schema::Indexes`

  Wrapper around the `connection.indexes(table_name)` method.  Env contains:

    Env Field    | Description | Initial value
    --- | --- | ---
    `:index_definitions` | The result of the lookup | `[]`
    `:connection` | The current ActiveRecord connection | *context*
    `:table_name` | The name of the table to query | *arg*
    `:query_name` | Label sometimes used by ActiveRecord logging | *arg*

  The base implementation appends its results to `env.index_definitions`

* `Schema::Tables`

  Wrapper around the `connection.tables()` method.  Env contains:

    Env Field    | Description | Initialized
    --- | --- | ---
    `:tables`     | The result of the lookup | `[]`
    `:connection` | The current ActiveRecord connection | *context*
    `:table_name` | (SQlite3 only) | *arg*
    `:database`   | (Mysql only) | *arg*
    `:like`       | (Mysql only) | *arg*
    `:query_name` | Label sometimes used by ActiveRecord logging | *arg*

  The base implementation appends its results to `env.tables`

### Model

Stacks for class methods on ActiveRecord models.

* `Model::Columns`

  Wrapper around the `Model.columns` query

      Env Field    | Description | Initialized
    --- | --- | ---
    `:columns`     | The resulting Column objects | `[]`
    `:model` | The model Class being queried | *context*

  The base implementation appends its results to `env.columns`

* `Model::ResetColumnInformation`

    Wrapper around the `Model.reset_column_information` method

        Env Field    | Description | Initialized
        --- | --- | ---
        `:model` | The model Class being reset | *context*

    The base implementation performs the reset.

### Migration

Stacks for operations that change the schema.  In some cases the operation immediately modifies the database schema, in others the operation defines ActiveRecord objects (e.g., column definitions in a create_table definition) and the actual modification of the database schema will happen some time later.

* `Migration::Column`

  Callback stack for various ways to define or modify a column.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:caller`     | The ActiveRecord instance responsible for performing the action | *context*
    `:operation`  | One of `:add`, `:change`, `:define`, `:record` | *context*
    `:table_name` | The name of the table | *arg*
    `:column_name` | The name of the column | *arg*
    `:type`       | The ActiveRecord column type (`:integer`, `:datetime`, etc.) | *arg*
    `:options`    | The column options | *arg*, default `{}`

  The base implementation performs the column operation.  No value is returned.

  Notes:

  1. The `:operation` field has the following meanings:

      * `:add` - The column will be added immediately (`Migration#add_column`)
      * `:change` - The column will be changed immediately (`Migration#change_column`)
      * `:define` - The column will be added to table definition, which will be emitted later
      * `:record` - The column info will be added to a migration command recorder, for later playback in reverse by `Migration#down`

  2. In the case of a table definition using `t.references` or `t.belongs_to`, the `:type` field will be set to `:reference` and the `:column_name` will include the `"_id"` suffix

  3. ActiveRecord's base implementation may make nested calls to column creation.  For example: References result in a nested call to create an integer column; Polymorphic references nest calls to create two columns; Sqlite3 implements `:change` by a nested call to a new table definition.  SchemaPlus::Core doesn't attempt to normalize or suppress these; each such nested call will result in its own `Migration::Column` stack execution.

* `Migration::DropTable`

  Drops a table from the database

    Env Field    | Description | Initialized
    --- | --- | ---
    `:connection`     | The current ActiveRecord connection | *context*
    `:table_name` | The name of the table | *arg*
    `:options`    | Drop table options | *arg*

   The base implementation drops the table.  No value is returned.

* `Migration::Index`

  Callback stack for various ways to define an index.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:caller`     | The ActiveRecord instance responsible for performing the action | *context*
    `:operation`  | `:add` or `:define` | *context*
    `:table_name` | The name of the table | *arg*
    `:column_names` | The names of the columns | *arg*
    `:options`    | The index options | *arg*, default `{}`

  The base implementation performs the index creation operation.  No value is returned.

  Notes:

  1. The `:operation` field has the following meanings:

      * `:add` - The index will be added immediately (`Migration#add_index`)
      * `:define` - The index will be added to a table definition, which will be emitted later.

### Sql

Stacks for internal operations that generate SQL.

* `Sql::ColumnOptions`

  Callback stack around generation of the SQL options for a column definition.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:sql` | The resulting SQL | `""`
    `:caller`     | The ActiveRecord::SchemaCreation instance | *context*
    `:connection` | The current ActiveRecord connection | *context* |
    `:column`     | The column definition object | *context* |
    `:options`    | The column definition options | *context* |

  The base implementation appends the options SQL to `env.sql`

* `Sql::IndexComponents`

  Callback stack around generation of the SQL for an index definition.

    Env Field    | Description | Initialized
    --- | --- | ---
    `:sql` | The resulting SQL components, in a struct with fields `:name`, `:type`, `:columns`, `:options`, `:algorithm`, `:using` | *empty struct*
    `:connection` | The current ActiveRecord connection | *context*
    `:table_name` | The name of the table | *context*
    `:column_names` | The names of the columns | *context*
    `:options`    | The index options | *context*

  The base implementation *overwrites* the contents of `env.sql`

  Notes:

  1. SQLite3 ignores the `:type`, `:algoritm`, and `:using` fields of `env.sql`

* `Sql::Table`

  Callback stack around generation of the SQL for a table

    Env Field    | Description | Initialized
    --- | --- | ---
    `:sql` | The resulting SQL components in a struct with fields `:command`, `:name`, `:body`, `:options`, `:quotechar` | *empty struct*
    `:caller`     | The ActiveRecord::SchemaCreation instance | *context*
    `:connection` | The current ActiveRecord connection | *context*
    `:table_definition` | The TableDefinition object for the table | *context*

  The base implementation *overwrites* the contents of `env.sql`

  Notes:

  1. `env.sql.command` contains the index creation command such as `CREATE TABLE` or `CREATE TEMPORARY TABLE`
  2. `env.sql.quotechar` contains the quote character ', ", or \` to wrap `env.sql.name` in.


### Query

Stacks around low-level query execution

* `Query::Exec`

  Callback stack wraps the emission of sql to the underlying dbms gem.

      Env Field    | Description | Initialized
    --- | --- | ---
    `:result` | The result of the database query | *unset*
    `:caller`     | The ActiveRecord::SchemaCreation instance | *context*
    `:sql`        | The SQL string | *context*
    `:binds`      | Values to substitute into the SQL string
    `:query_name` | Label sometimes used by ActiveRecord logging | *arg*


### Dumper

SchemaPlus::Core provides a state object and of callbacks to various phases of the schema dumping process.  The dumping process fleshes out the state object-- nothing is actually written to the dump file until after the state is fleshed out.

#### Schema Dump state

* `Class SchemaPlus::Core::SchemaDump`

  An instance of `SchemaPlus::Core::SchemaDump` gets passed to each of the callback stacks; the dump gets built up by fleshing out its contents.  `SchemaDump` has the following fields and methods:

  * `dump.initial = []` - an array of strings containing statements to start the schema with, such as `enable_extension 'hstore'`
  * `dump.data = OpenStruct.new` - a place for clients to store arbitrary data between phases
  * `dump.tables = {}` - a hash mapping table names to SchemaDump::Table objects
  * `dump.final = []` - an array of strings containing statements to end the schema with.
  * `dump.depends(table_name, [prerequisite_table_names])` - call this method to ensure that the definition of `table_name` won't be output before its prerequisites.

* `Class SchemaPlus::Core::SchemaDump::Table`

  Each table in the dump has its contents in a SchemaDump::Table object, with these fields:

  * `table.name` - the table name
  * `table.pname` - table as actually used in SQL (without any prefixes or suffixes)
  * `table.options` - a string containing the options to `Migration.create_table`
  * `table.columns = []` - an array of SchemaDump::Table::Column objects
  * `table.indexes = []` - an array of SchemaDump::Table::Index objects
  * `table.statements` - a collection of statements to include in the table definition; each is a string that should start with `"t."`
  * `table.trailer` - a collection of migration statements to include immediately outside the table definition.  Each is a string

* `Class SchemaPlus::Core::SchemaDump::Table::Column`

  Each column in a table has its contents in a SchemaDump::Table::Column object, with these fields and methods:

  * `column.name` - the column name
  * `column.type` - the column type (i.e., what comes after `"t."`)
  * `column.options` - an optional string containing the options for the column
  * `column.comments` - an optional string containing a comment to put in the schema dump
  * `column.add_option(option)` - adds an option to the current string, separating with a "," if the current set isn't blank
  * `column.add_comment(comment)` - adds an option to the current string, separating with a ";" if the current string isn't blank

* `Class SchemaPlus::Core::SchemaDump::Table::Index`

  Each index in a table has its contents in a SchemaDump::Table::Index object, with these fields and methods:

  * `index.name` - the index name
  * `index.columns` - the columns that are in the index
  * `index.options` - an optional string containing the options for the index
  * `index.add_option(option)` - adds an option to the current string, separating with a "," if the current set isn't blank

#### Schema Dump Middleware stacks

* `Dumper::Initial`

  Callback stack wraps the creation of initial statements for the dump.

      Env Field    | Description | Initialized
    --- | --- | ---
    `:initial` | The initial statements | []
    `:dump`    | The SchemaDump object | *context*
    `:dumper`  | The current ActiveRecord::SchemaDumper instance| *context*
    `:connection`      | The current ActiveRecord connection | *context*

  The base method appends initial statements to `env.initial`.

* `Dumper::Tables`

  Callback stack wraps the dumping of all tables.

      Env Field    | Description | Initialized
    --- | --- | ---
    `:dump`    | The SchemaDump object | *context*
    `:dumper`  | The current ActiveRecord::SchemaDumper instance| *context*
    `:connection`      | The current ActiveRecord connection | *context*

  The base method iterates through all tables, dumping each.


* `Dumper::Table`

  Callback stack wraps the dumping of each table

      Env Field    | Description | Initialized
    --- | --- | ---
    `:table` | A SchemaDump::Table object| `table.name` *only*
    `:dump`    | The SchemaDump object | *context*
    `:dumper`  | The current ActiveRecord::SchemaDumper instance| *context*
    `:connection`      | The current ActiveRecord connection | *context*

  The base method iterates through all columns and indexes of the table, and *overwrites* the contents of `table`,

  Notes:

  1. When the stack is called, `env.dump.tables[env.table.name]` contains the `env.table` object.

* `Dumper::Indexes`

  Callback stack wraps the dumping of the indexes of a table

      Env Field    | Description | Initialized
    --- | --- | ---
    `:table` | A SchemaDump::Table object| *context*
    `:dump`    | The SchemaDump object | *context*
    `:dumper`  | The current ActiveRecord::SchemaDumper instance| *context*
    `:connection`      | The current ActiveRecord connection | *context*

  The base method appends the collection of SchemaDump::Table::Index objects to `env.table.indexes`


## History

* 0.1.0 Initial release

## Development & Testing

Are you interested in contributing to SchemaPlus::Core?  Thanks!  Please follow the standard protocol: fork, feature branch, develop, push, and issue pull request.

Some things to know about to help you develop and test:

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_DEV - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_dev**:  SchemaPlus::Core uses [schema_dev](https://github.com/SchemaPlus/schema_dev) to
  facilitate running rspec tests on the matrix of ruby, activerecord, and database
  versions that the gem supports, both locally and on
  [travis-ci](http://travis-ci.org/SchemaPlus/schema_plus_core)

  To to run rspec locally on the full matrix, do:

        $ schema_dev bundle install
        $ schema_dev rspec

  You can also run on just one configuration at a time;  For info, see `schema_dev --help` or the [schema_dev](https://github.com/SchemaPlus/schema_dev) README.

  The matrix of configurations is specified in `schema_dev.yml` in
  the project root.


<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_DEV - end -->

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_MONKEY - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_monkey**: SchemaPlus::Core is implemented as a
  [schema_monkey](https://github.com/SchemaPlus/schema_monkey) client,
  using [schema_monkey](https://github.com/SchemaPlus/schema_monkey)'s
  convention-based protocols for extending ActiveRecord and using middleware stacks.
  For more information see [schema_monkey](https://github.com/SchemaPlus/schema_monkey)'s README.

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_MONKEY - end -->
