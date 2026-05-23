### Snapshot Creation and Time Travel in Apache Iceberg



Apache Iceberg has revolutionized data management in data lake architectures by offering robust metadata management and data snapshotting capabilities. This post dives deep into how snapshot creation and time travel work in Iceberg, exploring the metadata file structure, snapshot log, and the atomic swap mechanism that ensures consistency. Using PyIceberg examples and Polaris for demonstrations, we’ll illustrate how Iceberg allows consistent data state retrieval and time-travel queries with ease.



---



#### File Hierarchy in Iceberg



Apache Iceberg organizes its data in a hierarchical structure of files:

1. **Root Metadata File**: The central metadata file containing the entire table state.

2. **Manifest Lists**: Files referenced by the root metadata file that point to manifest files.

3. **Manifest Files**: These files hold information about actual data files (such as Parquet or ORC).

4. **Data Files**: Contain the actual table data.



This structured metadata model enables Iceberg to manage data and snapshots efficiently, supporting time-travel capabilities with minimal overhead.



---



#### Snapshot Creation and the Root Metadata File



Every time an Iceberg table is updated—whether through insertions, deletions, or schema changes—Iceberg creates a new version of the **root metadata file**. This file is the top of the metadata hierarchy and acts as a pointer to the latest state of the table. Here’s how it works:

- **Versioning**: Each root metadata file is versioned and stored separately (e.g., `v1.metadata.json`, `v2.metadata.json`).

- **Atomic Swap**: The catalog’s atomic swap mechanism ensures that only one root metadata file points to the current state at any given time. This prevents conflicts when multiple writers update the table concurrently.



For illustration, let’s look at an example using PyIceberg.



#### Snapshot Creation with PyIceberg Example



Here’s how you could create a snapshot and see it registered in Iceberg using PyIceberg:



```python

from pyiceberg.catalog import Catalog, load_catalog

from pyiceberg.schema import Schema

from pyiceberg.table import Table

from pyiceberg.expressions import Ref, Equals



# Load catalog

catalog = load_catalog("my_catalog")



# Initialize or load an Iceberg table

table = catalog.load_table("my_namespace.my_table")



# Insert new data and create a snapshot

table.append([{"id": 1, "value": "example"}])



# Retrieve latest snapshot information

latest_snapshot = table.current_snapshot()

print(f"Snapshot ID: {latest_snapshot.snapshot_id}, Timestamp: {latest_snapshot.timestamp_ms}")

```



Each insertion or data modification in the table creates a new snapshot, stored as a unique ID with metadata information. In Iceberg, snapshots act like checkpoints, allowing access to specific table versions and enabling time-travel queries.



---



#### Viewing Snapshots with Polaris



Using Polaris, we can visually observe and manage snapshots in Iceberg. When a new data change is committed, Iceberg updates the root metadata file and registers the snapshot in the snapshot log. Polaris will display the latest snapshot with all prior history available for reference or time travel.



**Atomic Swap and Catalog API**



Iceberg’s catalog uses an atomic swap to point to the latest root metadata file, ensuring that the table state is consistent. The core API in Iceberg for handling this atomic swap is:



```java

boolean commit(TableIdentifier identifier, TableMetadata base, TableMetadata metadata)

```



- **identifier**: The table being updated.

- **base**: Current metadata of the table.

- **metadata**: New metadata to commit.

- **Return**: `true` if the commit was successful; `false` otherwise (often due to concurrent changes).



This API performs an atomic compare-and-swap (CAS) operation to update the root pointer only if the base metadata matches the catalog’s current state. This mechanism ensures serializable isolation, where changes are committed only when the latest metadata file is referenced.



---



#### Structure of the Root Metadata File



The root metadata file (`metadata.json`) not only tracks the table schema, partition specs, and properties but also maintains a log of **snapshots**. This log is key for time travel and version history.



Example of the snapshot log structure within `metadata.json`:



```json

{

  "snapshot-log": [

    {"timestamp-ms": 1672480478123, "snapshot-id": 1001},

    {"timestamp-ms": 1672560878123, "snapshot-id": 1002},

    {"timestamp-ms": 1672641278123, "snapshot-id": 1003}

  ]

}

```



Each snapshot entry in this log includes:

- **Timestamp**: When the snapshot was created.

- **Snapshot ID**: Unique identifier for the snapshot.

- **Manifest List Reference**: Points to the manifest files associated with this snapshot.



---



#### Leveraging Snapshots for Time Travel Queries



With Iceberg, you can query historical data states as they existed in any past snapshot. Using the snapshot log, you can retrieve and use any previous snapshot ID to access the table’s state at that specific point in time.



For example, in PyIceberg, you can load a previous snapshot by specifying the snapshot ID:



```python

# Load a specific snapshot

snapshot_id = 1002  # replace with the desired snapshot ID

table = catalog.load_table("my_namespace.my_table", snapshot_id=snapshot_id)

```



This time-travel capability is essential for analytical and auditing purposes, providing flexibility in data management.



---



#### Conclusion



Iceberg’s snapshot creation and time travel features are built on a robust metadata structure, with the root metadata file acting as the primary point of reference for the table’s state. The atomic swap mechanism in the catalog API ensures that only one version of the table’s root metadata file is current, facilitating strong consistency and serializable isolation.



Iceberg's snapshot management system offers a unique combination of version history, time-travel queries, and efficient metadata management, making it an ideal choice for data lake environments requiring robust and scalable table formats.
