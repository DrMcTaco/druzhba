# Druzhba Documentation

## Getting Started

### Install Druzhba

Install locally in a Python3 `venv` or wherever you like:
```
python3 -m venv druzhba_test
source druzhba_test/bin/activate
pip install druzhba
#  or `pip install -e .`
```


### Configure your pipeline

Druzhba's behavior is defined by a directory of YAML configuration files.

As minimal example, create a directory `/pipeline`.

Create a file `pipeline/_pipeline.yaml`:

```
---
connection:
  host: ${REDSHIFT_HOST}
  port: 5439
  database: ${REDSHIFT_DATABASE}
  user: ${REDSHIFT_USER}
  password: ${REDSHIFT_PASSWORD}
index:
  schema: druzhba_raw
  table: pipeline_index
s3:
  bucket: ${S3_BUCKET}
  prefix: ${S3_PREFIX}
iam_copy_role: ${IAM_COPY_ROLE}
sources:
  - alias: pg
    type: postgres
```

Create a file `pipeline/pg.yaml`:

```
---
connection_string: postgresql://postgres:docker@localhost:54320/postgres
tables:
  - source_table_name: starter_table
    destination_table_name: starter_table
    destination_schema_name: druzhba_raw
    index_column: updated_at
    primary_key:
      - id
```

See [docs/configuration.md](docs/configuration.md) for more on the configuration files,
and [test/integration/config/](test/integration/config/) for more examples.

### Set up source and target databases

Run a local postgres instance:
```bash
# Run a local postgres instance in Docker on 54320
docker run -d --name pglocal -e POSTGRES_PASSWORD=docker -v my_dbdata:/var/lib/postgresql/data -p 54320:5432 postgres:11

# Run psql in the container
docker exec -it pglocal psql -Upostgres

# Inside PSQL:
CREATE TABLE starter_table (
    id SERIAL PRIMARY KEY,
    data VARCHAR(255),
    created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW()
);

INSERT INTO starter_table (data)
VALUES ('my first record'), ('my second record');

SELECT * FROM starter_table;
 id |       data       |        created_at         |        updated_at
----+------------------+---------------------------+---------------------------
  1 | my first record  | 2020-05-26 11:29:52.25809 | 2020-05-26 11:29:52.25809
  2 | my second record | 2020-05-26 11:29:52.25809 | 2020-05-26 11:29:52.25809
```

Connect to your Redshift instance somehow and:
```
CREATE USER druzhba_test PASSWORD 'Druzhba123';
CREATE SCHEMA druzhba_raw;
GRANT ALL ON SCHEMA druzhba_raw TO druzhba_test;
```


Set up your environment:
```
export DRUZHBA_CONFIG_DIR=pipeline
export REDSHIFT_USER=druzhba_test
export REDSHIFT_PASSWORD=Druzhba123
# ... set all the other envars from .env.test.sample for Redshift, AWS, S3...
```


### Invoke Druzhba

Once configuration is set up for your database, run Druzhba with:
```
druzhba -d pg -t starter_table
```

... your data is now in Redshift! Subsequent invocations will incrementally pull updated rows
from the source table. Of course, this is just the beginning of your pipeline.
See [docs/cli.md](docs/cli.md) for more on the command line interface.


## Usage Considerations

#### Index_column filters should be fast
Druzhba pulls incrementally according to the value of the `index_column` given in a table's configuration, and then
inserts-or-replaces new or updated rows according to an optional `primary_key`. On the first run (or if `--rebuild` is
given) Druzhba will create the target table. After that, it will use a SQL filter on `index_column` to only pull newly
updated rows.

Consequently, queries against `index_column` need to be fast! Usually, unless a table is `append_only`, an `updated_at` timestamp column
is used to for `index_column` - it is usually necessary to create a _database index_  (unfortunate name collision!) on this column to make
these pulls faster, which will slow down writes a little bit.


#### State
Druzhba currently tracks pipeline state by the _source_ database, database_alias, and table. Consequently, it supports
many-to-one pipelines from e.g. multiple copies of the same source database to a single shared target table.
But it does not support one-to-many pipelines, because it could not distinguish the state of the different pipelines.
SQL-based pipelines currently need to define a `source_table_name` which is used to track their state.


#### Manual vs Managed
A specific target table may be:
- "managed", meaning Druzhba handles the creation of the target table
  (inferred from datatypes on the source table) and the generation of
  the source-side query.
- "manual" - SQL queries are provided to read from the source (not
  necessarily from one table) and to create the target table (rather
  than inferring its schema from the source table).

Manual table creation is not supported for SQL Server.

## Testing

#### Unit Tests

To run unit tests locally:

```
python setup.py test
```

To use Docker for testing, create a `.env.test` file based on `env.test.sample` with
any environment variable overrides you wish to assign (can be empty).

To run unit tests in Docker:
```
docker-compose run test
```

#### Integration Tests

To run integration tests of full pipelines in Docker, you'll need
to add Redshift credentials to your environment or your `.env.test` file. This makes use
of a test schema in an existing Redshift database, and for safety will
fail if the schema name already exists.

Then,run:
```
source .env.test.sample
source .env.test  # For whatever overrides you need

docker-compose up -d postgres mysql
docker-compose build test

docker-compose run test bash test/integration/run_test.sh mysql postgres
```


### Releasing

See [docs/release.md](docs/release.md)