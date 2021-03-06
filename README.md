# How to transport database objects
- Conditions
    - There are two servers, one is the production server where PostgreSQL is already running, and the other is the temporary server.
    - The major version of PostgreSQL that you use must be larger than or equal to v12.
    - All the database objects to transport at the same time should be in the same tablespace. In other words,  if you want to transport the database objects in several tablespaces, you need to transport them separately for each tablespace.
1. Install PostgreSQL in the temporary server if not yet. Note that the following things must be the same between the production and temporary servers.
    - The major version of PostgreSQL (Also probably it's better to use the same minor version of PostgreSQL). For example, if PostgreSQL 12.2 is running in the production server, PostgreSQL 12.x should be installed in the temporary server.
    - The configure and compile options used when building PostgreSQL. Those options in the production server are viewable from the result of ```pg_config``` command.
1. Install pg_visibility contrib module in the temporary server if not yet.
1. Create the database cluster in the temporary server if not yet. Note that the settings (e.g., --encoding, --locale, --data-checksums, etc) specified when creating database cluster must be the same between the temporary and production servers.
1. Start PostgreSQL in the temporary server, with autovacuum disabled.
    - It's better to tune the configuration specially for high performance data bulkloading.
    - The parameter autovacuum must be set to false in the postgresql.conf.
1. Confirm that the results of the following SQL queries are the same between the production and temporary servers.
    ```
    SELECT
        pg_control_version,
        catalog_version_no
    FROM pg_control_system();

    SELECT
        max_data_alignment,
        database_block_size,
        blocks_per_segment,
        wal_block_size,
        bytes_per_wal_segment,
        max_identifier_length,
        max_index_columns,
        max_toast_chunk_size,
        large_object_chunk_size,
        float8_pass_by_value,
        data_page_checksum_version
    FROM pg_control_init();

    SELECT name, setting FROM pg_settings
    WHERE name IN ('block_size', 'data_checksums',
        'data_directory_mode', 'integer_datetimes',
        'max_function_args', 'max_identifier_length',
       'max_index_keys', 'segment_size', 'wal_block_size',
       'wal_segment_size')
    ORDER BY name;

    SELECT
        datname, pg_encoding_to_char(encoding),
        datcollate, datctype
    FROM pg_database
    WHERE datname = current_database();
    ```
    - If the results are not the same, you need to prepare the temporary server again so that they become the same.
1. Install the functions to use for transporting the tables, by executing pg_transport_table.sql, in both production and temporary servers if not installed yet. For example,
    ```
    [prod] $ psql -f pg_transport_table.sql
    [temp] $ psql -f pg_transport_table.sql
    ```
1. Make pg_visibility contrib module available in the temporary server if not yet, by executing ```CREATE EXTENSION```. For example,
    ```
    [temp] $ psql
    =# CREATE EXTENSION pg_visibility;
    ```
1. Confirm that the latest checkpoint redo location in the production server is larger than the current WAL write location in the temporary server. Save and mark the latest checkpoint redo location in the production server, as ***[1]***.
    - If the latest checkpoint redo location in the production server is less than or equal to the current WAL write location in the temporary server, you need to back to the step that creates the database cluster. Or you need to wait until many transactions happen in the production server and its latest checkpoint redo location becomes enough large.
    - The latest checkpoint redo location is viewable from the result of ```pg_controldata``` command or ```pg_control_checkpoint()``` function.
        ```
        [prod] $ pg_controldata $PGDATA | grep "REDO location"
        [prod] $ psql
        =# SELECT redo_lsn FROM pg_control_checkpoint();
        ```
    - The current WAL write location is viewable from the result of ```pg_current_wal_lsn()``` function.
        ```
        [temp] $ psql
        =# SELECT pg_current_wal_lsn();
        ```
1. Create the tablespace where the tables to transport will be located, in the temporary server. For example,
    ```
    [temp] mkdir /mnt/pgtblspc/test_tmp
    [temp] psql
    =# CREATE TABLESPACE test_tmp LOCATION '/mnt/pgtblspc/test_tmp';
    ```
1. Create the tables to transport in the temporary server. Also create other database objects like indexes, partitions, etc related to the tables in the temporary server. For example,
    ```
    [temp] $ psql
    =# CREATE TABLE example (key BIGINT PRIMARY KEY USING INDEX TABLESPACE test_tmp, val TEXT) TABLESPACE test_tmp;
    ```
    - All the database objects to transport must be located in the same tablespace.
1. Load data to the tables in the temporary server. For example,
    ```
    [temp] $ psql
    =# \copy example from /tmp/input_data.csv with csv
    ```
1. Create the database objects required for the tables that will be transported, in the temporary server, if necessary. For example, you might want to create indexes after data loading,
    ```
    [temp] $ psql
    =# CREATE INDEX example_val_idx ON example (val) TABLESPACE test_tmp;
    ```
1. Execute ```VACUUM FREEZE``` on the tables to transport, in the temporary server. It's better to execute ```VACUUM FREEZE``` at least twice just in the case. For example,
    ```
    [temp] $ psql
    =# VACUUM FREEZE example;
    =# VACUUM FREEZE example;
    ```
    - Note that you must confirm that there are no concurent transactions while executing ```VACUUM FREEZE```.
1. Confirm that all the pages in the tables to transport have already been marked as *frozen*. In other words, you need to confirm that the numbers of all-frozen pages and all the pages in the tables to transport are the same.
    - The number of all-frozen pages in the table can be calculated by ```pg_visibility_map_summary()``` function that pg_visibility contrib module provides. For example,
        ```
        [temp] $ psql
        =# SELECT all_frozen FROM pg_visibility_map_summary('example');
        ```
    - The number of all the pages in the table can be calculated by dividing the relation size by the block size. For example,
        ```
        [temp] $ psql
        =# SELECT pg_relation_size('example') / pg_size_bytes(current_setting('block_size'));
        ```
    - Note that you must not execute any transactions accessing the tables to transport, in the temporary server, since the beginning of this step and until the tranportation succeeds.
1. Execute ```CHECKPOINT``` in the temporary server. It's better to execute ```CHECKPOINT``` at least twice just in the case. For example,
    ```
    [temp] psql
    =# CHECKPOINT;
    =# CHECKPOINT;
    ```
1. Execute ```pgtp_create_manifest()``` function with each table to transport, in the temporary server. Write the output of the function into the file. For example,
    ```
    [temp] psql
    =# \t
    =# \o /tmp/transport_example_temp.sh
    =# SELECT pgtp_create_manifest('example');
    ```
    - Note that tuple-only mode should be enabled in psql when executing ```pgtp_create_manifest()``` so that only actual function result is output.
    - The name of table to transport needs to be specified in the argument of ```pgtp_create_manifest()```.
1. Shutdown PostgreSQL in the temporary server.
1. Move to the directory where the tables to transport are located, in the temporary server. Execute the file output by ```pgtp_create_manifest()``` as the shell script, there. For example,
    ```
    [temp] $ cd /mnt/pgtblspc/test_tmp/PG_13_202004241/12924
    [temp] $ chmod 744 /tmp/transport_example_temp.sh
    [temp] $ /tmp/transport_example_temp.sh
    ```
    - The directory where the tables to transport are located is the subdirectory with a path that depends on the PostgreSQL version and database OID, under the tablespace directory.
1. Execute ```umount``` to detach the file system where the tables to transport are located, in the temporary server. For example,
    ```
    [temp] $ sudo umount /dev/sdb1
    ```
1. Execute ```mount``` to attach the file system where the tables to transport are located, in the production server. For example,
    ```
    [prod] $ sudo mount /dev/sdb1 /mnt/pgtblspc
    ```
1. Create the tablespace where the tables to transport will be finally located, in the production server. For example,
    ```
    [prod] mkdir /mnt/pgtblspc/test
    [prod] psql
    =# CREATE TABLESPACE test LOCATION '/mnt/pgtblspc/test';
    ```
1. Create tables to transport in the production server. Also create other database objects like indexes, partitions, etc related to the tables in the production server. All the database objects related to the tables to transport must be created in the same way in both temporary and production servers.
    - All the database objects to transport must be located in the same tablespace.
1. Execute ```pgtp_apply_manifest()``` with each table to transport, in the production server. Write the output of the function into the file. For example,
    ```
    [prod] $ psql
    =# \t
    =# \o /tmp/transport_example_prod.sh
    =# SELECT pgtp_apply_manifest('example', '/mnt/pgtblspc/test_tmp/PG_13_202004241/12924/');
    ```
    - Note that tuple-only mode should be enabled in psql when executing ```pgtp_apply_manifest()``` so that only actual function result is output.
    - The name of table to transport needs to be specified in the first argument of ```pgtp_apply_manifest()```.
    - The path to the directory where the tables transported by the temporary server are located, should be specified in the second argument of ```pgtp_apply_manifest()```.
1. Move to the directory where the tables to transport are located, in the production server. Execute the file output by ```pgtp_apply_manifest()``` as the shell script, there. For example,
    ```
    [prod] $ cd /mnt/pgtblspc/test_tmp/PG_13_202004241/12924
    [prod] $ chmod 744 /tmp/transport_example_prod.sh
    [prod] $ /tmp/transport_example_prod.sh
    ```
1. Move all the files in the directory to the directory.
1. Confirm that ***[1]*** is larger than the current WAL write location (i.e., pg_current_wal_lsn()) in the temporary server.
    - If ***[1]*** is less than or equal to the current WAL write location in the temporary server, you need to back to the step that creates the database cluster.
1. Move under the database cluster directory in the temporary server, and execute the file output by the above step, as the shell script. This shell script renames the files of the database objects to transport, so that the production server can handle them. For example,
    ```
    [prod] $ cd $PGDATA
    [prod] $ chmod 744 /tmp/transport_example.sh
    [prod] $ /tmp/transport_example.sh
    ```
1. Copy all the renamed files from the temporary server to the production server.
