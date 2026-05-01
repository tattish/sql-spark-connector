<p align="center">
  <img src="sql-spark-connector-icon.svg" alt="Apache Spark Connector for SQL Server and Azure SQL" width="150"/>
</p>

# This is a repo to give LTS for the SQL-connector for SPARK

This connector in many cases for our customers is a way better solution then the built in JDBC for Databricks, so we need to use it in performance critical scenarios for MSSQL server, like loading very large datasets to PBI efficiently. The process for loading, DBX-MSSQL-PBI is extrem less resource intensive for the PBI resource and for this cases many times faster then the large loading from the Databricks SQL endpoint. The issues is the segment creation capability of PBI, and in general that the SQL server capacity is a way cheaper resource then the PBI computing capacity. This way the more efficient segment compression between MSSQL to PBI is making it more economical then the DBX-SQL to PBI workflow. Direct connection from DBX to MSSQL is also a lot faster for reading and writing then the native DBX built in resource due to MPP model of the SPARK connector.

This readme file contains lot of content from the original project, some for the build added.

## Important Notice (Feb 2025) No official MS support, the last supported DBX version was 14.3 LTS
This project is no longer maintained. For any interested parties, please feel free to fork and maintain it on GitHub.

# Apache Spark Connector for SQL Server and Azure SQL
Born out of Microsoft's SQL Server Big Data Clusters investments, the Apache Spark Connector for SQL Server and Azure SQL is a high-performance connector that enables you to use transactional data in big data analytics and persists results for ad-hoc queries or reporting. The connector allows you to use any SQL database, on-premises or in the cloud, as an input data source or output data sink for Spark jobs.

This library contains the source code for the Apache Spark Connector for SQL Server and Azure SQL.

