### Snapshot Creation and Time Travel in Apache Iceberg

Apache Iceberg has revolutionized data management in data lake architectures by offering robust metadata management and data snapshotting capabilities. This post dives deep into how snapshot creation and time travel work in Iceberg, exploring the metadata file structure, snapshot log, and the atomic swap mechanism that ensures consistency. Using PyIceberg examples and Polaris for demonstrations, we’ll illustrate how Iceberg allows consistent data state retrieval and time-travel queries with ease.

---
#### Snapshot Creation

Snapshot forms one of the fundamental abstractions of iceberg which unlocks a bunch of other features including time travel, re-constructing table state, views, etc. Every time an Iceberg table is updated—whether through insertions, deletions, or schema changes—Iceberg creates a new snapshot.

Each snapshot writes these files:
- root metadata file
- manifest list file
- manifest file
- data file/s

Every snapshot writes it's own root metadata file making two writes writing at the same time possible, final decision of which write gets registered happens further on catalog. We will discuss this exact scenario and it's nitties-gritties in an upcoming blogpost.
#### File Hierarchy in Iceberg

// TODO: Add tree command output here
// TODO: Add a block diagram to show hierarchy of these files and how a manifest list file can point to multiple old manifest files
// TODO: add iceberg table diagrams

Let's first understand how these files are laid out and what do they contain on a higher level. Apache Iceberg organizes its data in a hierarchical structure of files:

1. **Root Metadata File**: The central metadata file containing the entire table state.

2. **Manifest Lists**: Files referenced by the root metadata file that point to manifest files.

3. **Manifest Files**: These files hold information about actual data files (such as Parquet, ORC or Avro).

4. **Data Files**: Contain the actual table data.

This structured metadata model enables Iceberg to manage data and snapshots efficiently, supporting time-travel capabilities with minimal overhead.
#### Hands On!

##### Setup

Now to study these files in detail, let's get our hands dirty and try to make these files on local with example provided here: https://iceberg.apache.org/spark-quickstart/ , as this example may change in the future, let's paste all details here just in case xD

This is the docker-compose file:

```
version: "3"

services:
  spark-iceberg:
    image: tabulario/spark-iceberg
    container_name: spark-iceberg
    build: spark/
    networks:
      iceberg_net:
    depends_on:
      - rest
      - minio
    volumes:
      - ./warehouse:/home/iceberg/warehouse
      - ./notebooks:/home/iceberg/notebooks/notebooks
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    ports:
      - 8888:8888
      - 8080:8080
      - 10000:10000
      - 10001:10001
  rest:
    image: tabulario/iceberg-rest
    container_name: iceberg-rest
    networks:
      iceberg_net:
    ports:
      - 8181:8181
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
      - CATALOG_WAREHOUSE=s3://warehouse/
      - CATALOG_IO__IMPL=org.apache.iceberg.aws.s3.S3FileIO
      - CATALOG_S3_ENDPOINT=http://minio:9000
  minio:
    image: minio/minio
    container_name: minio
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
      - MINIO_DOMAIN=minio
    networks:
      iceberg_net:
        aliases:
          - warehouse.minio
    ports:
      - 9001:9001
      - 9000:9000
    command: ["server", "/data", "--console-address", ":9001"]
  mc:
    depends_on:
      - minio
    image: minio/mc
    container_name: mc
    networks:
      iceberg_net:
    environment:
      - AWS_ACCESS_KEY_ID=admin
      - AWS_SECRET_ACCESS_KEY=password
      - AWS_REGION=us-east-1
    entrypoint: >
      /bin/sh -c "
      until (/usr/bin/mc config host add minio http://minio:9000 admin password) do echo '...waiting...' && sleep 1; done;
      /usr/bin/mc rm -r --force minio/warehouse;
      /usr/bin/mc mb minio/warehouse;
      /usr/bin/mc policy set public minio/warehouse;
      tail -f /dev/null
      "
networks:
  iceberg_net:
```

We save this file and run `docker-compose up` on it. Once all the containers are up, enter a spark-sql prompt to make our table and finally do a write.

```
docker exec -it spark-iceberg spark-sql
```

