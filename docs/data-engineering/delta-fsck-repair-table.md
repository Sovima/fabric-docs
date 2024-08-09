---
title: Delta Lake FSCK REPAIR TABLE command
description: Learn how to fix your delta table when the delta log contains references to missing files
ms.reviewer: snehagunda
ms.author: t-sofyam
author: DaniBunny
ms.topic: 
ms.custom:
  - None
  - 
ms.date: 
ms.search.form: 
---

# Delta Lake FSCK REPAIR TABLE command

Sometimes, the files that are referenced within the delta log may get removed from the file system. In that case, the user will start running into an error similar to `org.apache.spark.SparkFileNotFoundException: File <FilePath> does not exist`. The FSCK command is intended to make the table usable again. Specifically by removing the files referenced in the transaction log that are not found in the filesystem. This guide will go over how to use the `FSCK REPAIR TABLE` command and how that might affect the stored data.

## How to avoid using FSCK?

Note that running the FSCK command does not preserve data constistency so it is better to avoid using this command in the first place. While sometimes the files can get removed randomly, there are some specific actions taken by the user that can lead to the `File does not exist` error. The following sequence of actions is likely to lead to this scenario:
1. Create a table, insert data into it and delete some of the data.
2. Clean up the files unreferenced in the transaction log by running the VACUUM command.
3. Restore the table to the version before the vacuum retention period (by default it's set to 7 days).

Now, the table we have has references to files that were vacuumed which means the table is not usable. Not performing this sequence of actions will decrease your changes of having to use this command. However, if do run into `File does not exist` errors, see below for how to correctly use the `FSCK REPAIR TABLE` command to fix the table.

## How will the data be affected?

There are two options (based on the `spark.databricks.delta.fsck.missingDVsResolution` config) for what might happen when missing deletion vectors are found:
- By default, the command throws an exception when a missing deletion vector is found. In this case, no changes get committed so the table is unchanged. 
- The second option is the deletion vector is deleted from the delta log and the accosiated parquet file is unchanged. In this scenario, we might see the deleted entries come back, or we might see duplicate entries (if an entry was updated).

In terms of missing parquet files, since running the FSCK command removes references to the parquet files, the data that was contained in those parquet files will not be in the table anymore. 

__Make sure to keep data consistency in mind when running this command.__

# Syntax

```sql
FSCK REPAIR TABLE <table|delta.fileOrFolderPath> [DRY RUN]
```

## Parameters

- `<table|fileOrFolderPath>`
  Reference to __an existing__ delta table. The command supports spaces in table names, full, and relative paths. 
- `DRY RUN`
  
  __Note__: it is strongly encouraged to run the command in dry run mode first before running it without dry run to get an idea what files will be removed before they are removed.
  
  Running the command in `DRY RUN` mode provides insights into which files are present in the transaction log (both parquet and .bin files) and are missing in the filesystem without changing the delta log. Speficially, the generated output will provde file paths, deletion vector paths, and which file is missing in the filesystem. By default, the first `1000` files are displayed but that limit can be changed by updating the `spark.databricks.delta.fsck.maxNumEntriesInResult` config. This limit will only be applied to DRY RUN mode. When ran without DRY RUN, all of the deleted files will be displayed.

## Configs
`spark.databricks.delta.fsck.maxNumEntriesInResult` - used to set the limit for how many 
`spark.databricks.delta.fsck.missingDVsResolution` - can be set to "exception" (default) or "removeDVs". By default, an exception is thrown when a deletion vector is detected. If the "removeDVs" option is selected, the transaction log entries that have a missing deletion vector are removed and then added back but with no accosiated deletion vector. That way we are preserving the associated parquet file but extra data might return due to a removed deletion vector. 

## Returns
In both cases (`DRY RUN` and normal), the following columns will be reported to the user:
- `dataFilePath STRING NOT NULL` - path of the missing file
- `dataFileMissing BOOLEAN NOT NULL` - whether the data file is missing on disk or not. Since only the missing files are reported, in this version of the command this column is always set to True

# Examples

## Example 1 

Assume here that the transaction log for table `t` contains `file1.parquet`, `file2.parquet`, and `file3.parquet` but `file1.parquet` is missing on disk and the deletion vector associated with file2 is missing. This run is assuming `spark.databricks.delta.fsck.missingDVsEnabled` set to `removeDV`.
```sql
> FSCK REPAIR TABLE t DRY RUN;
Found (1) file(s) with missing deletion vectors.
Found (1) file(s) to be removed from the delta log. Listing all row
+--------------------+---------------+------------------+-------------------------+
|        dataFilePath|dataFileMissing|deletionVectorPath|deletionVectorFileMissing|
+--------------------+---------------+------------------+-------------------------+
|file1.parquet       |           true|              NULL|                    false|
+--------------------+---------------+------------------+-------------------------+
|file2.parquet       |          false| deletion_vector2.|                    true |
+--------------------+---------------+------------------+-------------------------+
> FSCK REPAIR TABLE t;
Found (1) file(s) with missing deletion vectors.
Removed (1) file(s) from the delta log.
+--------------------+---------------+------------------+-------------------------+
|        dataFilePath|dataFileMissing|deletionVectorPath|deletionVectorFileMissing|
+--------------------+---------------+------------------+-------------------------+
|file1.parquet       |           true|              NULL|                    false|
+--------------------+---------------+------------------+-------------------------+
|file2.parquet       |          false| deletion_vector2.|                    true |
+--------------------+---------------+------------------+-------------------------+
```

## Example 2
Assume here that the transaction log for table `t` contains partitions `file1.parquet`, `file2.parquet`, and `file3.parquet` and each of the files has an associated deletion vector. The deletion vectors for files `file2.parquet` and `file3.parquet` are missing.
```sql
> FSCK REPAIR TABLE t DRY RUN;
Found (2) file(s) with missing deletion vectors.
Found (3) file(s) to be removed from the delta log. Listing all row
+--------------------+---------------+------------------+-------------------------+
|        dataFilePath|dataFileMissing|deletionVectorPath|deletionVectorFileMissing|
+--------------------+---------------+------------------+-------------------------+
|file1.parquet       |           true| deletion_vector1.|                    false|
+--------------------+---------------+------------------+-------------------------+
|file2.parquet       |          true | deletion_vector2.|                    true |
+--------------------+---------------+------------------+-------------------------+
|file3.parquet       |          true | deletion_vector3.|                    true |
+--------------------+---------------+------------------+-------------------------+
> FSCK REPAIR TABLE t;
Found (2) file(s) with missing deletion vectors.
Removed (3) file(s) from the delta log.
+--------------------+---------------+------------------+-------------------------+
|        dataFilePath|dataFileMissing|deletionVectorPath|deletionVectorFileMissing|
+--------------------+---------------+------------------+-------------------------+
|file1.parquet       |           true| deletion_vector1.|                    false|
+--------------------+---------------+------------------+-------------------------+
|file2.parquet       |          true | deletion_vector2.|                    true |
+--------------------+---------------+------------------+-------------------------+
|file3.parquet       |          true | deletion_vector3.|                    true |
+--------------------+---------------+------------------+-------------------------+
```

## Related content

- [What is Delta Lake?](/azure/synapse-analytics/spark/apache-spark-what-is-delta-lake)
- [Lakehouse and Delta Lake](lakehouse-and-delta-tables.md)

