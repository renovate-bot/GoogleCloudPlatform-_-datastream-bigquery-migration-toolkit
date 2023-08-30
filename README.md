# Datastream BigQuery Migration Toolkit


Datastream BigQuery Migration Toolkit is an open-source software offered by Google Cloud which makes it easy for customers to migrate from Dataflow's [Datastream to BigQuery template](https://cloud.google.com/dataflow/docs/guides/templates/provided/datastream-to-bigquery) to [Datastream's native BigQuery replication solution](https://cloud.google.com/datastream-for-bigquery).  
The toolkit creates a Datastream-compatible BigQuery table in the user's Google Cloud project, and copies the content of a BigQuery table created by Dataflow to the newly created BigQuery table.

On a high level, the migration story consists of the following steps:
1. Create, start and pause a Datastream stream with a BigQuery destination.
2. Run the migration tool on each BigQuery table that needs to be migrated.
3. Resume the stream.

## Overview
The toolkit does the following:
1. Retrieves the source table schema using Datastream's [discover API](https://cloud.google.com/datastream/docs/using-datastream-apis#creating_and_managing_connection_profiles).
2. Creates a Datastream-compatible BigQuery table based on the retrieved schema.
3. Fetches the schema of the BigQuery table from which you're migrating to determine the necessary data type conversions.
4. Copies all existing rows from the original table to the new table, including appropriate column type [casts](https://cloud.google.com/bigquery/docs/reference/standard-sql/conversion_functions#cast).

This migration is designed for Datastream customers migrating off of Dataflow's [Datastream to BigQuery template](https://cloud.google.com/dataflow/docs/guides/templates/provided/datastream-to-bigquery), but it can also assist in migration from other pipelines, as explained below.

## Limitations
* The toolkit expects column names in the existing and new BigQuery tables to match (ignoring metadata columns). This should be the case if no Dataflow user-defined functions (UDFs) were applied on the existing table.
* Cross-region and cross-project migrations aren't supported.
* The migration works on a per-table basis.
* Supports only Oracle and MySQL sources.

## Code structure
The toolkit consists of 2 subpackages:
- *SQL generators*: Python modules which generate SQL statements and write them to local files.
- *Executors*: Python modules that read the files generated by the SQL generators and execute them using BigQuery and Datastream Python SDKs.

The toolkit is structured this way to allow maximal flexibility and visibility over the migration.  
The entrypoint for the migration is the `migration_toolkit/migrate_table.py` file.


## Arguments

```
usage: migrate_table.py [-h] [--force] [--verbose] --project-id PROJECT_ID --stream-id STREAM_ID --datastream-region DATASTREAM_REGION --source-schema-name SOURCE_SCHEMA_NAME --source-table-name SOURCE_TABLE_NAME --bigquery-source-dataset-name BIGQUERY_SOURCE_DATASET_NAME --bigquery-source-table-name BIGQUERY_SOURCE_TABLE_NAME
                        {dry_run,create_table,full}

Datastream BigQuery Migration Toolkit arguments

positional arguments:
  {dry_run,create_table,full}
                        Migration mode.
                        'dry_run': only generate the DDL for 'CREATE TABLE' and SQL for copying data, without executing.
                        'create_table': create a table in BigQuery, and only generate SQL for copying data without executing.
                        'full': create a table in BigQuery and copy all rows from existing BigQuery table.

optional arguments:
  -h, --help            show this help message and exit
  --force, -f           Don't wait for the user prompt.
  --verbose, -v         Verbose logging.

required arguments:
  --project-id PROJECT_ID
                        Google Cloud project ID/number (cross-project migration isn't supported), for example `datastream-proj` or `5131556981`.
  --stream-id STREAM_ID
                        Datastream stream ID, for example `mysql-to-bigquery`.
  --datastream-region DATASTREAM_REGION
                        Datastream stream location, for example `us-central1`.
  --source-schema-name SOURCE_SCHEMA_NAME
                        Source schema name, for example `my_db`.
  --source-table-name SOURCE_TABLE_NAME
                        Source table name, for example `my_table`.
  --bigquery-source-dataset-name BIGQUERY_SOURCE_DATASET_NAME
                        BigQuery dataset name of the existing BigQuery table, for example `dataflow_dataset`.
  --bigquery-source-table-name BIGQUERY_SOURCE_TABLE_NAME
                        The name of the existing BigQuery table, for example `dataflow_table`.
```
## Setup
The most convenient way to run the migration toolkit is by using `docker`:
1. Clone the repository and change directory into it:
   ```
   git clone https://github.com/GoogleCloudPlatform/datastream-bigquery-migration-toolkit &&
   cd datastream-bigquery-migration-toolkit
   ```
2. Build the image:
   ```
   docker build -t migration-service .
   ```
3. Authenticate with your gcloud credentials:
   ```
   docker run -ti --name gcloud-config migration-service gcloud auth application-default login
   ```
4. Set your Google Cloud project property:
   ```
   docker run -ti --volumes-from gcloud-config migration-service gcloud config set project <YOUR GOOGLE CLOUD PROJECT>
   ```

You're all set!

## Step-by-step guide for migration from Dataflow to Datastream's native solution

1. [Create a Datastream stream with a BigQuery destination](https://cloud.google.com/datastream/docs/quickstart-replication-to-bigquery)
    > When configuring the source, make sure to choose manual backfill under `Choose backfill mode for historical data -> Manual` ([learn more](https://cloud.google.com/datastream/docs/create-a-stream#configuresourcedb)).
2. Start the stream, and immediately pause it. This allows Datastream to capture CDC events before the migration starts.
    >   NOTE: It is possible that between starting and pausing, Datastream processes some events, thus creating a BigQuery table. In such a case, you must manually delete the table before proceeding with the migration, otherwise the migration will fail.

3. Drain the Datastream and Dataflow pipeline:
   1. Pause the existing Cloud Storage destination stream.
   2. Check the total latency metric for the stream and wait at least as long as the current latency to ensure that any in-flight events are written to the destination.
   3. [Drain the Dataflow job](https://cloud.google.com/dataflow/docs/guides/stopping-a-pipeline#drain).

4. Execute the migration:
   1. Run the migration in `dry_run` mode:
      ```
      docker run -v output:/output -ti --volumes-from gcloud-config migration-service python3 ./migration/migrate_table.py dry_run \
      --project-id <GOOGLE_CLOUD_PROJECT_ID> \
      --stream-id <STREAM_NAME> \
      --datastream-region <STREAM_REGION> \
      --source-schema-name <SOURCE_SCHEMA_NAME> \
      --source-table-name <SOURCE_TABLE_NAME> \
      --bigquery-source-dataset-name <BIGQUERY_SOURCE_DATASET_NAME> \
      --bigquery-source-table-name <BIGQUERY_SOURCE_TABLE_NAME> 
      ```
   2. Inspect the `.sql` files under `output/create_target_table` and `output/copy_rows`. These are the SQL commands that will be executed on your Google Cloud project:
      ```
      docker run -v output:/output -ti migration-service find output/create_target_table -type f -print -exec cat {} \;
      ```
      ```
      docker run -v output:/output -ti migration-service find output/copy_rows -type f -print -exec cat {} \;
      ```
   3. To execute the SQL commands, run the migration in `full` mode:
      ```
      docker run -v output:/output -ti --volumes-from gcloud-config migration-service python3 ./migration/migrate_table.py full \
      --project-id <GOOGLE_CLOUD_PROJECT_ID> \
      --stream-id <STREAM_NAME> \
      --datastream-region <STREAM_REGION> \
      --source-schema-name <SOURCE_SCHEMA_NAME> \
      --source-table-name <SOURCE_TABLE_NAME> \
      --bigquery-source-dataset-name <BIGQUERY_SOURCE_DATASET_NAME> \
      --bigquery-source-table-name <BIGQUERY_SOURCE_TABLE_NAME>
      ```

5. Resume the stream paused in step 2.
6. Open Google Cloud's Logs Explorer, and look for Datastream logs with the following query:
   ```
   resource.type="datastream.googleapis.com/Stream"
   resource.labels.stream_id=<STREAM_NAME>
   ```
   Look for the following log (where `%d` is a number):
   ```
   Completed writing %d records into..
   ```
   This log signals that the new stream successfully loaded data to BigQuery.  
   Note: this log will appear only if there is CDC data to ingest.


## Migrating from other pipelines
The toolkit enables you to onboard from other pipelines to Datastream's native BigQuery solution.  
The toolkit can generate `CREATE TABLE` DDLs for Datastream-compatible BigQuery tables, based on source database schema, by using `dry_run`:
```
docker run -v output:/output -ti --volumes-from gcloud-config migration-service python3 ./migration/migrate_table.py dry_run \
--project-id <GOOGLE_CLOUD_PROJECT_ID> \
--stream-id <STREAM_NAME> \
--datastream-region <STREAM_REGION> \
--source-schema-name <SOURCE_SCHEMA_NAME> \
--source-table-name <SOURCE_TABLE_NAME> \
--bigquery-source-dataset-name <BIGQUERY_SOURCE_DATASET_NAME> \
--bigquery-source-table-name <BIGQUERY_SOURCE_TABLE_NAME>
```
Since the BigQuery table schemas may vary, we can't provide a universal SQL statement for copying rows.  
You can use the schemas at `output/create_target_table`, `output/source_table_ddl` to compose a SQL statement, with the appropriate [casts](https://cloud.google.com/bigquery/docs/reference/standard-sql/conversion_functions) on the `{source_columns}`.  
The following is an example SQL statement format that you can use:
```
INSERT INTO {destination_table} ({destination_columns}) SELECT {source_columns} FROM {source_table};
```