We will create a table using:

```
CREATE TABLE demo.nyc.taxis
(
  vendor_id bigint,
  trip_id bigint,
  trip_distance float,
  fare_amount double,
  store_and_fwd_flag string
) PARTITIONED BY (vendor_id);
```

At this state, let's have a look at what metadata files and folders we have. All our work is persisted on minio, let's explore it by visiting http://localhost:9001/ and entering credentials as `admin` and `password`.

Currently our tree looks like this:
```
.
├── warehouse
├── nyc
│   ├── taxis
│       ├── metadata
│           └── 00000-d9fade81-c556-4458-8e30-e50151a43fed.metadata.json

4 directories, 1 files
```

Now navigate to `warehouse > nyc > taxis > metadata` (note we don't have a data folder yet here), over here you would notice a file with format `00000-<uuid>.metadata.json`, this is our table metadata file. A snapshot file created to indicate a new table is made.

##### Root table metadata file

On downloading the file and viewing it, we get:
```
{
  "format-version" : 2,
  "table-uuid" : "64919af4-5340-4aba-aa5a-93002be409b0",
  "location" : "s3://warehouse/nyc/taxis",
  "last-sequence-number" : 0,
  "last-updated-ms" : 1731430316331,
  "last-column-id" : 5,
  "current-schema-id" : 0,
  "schemas" : [ {
    "type" : "struct",
    "schema-id" : 0,
    "fields" : [ {
      "id" : 1,
      "name" : "vendor_id",
      "required" : false,
      "type" : "long"
    }, {
      "id" : 2,
      "name" : "trip_id",
      "required" : false,
      "type" : "long"
    }, {
      "id" : 3,
      "name" : "trip_distance",
      "required" : false,
      "type" : "float"
    }, {
      "id" : 4,
      "name" : "fare_amount",
      "required" : false,
      "type" : "double"
    }, {
      "id" : 5,
      "name" : "store_and_fwd_flag",
      "required" : false,
      "type" : "string"
    } ]
  } ],
  "default-spec-id" : 0,
  "partition-specs" : [ {
    "spec-id" : 0,
    "fields" : [ {
      "name" : "vendor_id",
      "transform" : "identity",
      "source-id" : 1,
      "field-id" : 1000
    } ]
  } ],
  "last-partition-id" : 1000,
  "default-sort-order-id" : 0,
  "sort-orders" : [ {
    "order-id" : 0,
    "fields" : [ ]
  } ],
  "properties" : {
    "owner" : "root",
    "write.parquet.compression-codec" : "zstd"
  },
  "current-snapshot-id" : -1,
  "refs" : { },
  "snapshots" : [ ],
  "statistics" : [ ],
  "partition-statistics" : [ ],
  "snapshot-log" : [ ],
  "metadata-log" : [ ]
}
```

This does not contain any snapshots, as mentioned earlier, it is only an indication of a new table being made. It stores important info like schema, partition spec, location, etc.

Now let's make a write and see what changes under `warehouse > nyc > taxis`.

We will insert entries like this:

```
INSERT INTO demo.nyc.taxis
VALUES (1, 1000371, 1.8, 15.32, 'N'), (2, 1000372, 2.5, 22.15, 'N'), (2, 1000373, 0.9, 9.01, 'N'), (1, 1000374, 8.4, 42.13, 'Y');
```

```
.
├── warehouse
├── nyc
│   ├── taxis
│       ├── metadata
│           └── 00000-d9fade81-c556-4458-8e30-e50151a43fed.metadata.json

4 directories, 1 files

.
├── warehouse
├── nyc
│   ├── taxis
│   │   ├── data
│   │   │   ├── asdf
│   │   │   ├── PART2.md
│	│	├── metadata 
│   ├── ld_preload
│   │   ├── PART1.md
│   │   └── PART2.md
│   ├── libuv-io-uring
│   │   ├── DRAFT_README.md
│   │   └── README.md
│   └── ll-hls-in-depth
│       └── README.md
├── 2024
│   ├── bf-interpreter
│   │   ├── PART1.md
│   │   ├── PART2.md
│   │   └── PART3.md
│   ├── faster-shell-boot
│   │   └── README.md
│   ├── iceberg-snapshot
│   │   ├── Iceberg_Snapshots.md
│   │   └── Iceberg_snapshots_karthic.md
│   └── writing-more
│       └── README.md
└── README.md

13 directories, 15 files
```

We have a new folder named `data`, so our data has been saved successfully, but there are also changes under `metadata` dir, you will see three new files, two ending with `.avro` -- we will explore these later -- and one of format `00001-uuid.metadata.json`, this is the root metadata file of snapshot we were talking about!

##### Snapshot file

Let's download our new json file from browser and explore it:

```
{
  "format-version" : 2,
  "table-uuid" : "64919af4-5340-4aba-aa5a-93002be409b0",
  "location" : "s3://warehouse/nyc/taxis",
  "last-sequence-number" : 1,
  "last-updated-ms" : 1731430821537,
  "last-column-id" : 5,
  "current-schema-id" : 0,
  "schemas" : [ {
    "type" : "struct",
    "schema-id" : 0,
    "fields" : [ {
      "id" : 1,
      "name" : "vendor_id",
      "required" : false,
      "type" : "long"
    }, {
      "id" : 2,
      "name" : "trip_id",
      "required" : false,
      "type" : "long"
    }, {
      "id" : 3,
      "name" : "trip_distance",
      "required" : false,
      "type" : "float"
    }, {
      "id" : 4,
      "name" : "fare_amount",
      "required" : false,
      "type" : "double"
    }, {
      "id" : 5,
      "name" : "store_and_fwd_flag",
      "required" : false,
      "type" : "string"
    } ]
  } ],
  "default-spec-id" : 0,
  "partition-specs" : [ {
    "spec-id" : 0,
    "fields" : [ {
      "name" : "vendor_id",
      "transform" : "identity",
      "source-id" : 1,
      "field-id" : 1000
    } ]
  } ],
  "last-partition-id" : 1000,
  "default-sort-order-id" : 0,
  "sort-orders" : [ {
    "order-id" : 0,
    "fields" : [ ]
  } ],
  "properties" : {
    "owner" : "root",
    "write.parquet.compression-codec" : "zstd"
  },
  "current-snapshot-id" : 5288760946744945553,
  "refs" : {
    "main" : {
      "snapshot-id" : 5288760946744945553,
      "type" : "branch"
    }
  },
  "snapshots" : [ {
    "sequence-number" : 1,
    "snapshot-id" : 5288760946744945553,
    "timestamp-ms" : 1731430821537,
    "summary" : {
      "operation" : "append",
      "spark.app.id" : "local-1731429992596",
      "added-data-files" : "2",
      "added-records" : "4",
      "added-files-size" : "3074",
      "changed-partition-count" : "2",
      "total-records" : "4",
      "total-files-size" : "3074",
      "total-data-files" : "2",
      "total-delete-files" : "0",
      "total-position-deletes" : "0",
      "total-equality-deletes" : "0"
    },
    "manifest-list" : "s3://warehouse/nyc/taxis/metadata/snap-5288760946744945553-1-34ebd771-5245-4ca7-8cab-d80b094bbcd0.avro",
    "schema-id" : 0
  } ],
  "statistics" : [ ],
  "partition-statistics" : [ ],
  "snapshot-log" : [ {
    "timestamp-ms" : 1731430821537,
    "snapshot-id" : 5288760946744945553
  } ],
  "metadata-log" : [ {
    "timestamp-ms" : 1731430316331,
    "metadata-file" : "s3://warehouse/nyc/taxis/metadata/00000-0fbb5ecb-6357-4571-bd80-97ba28994012.metadata.json"
  } ]
}
```

We see a lot more new fields populated this time! A snapshot file has these fields:

- `snapshot-id`
- `parent-snapshot-id`
- `sequence-number`
- `timestamp-ms`
- `manifest-list`
- `manifests`
- `summary`
- `schema-id`
- `first-row-id`

as listed here: https://iceberg.apache.org/spec/#snapshots

We can cross check struct json to make sure our writes were properly registered. We did an insert so `operation` has an `append` type. We added 4 records reflected by `added-records`. Number of `data` files added is told by `added-data-files` , we can cross check it by visiting `warehouse > nyc > taxis > data` , we see two folders each containing a single file placed according to partition key.

##### Manifest List file

Manifest list file as the name suggests lists down manifest file(s) associated with a particular snapshot. These manifest files contain data relating to a particular row, so let's say if a row is inserted on snapshot S1(=M1 -- corresponding manifest file) and updated later on snapshot S2(=M2 -- corresponding manifest file). Manifest list of S2 will contain M1 and M2 both. 

As this is an avro file so we can't inspect it's text directly. But we can leverage `spark-sql` to query these files. Firstly, let's try to see snapshot by running:

```
select snapshot_id, manifest_list from demo.nyc.taxis.snapshots;
```

we get output as:

```
8253356480484172902     s3://warehouse/nyc/taxis/metadata/snap-8253356480484172902-1-b8ea2e6e-979f-4511-a3dd-9d155ced2da1.avro
```

We see our manifest list filed given as `s3://warehouse/nyc/taxis/metadata/snap-8253356480484172902-1-b8ea2e6e-979f-4511-a3dd-9d155ced2da1.avro`. Next let's check contents of this manifest list by running:

```
select * from demo.nyc.taxis.manifests;
```

we get output as:

```
0       s3://warehouse/nyc/taxis/metadata/b8ea2e6e-979f-4511-a3dd-9d155ced2da1-m0.avro  7211    0       8253356480484172902     2       0       0       0       0       0       [{"contains_null":false,"contains_nan":false,"lower_bound":"1","upper_bound":"2"}]
```

we see this manifest list file contain one single mainfest file `s3://warehouse/nyc/taxis/metadata/b8ea2e6e-979f-4511-a3dd-9d155ced2da1-m0.avro`, and some metadata: `[{"contains_null":false,"contains_nan":false,"lower_bound":"1","upper_bound":"2"}]`

these are some quick statistics:
- does contain null values
- does not contain nan values
- `vendor_id` has lower bound as 1
- `vendor_id` has upper bound as 2

##### Manifest file

Now let's check contents of manifest file by running:

```
select * from demo.nyc.taxis.files;
```

we get output as:

```
0       s3://warehouse/nyc/taxis/data/vendor_id=1/00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00001.parquet  PARQUET 0       {"vendor_id":1} 2       1516    {1:70,2:48,3:40,4:48,5:42}      {1:2,2:2,3:2,4:2,5:2}   {1:0,2:0,3:0,4:0,5:0}   {3:0,4:0}    {1:,2:�C,3:ff�?,4:�p=
ף.@,5:N}        {1:,2:�C,3:ffA,4:q=
ףE@,5:Y}        NULL    [4]     NULL    0       {"fare_amount":{"column_size":48,"value_count":2,"null_value_count":0,"nan_value_count":0,"lower_bound":15.32,"upper_bound":42.13},"store_and_fwd_flag":{"column_size":42,"value_count":2,"null_value_count":0,"nan_value_count":null,"lower_bound":"N","upper_bound":"Y"},"trip_distance":{"column_size":40,"value_count":2,"null_value_count":0,"nan_value_count":0,"lower_bound":1.8,"upper_bound":8.4},"trip_id":{"column_size":48,"value_count":2,"null_value_count":0,"nan_value_count":null,"lower_bound":1000371,"upper_bound":1000374},"vendor_id":{"column_size":70,"value_count":2,"null_value_count":0,"nan_value_count":null,"lower_bound":1,"upper_bound":1}}

---------

0       s3://warehouse/nyc/taxis/data/vendor_id=2/00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00002.parquet  PARQUET 0       {"vendor_id":2} 2       1558    {1:70,2:48,3:40,4:48,5:67}      {1:2,2:2,3:2,4:2,5:2}   {1:0,2:0,3:0,4:0,5:0}   {3:0,4:0}    {1:,2:�C,3:fff?,4:��Q�"@,5:N}   {1:,2:�C,3: @,4:fffff&6@,5:N}   NULL    [4]     NULL    0       {"fare_amount":{"column_size":48,"value_count":2,"null_value_count":0,"nan_value_count":0,"lower_bound":9.01,"upper_bound":22.15},"store_and_fwd_flag":{"column_size":67,"value_count":2,"null_value_count":0,"nan_value_count":null,"lower_bound":"N","upper_bound":"N"},"trip_distance":{"column_size":40,"value_count":2,"null_value_count":0,"nan_value_count":0,"lower_bound":0.9,"upper_bound":2.5},"trip_id":{"column_size":48,"value_count":2,"null_value_count":0,"nan_value_count":null,"lower_bound":1000372,"upper_bound":1000373},"vendor_id":{"column_size":70,"value_count":2,"null_value_count":0,"nan_value_count":null,"lower_bound":2,"upper_bound":2}}
```

We see two entries returned corresponding to two parquet data files we have at locations:
- s3://warehouse/nyc/taxis/data/vendor_id=1/00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00001.parquet
-  s3://warehouse/nyc/taxis/data/vendor_id=2/00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00002.parquet

We are using `PARQUET` format to store data and remaining details are much more detailed statistics as against light weight stats stored in manifest list. This helps a lot in trimming data before reading all data files in detail. In the last column of our result we have specific stats for each of the column, for `fare_amount`:
- fare_amount
	- column's size is 48
	- value's count is 2
	- there are no null values
	- there are no nan values
	- lower bound is 9.02
	- and upper bound is 22.15

##### Parquet data file

We can query back our data using:

```
select * from demo.nyc.taxis;
```

But how about we check parquet files directly, I am going to use a nice Rust based parquet reader CLI: https://github.com/manojkarthick/pqrs

pqrs cat is used to inspect contents of a parquet file, for our case it would be:

```
pqrs cat 00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00001.parquet
pqrs cat 00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00002.parquet
```

this gives output:

```
##################################################################
File: 00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00001.parquet
##################################################################

{vendor_id: 1, trip_id: 1000371, trip_distance: 1.8, fare_amount: 15.32, store_and_fwd_flag: "N"}
{vendor_id: 1, trip_id: 1000374, trip_distance: 8.4, fare_amount: 42.13, store_and_fwd_flag: "Y"}
```

and

```
##################################################################
File: 00000-4-49cbfb21-063f-41d3-b9bf-04a6a991739a-0-00002.parquet
##################################################################

{vendor_id: 2, trip_id: 1000372, trip_distance: 2.5, fare_amount: 22.15, store_and_fwd_flag: "N"}
{vendor_id: 2, trip_id: 1000373, trip_distance: 0.9, fare_amount: 9.01, store_and_fwd_flag: "N"}
```

giving us the four entries exactly as we have inserted in our query!

---

#### Leveraging Snapshots for Time Travel Queries

With Iceberg, we can query historical data states as they existed in any past snapshot. Using the snapshot log, we can retrieve and use any previous snapshot ID to access the table’s state at that specific point in time.

Let's see this in action, we currently only have one snapshot, let's make them two and then query our first snapshot:

```
INSERT INTO demo.nyc.taxis
VALUES (3, 1000671, 2.1, 11.67, 'W');
```

On listing all rows in our table we get:

```
3       1000671 2.1     11.67   W
1       1000371 1.8     15.32   N
1       1000374 8.4     42.13   Y
2       1000372 2.5     22.15   N
2       1000373 0.9     9.01    N
```

Now we can load our first snapshot by specifying the snapshot ID:

```
select * from demo.nyc.taxis version as of 8253356480484172902;
```

we get output as:

```
1       1000371 1.8     15.32   N
1       1000374 8.4     42.13   Y
2       1000372 2.5     22.15   N
2       1000373 0.9     9.01    N
```

Our new entry is not present!

This time-travel capability is essential for analytical and auditing purposes, providing flexibility in data management.

---
#### Conclusion

Iceberg’s snapshot creation and time travel features are built on a robust metadata structure, with the root metadata file acting as the primary point of reference for the table’s state.

Iceberg's snapshot management system offers a unique combination of version history, time-travel queries, and efficient metadata management, making it an ideal choice for data lake environments requiring robust and scalable table formats.