[Apache Spark](https://spark.apache.org/) is a unified analytics engine for large-scale data processing.

There are three version sets of the connector available through Maven, a 2.4.x, a 3.0.x and a 3.1.x compatible version. All versions can be found [here](https://search.maven.org/search?q=spark-mssql-connector) and can be imported using the coordinates below:

| Connector                        | Maven Coordinate                                       | Scala Version |
|----------------------------------|--------------------------------------------------------|---------------|
| Spark 2.4.x compatible connector | `com.microsoft.azure:spark-mssql-connector:1.0.2`      | 2.11          |
| Spark 3.0.x compatible connector | `com.microsoft.azure:spark-mssql-connector_2.12:1.1.0` | 2.12          |
| Spark 3.1.x compatible connector | `com.microsoft.azure:spark-mssql-connector_2.12:1.2.0` | 2.12          |
| Spark 3.3.x compatible connector | `com.microsoft.azure:spark-mssql-connector_2.12:1.3.0` | 2.12          |
| Spark 3.4.x compatible connector | `com.microsoft.azure:spark-mssql-connector_2.12:1.4.0` | 2.12          |
| Spark 3.5.x compatible connector | `com.microsoft.azure:spark-mssql-connector_2.12:1.5.0` | 2.12          |

## Current Releases

The latest Spark 2.4.x compatible connector is on v1.0.2.

The latest Spark 3.0.x compatible connector is on v1.1.0.

The latest Spark 3.1.x compatible connector is on v1.2.0.

For main changes from previous releases and known issues please refer to [CHANGELIST](docs/CHANGELIST.md)

## Databricks Runtime 15.4 LTS Compatibility

This fork adds support for **Databricks Runtime 15.4 LTS** (Spark 3.5.1, Scala 2.12.15). The following changes were made to enable compatibility.

### Changes Summary

#### `pom.xml`
- Added a new `spark35` Maven profile (active by default) targeting Spark 3.5.1, Scala 2.12.15
- The original `spark34` profile is retained for backward compatibility
- `maven-compiler-plugin` updated to 3.13.0, source/target set to Java 8 (`1.8`) — DBR 15.4 LTS executes JARs on a Java 8 JVM despite using a newer toolchain
- `scala-maven-plugin` downgraded to 4.8.1 to avoid pulling in `sbt-compiler-bridge 1.10.0` which was compiled with Java 11 (class file version 55.0) and is incompatible with the cluster's Java 8 runtime
- `-target:jvm-1.8` added as a Scala compiler argument to ensure Java 8 bytecode output
- `scalatest-maven-plugin` updated to 2.2.0
- `scalatest` dependency updated to 3.2.18
- `maven-javadoc-plugin` updated to 3.6.3
- `mssql-jdbc` updated to `12.6.1.jre8` (the `jre8` classifier is required — `jre11` produces class file version 55.0 which the cluster rejects)

#### `BulkCopyUtils.scala`
- `JdbcUtils.getSchema` is called via reflection to handle the breaking API change between Databricks and open-source Spark. Databricks Runtime 15.4 LTS uses a custom 5-parameter signature `getSchema(Connection, ResultSet, JdbcDialect, Boolean, Boolean)` that does not exist in any published open-source Spark release. The connector detects the available signature at runtime and calls the correct overload:

  | Runtime | Parameters | Signature |
  |---|---|---|
  | Databricks Runtime 15.4 LTS | 5 | `(Connection, ResultSet, JdbcDialect, false, false)` |
  | Spark 3.5 open-source | 2 | `(ResultSet, JdbcDialect)` |
  | Spark 3.4 open-source | 3 | `(ResultSet, JdbcDialect, false)` |

### Building for Databricks Runtime 15.4 LTS

**Prerequisites:**
- Java 8 JDK (e.g. [Azul Zulu 8](https://www.azul.com/downloads/)) — must use Java 8 to compile, not Java 11 or later
- Maven 3.6+

**Set JAVA_HOME to your Java 8 JDK before building:**
```powershell
# Windows (PowerShell) — adjust path to your JDK 8 install location
$env:JAVA_HOME = "C:\Program Files\Zulu\zulu-8"
$env:Path = "$env:JAVA_HOME\bin;" + $env:Path
mvn -version  # should show Java version: 1.8.x
```

**Build:**
```powershell
mvn package "-DskipTests" "-Dgpg.skip=true" -Pspark35
```

The output JAR will be at `target/spark-mssql-connector-X.X.X.jar`.

**Verify the bytecode version is Java 8 (52.0):**
```powershell
javap -verbose -classpath target\spark-mssql-connector-1.4.0.jar com.microsoft.sqlserver.jdbc.spark.ColumnMetadata | findstr "major version"
# Expected: major version: 52
```

**Build for legacy Spark 3.4:**
```powershell
mvn package "-DskipTests" "-Dgpg.skip=true" -Pspark34
```

### Known Issues and Fixes

#### `java.lang.NoSuchMethodError: JdbcUtils$.getSchema`
Databricks Runtime uses a forked internal Spark build where `JdbcUtils.getSchema` has a different signature than open-source Spark. This is handled via runtime reflection in `BulkCopyUtils.scala` — no action required if using this fork.

#### `java.lang.UnsupportedClassVersionError: class file version 55.0`
The JAR or one of its dependencies was compiled with Java 11. Ensure:
- Your `JAVA_HOME` points to a Java 8 JDK when building
- You are using `mssql-jdbc` with the `jre8` classifier, not `jre11`
- `scala-maven-plugin` is on version `4.8.1` (not `4.9.x` which pulls in a Java 11 compiler bridge)

#### `[WARNING] The requested profile "spark35" could not be activated`
Your local `pom.xml` does not contain the `spark35` profile — you are running against the original unmodified file. Replace it with the updated `pom.xml` from this fork.

#### PowerShell: `Unknown lifecycle phase ".skip=true"`
PowerShell splits `-D` flags at the `=` sign. Wrap each flag in quotes:
```powershell
mvn package "-DskipTests" "-Dgpg.skip=true" -Pspark35
```

---

## Supported Features
* Support for all Spark bindings (Scala, Python, R)
* Basic authentication and Active Directory (AD) Key Tab support
* Reordered DataFrame write support
* Support for write to SQL Server Single instance and Data Pool in SQL Server Big Data Clusters
* Reliable connector support for Sql Server Single Instance


| Component                            | Versions Supported                  |
|--------------------------------------|-------------------------------------|
| Apache Spark                         | 2.4.x, 3.0.x, 3.1.x, 3.3.x, 3.5.x |
| Scala                                | 2.11, 2.12                          |
| Microsoft JDBC Driver for SQL Server | 8.4.1 (Spark 3.4), 12.6.1 (Spark 3.5) |
| Microsoft SQL Server                 | SQL Server 2008 or later            |
| Azure SQL Databases                  | Supported                           |
| Databricks Runtime                   | 15.4 LTS (Spark 3.5.1)             |

*Note: Azure Synapse (Azure SQL DW) use is not tested with this connector. While it may work, there may be unintended consequences.*

### Supported Options
The Apache Spark Connector for SQL Server and Azure SQL supports the options defined here: [SQL DataSource JDBC](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)

In addition following options are supported
| Option | Default | Description |
| --------- | ------------------ | ------------------------------------------ |
| reliabilityLevel | "BEST_EFFORT" | "BEST_EFFORT" or "NO_DUPLICATES". "NO_DUPLICATES" implements an reliable insert in executor restart scenarios |
| dataPoolDataSource | none | none implies the value is not set and the connector should write to SQl Server Single Instance. Set this value to data source name to write a Data Pool Table in Big Data Cluster|
| isolationLevel | "READ_COMMITTED" | Specify the isolation level |
| tableLock | "false" | Implements an insert with TABLOCK option to improve write performance |
| schemaCheckEnabled | "true" | Disables strict dataframe and sql table schema check when set to false |

Other [Bulk api options](https://docs.microsoft.com/en-us/sql/connect/jdbc/using-bulk-copy-with-the-jdbc-driver?view=sql-server-2017#sqlserverbulkcopyoptions) can be set as options on the dataframe and will be passed to bulkcopy apis on write

## Performance comparison
Apache Spark Connector for SQL Server and Azure SQL is up to 15x faster than generic JDBC connector for writing to SQL Server. Note performance characteristics vary on type, volume of data,  options used and may show run to run variations. The following performance results are the time taken to overwrite a sql table with 143.9M rows in a spark dataframe. The spark dataframe is constructed by reading store_sales HDFS table generated using [spark TPCDS Benchmark](https://github.com/databricks/spark-sql-perf). Time to read store_sales to dataframe is excluded. The results are averaged over 3 runs.
*Note: The following results were achieved using the Apache Spark 2.4.5 compatible connector. These numbers are not a guarantee of performance.*

| Connector Type | Options | Description |  Time to write |
| --------- | ------------------ | -------------------------------------| ---------- |
| JDBCConnector | Default | Generic JDBC connector with default options |  1385s |
| sql-spark-connector | BEST_EFFORT | Best effort sql-spark-connector  with default options |580s |
| sql-spark-connector | NO_DUPLICATES | Reliable sql-spark-connector | 709s |
| sql-spark-connector | BEST_EFFORT + tabLock=true | Best effort sql-spark-connector with table lock enabled | 72s |
| sql-spark-connector | NO_DUPLICATES + tabLock=true| Reliable sql-spark-connector with table lock enabled| 198s |

Config
- Spark config : `num_executors = 20`, `executor_memory = '1664m'`, `executor_cores = 2`
- Data Gen config : `scale_factor=50`, `partitioned_tables=true`
- Data file Store_sales with number of of rows 143,997,590

Environment
- [SQL Server Big Data Cluster](https://docs.microsoft.com/en-us/sql/big-data-cluster/release-notes-big-data-cluster?view=sql-server-ver15) CU5
- Master + 6 nodes
- Each node gen 5 server, 512GB Ram, 4TB NVM per node, NIC 10GB

## Commonly Faced Issues

### `java.lang.NoClassDefFoundError: com/microsoft/aad/adal4j/AuthenticationException`

This issue arises from using an older version of the mssql driver (which is now included in this connector) in your hadoop environment. If you are coming from using the previous Azure SQL Connector and have manually installed drivers onto that cluster for AAD compatibility, you will need to remove those drivers.

Steps to fix the issue:

1. If you are using a generic Hadoop environment, check and remove the mssql jar: `rm $HADOOP_HOME/share/hadoop/yarn/lib/mssql-jdbc-6.2.1.jre7.jar`.
If you are using Databricks, add a global or cluster init script to remove old versions of the mssql driver from the `/databricks/jars` folder, or add this line to an existing script: `rm /databricks/jars/*mssql*`
2. Add the `adal4j` and `mssql` packages, I used Maven, but anyway should work. DO NOT install the SQL spark connector this way.
3. Add the driver class to your connection configuration:

```
connectionProperties = {
  "Driver": "com.microsoft.sqlserver.jdbc.SQLServerDriver"
}
```

For more information and explanation, visit the closed [issue](https://github.com/microsoft/sql-spark-connector/issues/26).

## Get Started

The Apache Spark Connector for SQL Server and Azure SQL is based on the Spark DataSourceV1 API and SQL Server Bulk API and uses the same interface as the built-in JDBC Spark-SQL connector. This allows you to easily integrate the connector and migrate your existing Spark jobs by simply updating the format parameter with `com.microsoft.sqlserver.jdbc.spark`.

To include the connector in your projects download this repository and build the jar using SBT.

### Migrating from Legacy Azure SQL Connector for Spark

#### Receiving `java.lang.NoClassDefFoundError` when trying to use the new connector with Azure Databricks?

If you are migrating from the previous Azure SQL Connector for Spark and have manually installed drivers onto that cluster for AAD compatibility, you will most likely need to remove those custom drivers, restore the previous drivers that ship by default with Databricks, uninstall the previous connector, and restart your cluster.  You may be better off spinning up a new cluster.

With this new connector, you should be able to simply install onto a cluster (new or existing cluster that hasn't had its drivers modified) or a cluster which previously used modified drivers for the older Azure SQL Connector for Spark provided the modified drivers were removed and the previous default drivers restored.

See [Issue #26](https://github.com/microsoft/sql-spark-connector/issues/26) for more details.

### Executing custom SQL through the connector
The previous Azure SQL Connector for Spark provided the ability to execute custom SQL code like DML or DDL statements through the connector. This functionality is out-of-scope of this connector since it is based on the DataSource APIs. This functionality is readily provided by libraries like pyodbc or you can use the standard java sql interfaces as well.

You can read the closed issue and view community provided alternatives in [Issue #21](https://github.com/microsoft/sql-spark-connector/issues/21).

### Write to a new SQL Table

:warning: **Important: using the `overwrite` mode will first DROP the table if it already exists in the database by default. Please use this option with due care to avoid unexpected data loss!**


:warning: **When using mode `overwrite` if you do not use the option `truncate`, on recreation of the table indexes will be lost. For example a columnstore table would now be a heap. If you want to maintain existing indexing please also specify option `truncate` with value true. i.e `.option("truncate",true)`**


```python
server_name = "jdbc:sqlserver://{SERVER_ADDR}"
database_name = "database_name"
url = server_name + ";" + "databaseName=" + database_name + ";"

table_name = "table_name"
username = "username"
password = "password123!#" # Please specify password here

try:
  df.write \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .mode("overwrite") \
    .option("url", url) \
    .option("dbtable", table_name) \
    .option("user", username) \
    .option("password", password) \
    .save()
except ValueError as error :
    print("Connector write failed", error)
```

### Append to SQL Table
```python
try:
  df.write \
    .format("com.microsoft.sqlserver.jdbc.spark") \
    .mode("append") \
    .option("url", url) \
    .option("dbtable", table_name) \
    .option("user", username) \
    .option("password", password) \
    .save()
except ValueError as error :
    print("Connector write failed", error)
```

### Specifying the isolation level
This connector by default uses `READ_COMMITTED` isolation level when performing the bulk insert into the database. If you wish to override this to another isolation level, please use the `mssqlIsolationLevel` option as shown below.
```python
    .option("mssqlIsolationLevel", "READ_UNCOMMITTED") \
```

### Read from SQL Table
```python
jdbcDF = spark.read \
        .format("com.microsoft.sqlserver.jdbc.spark") \
        .option("url", url) \
        .option("dbtable", table_name) \
        .option("user", username) \
        .option("password", password).load()
```

### Azure Active Directory Authentication

#### Python Example with Service Principal
```python
context = adal.AuthenticationContext(authority)
token = context.acquire_token_with_client_credentials(resource_app_id_url, service_principal_id, service_principal_secret)
access_token = token["accessToken"]

jdbc_db = spark.read \
        .format("com.microsoft.sqlserver.jdbc.spark") \
        .option("url", url) \
        .option("dbtable", table_name) \
        .option("accessToken", access_token) \
        .option("encrypt", "true") \
        .option("hostNameInCertificate", "*.database.windows.net") \
        .load()
```

#### Python Example with Active Directory Password
```python
jdbc_df = spark.read \
        .format("com.microsoft.sqlserver.jdbc.spark") \
        .option("url", url) \
        .option("dbtable", table_name) \
        .option("authentication", "ActiveDirectoryPassword") \
        .option("user", user_name) \
        .option("password", password) \
        .option("encrypt", "true") \
        .option("hostNameInCertificate", "*.database.windows.net") \
        .load()
```

A required dependency must be installed in order to authenticate using
Active Directory.

For **Scala,** the `com.microsoft.aad.adal4j` artifact will need to be installed.

For **Python,** the `adal` library will need to be installed.  This is available
via pip.


Please check the [sample notebooks](samples) for examples.

# Support
The Apache Spark Connector for Azure SQL and SQL Server is an open source project. This connector does not come with any Microsoft support. For issues with or questions about the connector, please create an Issue in this project repository. The connector community is active and monitoring submissions.

# Roadmap
Visit the Connector project in the **Projects** tab to see needed / planned items. Feel free to make an issue and start contributing!

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
