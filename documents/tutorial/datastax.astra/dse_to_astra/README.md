<span style="font-size: 32px">**Table of Contents**</span>

- [1. Overview](#1-overview)
- [2. Environment Setup](#2-environment-setup)
  - [2.1. Set up Azure Resources](#21-set-up-azure-resources)
  - [2.2. Set up and Prepare DSE/C* Cluster](#22-set-up-and-prepare-dsec-cluster)
  - [2.3. Set up and Prepare an Astra Database](#23-set-up-and-prepare-an-astra-database)
  - [2.4. Set up and Prepare Databricks Spark Cluster](#24-set-up-and-prepare-databricks-spark-cluster)
- [3. Migrate Data between two C* based Clusters using SCC](#3-migrate-data-between-two-c-based-clusters-using-scc)
  - [3.1. Spark 3.0 + SCC 3.0: Cassandra Catalog](#31-spark-30--scc-30-cassandra-catalog)
  - [3.2. Spark 2.4 + SCC 2.5: Multiple SCC connector](#32-spark-24--scc-25-multiple-scc-connector)


# 1. Overview

In my previous [tutorial](https://github.com/yabinmeng/dseutilities/tree/master/documents/tutorial/datastax.astra/databricks_conn), I demonstrated how to use Databricks Spark platform (e.g. Azure Databricks service) to load data from an external source (a CSV file) into a DataStax Astra database. 

In this tutorial, I will demonstrate how to use Databricks Spark to load data from a stand-alone Cassandra (C*) cluster into a DataStax Atra database.

# 2. Environment Setup

For testing purpose in this repo., we need a stand-alone C* cluster and a Databricks cluster and most importantly we need to make sure these two clusters can communicate with each other. In order to achieve this task, I'm taking the following approach:

* Launch a Databricks service on Azure using Azure Databricks service
* Launch several Azure virtual machines and install a DSE cluster on it
* Make sure the launched Azure Databricks service and and the DSE virtual machines **belong to the same Azure virtual network**.

## 2.1. Set up Azure Resources

The detailed procedure of setting up Azure resources is beyond the scope of this tutorial. Please refer to Azure   for it. In this repo., I will only list the Azure resources and relevant key characteristics that are needed by our testing in this repo. 

In the testing, all Azure resources are created in the same Azure region: **Central US**

1. Create an Azure resource group (RG), E.g. MyAstraDtbrksRG

2. Create an Azure virtual network (VN). When creating the VN, make sure to explicitly specify the following key properties:
   
   * Resource group: the RG that was created earlier (e.g. MyAstraDtbrksRG)
   * Name and CIDR range for VN address space, e.g. 
     * Name: MyAstraDtbrksVN
     * CIDR: 10.4.0.0/16
   * Name and CIDR range for one Subnet(SN), e.g.
     * Name: MyAstraDtbrksVN-Workload-SN
     * CIDR: 10.4.10.0/24

3. Create several Azure virtual machines (VM). When creating the VMs, make sure to explicitly specify the following key properties:

   * Resource group: the RG that was created earlier (e.g. MyAstraDtbrksRG)
   * Size: Azure instance type (e.g. Standard_D4s_v3, 16 Gib memory)
   * Virtual Network: the VN that was created earlier (e.g. MyAstraDtbrksVN)
   * Subnet: the SN that was created earlier (e.g. MyAstraDtbrksVN-Workload-SN)
  
Also create and share a common SSH key pair among all the created VMs. All other properties can be left as default

4. Create an Azure Databricks Workspace (ADW), make sure to explicitly specify the following key properties:

   * Resource group: the RG that was created earlier (e.g. MyAstraDtbrksRG)
   * Pricing Tier: (e.g. Standard)
   * Deploy Azure Databricks workspace in your own Virtual Network (VNet): Yes
     * Virtual Network: the VN that was created earlier (e.g. MyAstraDtbrksVN)
     * Public Subnet Name and CIDR. Make sure the CIDR range is aligned with the VN CIDR range that was specified earlier and not overlapping with the existing subnet for VMs, e.g.
       * Name: MyAzureDbrksPubSN
       * CIDR: 10.4.255.0/24
    * Public Subnet Name and CIDR. Make sure the CIDR range is aligned with the VN CIDR range that was specified earlier, e.g.
       * Name: MyAzureDbrksPrvSN
       * CIDR: 10.4.20.0/24 

## 2.2. Set up and Prepare DSE/C* Cluster

On each of the launched VMs, do the following tasks:

* Install OpenJDK 8 
* Install latest DSE 6.8 release (6.8.5) binary ([procedure](https://docs.datastax.com/en/install/6.8/install/installDEBdse.html))
* Make necessary changes in cassandra.yaml to form one DSE/C* cluster
* Create a C* keyspace (**testks**) and a table (**testbl_dse**) for testing purpose and insert some data in the table

```
cqlsh> desc table testks.testbl_dse ;

CREATE TABLE testks.testbl_dse (
    cola int PRIMARY KEY,
    colb text
)
... ...

cqlsh> select * from testks.testbl_dse ;

 cola | colb
------+-------------
  100 | azure-row-100
  200 | azure-row-200
  300 | azure-row-300

(3 rows)
```

## 2.3. Set up and Prepare an Astra Database

In this repo, I will use the same Astra database as in my [previous repo.](https://github.com/yabinmeng/dseutilities/tree/master/documents/tutorial/datastax.astra/databricks_conn).

Create a table (**testks.testbl_astra**) that has similar table structure as the one created above in the DSE cluster.

```
cqlsh> desc table testks.testbl_astra ;

CREATE TABLE testks.testbl_astra (
    cola int PRIMARY KEY,
    colb text
)
... ...

cqlsh> select * from testks.testbl_astra ;

 cola | colb
------+-------------
    0 | astra-row-0
    1 | astra-row-1
    2 | astra-row-2

(3 rows)
```


## 2.4. Set up and Prepare Databricks Spark Cluster

In the Azure Databricks workspace, create a Spark cluster with the following properties:

* Cluster Name: the name of the created cluster, e.g. MyDbrksSparkCluster
* Cluster Mode: Standard or High Concurrency
* Databricks Runtimes: 7.3 LTS (Scala 2.12, Spark 3.0.1)

Once the Spark cluster is created, 

* Install Spark Cassandra Connector (SCC) version 3.0 library (see [procedure](https://github.com/yabinmeng/dseutilities/tree/master/documents/tutorial/datastax.astra/databricks_conn#233-install-scc-as-databricks-cluster-library))

* Upload DataStax Astra secure connect bundle file (see [procedure](https://github.com/yabinmeng/dseutilities/tree/master/documents/tutorial/datastax.astra/databricks_conn#24-upload-data-into-databricks-cluster))

* Add the following cluster level Spark configuration (see [procedure](https://github.com/yabinmeng/dseutilities/tree/master/documents/tutorial/datastax.astra/databricks_conn#25-update-databricks-cluster-spark-configuration))

```
spark.files dbfs:/FileStore/tables/<secure_connect_bundle>.zip
spark.dse.continuousPagingEnabled false
```

**NOTE** that compared with my previous tutorial, I do NOT set the following Astra connection related Spark configuration items at **cluster** level. Instead, they're set at **catalog** level dynamically in the code. I will cover this with more details in the later sections.

```
spark.cassandra.connection.config.cloud.path <secure_connect_bundle>.zip
spark.cassandra.auth.username <astra_username>
spark.cassandra.auth.password <astra_passwd>
```

# 3. Migrate Data between two C* based Clusters using SCC

For most time, people use SCC for data migration/ETL related work between a C* cluster and another non-C* system (e.g. an RDBMS, another NoSQL database, etc.). But SCC can also connect to multiple C* clusters and therefore makes possible data migration between 2 C* based clusters. In our testing in this repo, I will use Databricks cluster + SCC to migrate data from a DSE cluster to a Astra database.

## 3.1. Spark 3.0 + SCC 3.0: Cassandra Catalog

Spark 3.0 adds support for Catalog Plugin API [SPARK-31121](https://issues.apache.org/jira/browse/SPARK-31121) which is an umbrella ticket that includes many improvements related Apache Spark DataSource V2 API.

Based on the improved functionalities/features of Spark DataSource V2 API, SCC 3.0 introduces the concept of Cassandra Catalog. Cassandra Catalog brings many advantages and greatly simplifies many tasks regarding Spark Cassandra connection. For example, using Cassandra Catalog, it is easy to connect to multiple Cassandra clusters in one Spark session, as demonstrated below:

<img src="https://github.com/yabinmeng/dseutilities/blob/master/documents/tutorial/datastax.astra/dse_to_astra/resources/screenshots/cassandra.catalog.png" width=400>

For more detailed introduction of **Cassandra Catalog**, please refer to Russell Spitzer's 2020 Spark Summit [presentation](https://databricks.com/session_na20/datasource-v2-and-cassandra-a-whole-new-world) (the above screenshot is also taken from his presentation).

With Cassandra Catalog, the code to migrate from the DSE cluster to the Astra database is straightforward. **Note** the code below can be executed directly in a Databricks notebook.

```
import com.datastax.spark.connector._
import com.datastax.spark.connector.cql._

//--------------------------
// Catalog to the source DSE cluster
val dseClusterAlias = "DseCluster"
val dseCatName = "spark.sql.catalog." + dseClusterAlias
val dseSrvIp = "<dse_srv_ip>"
val dseSrvPort = "9042"

spark.conf.set(dseCatName, "com.datastax.spark.connector.datasource.CassandraCatalog")
spark.conf.set(dseCatName + ".spark.cassandra.connection.host", dseSrvIp)
spark.conf.set(dseCatName + ".spark.cassandra.connection.port", dseSrvPort)

// -- read from DSE 
val tblDf_d = spark.read.table(dseClusterAlias + ".testks.testbl_dse")
println(">> [Step 1] Read from DSE: testks.testbl_dse")
tblDf_d.show()

//--------------------------
// Catalog  to the target Astra cluster
val astraClusterAlias = "AstraCluster"
val astraCatName = "spark.sql.catalog." + astraClusterAlias
val astraSecureConnFilePath = "dbfs:/FileStore/tables/secure_connect_myastradb.zip"
val astraSecureConnFileName = astraSecureConnFilePath.split("/").last
val astraUserName = "<astra_username>"
val astraPassword = "<astra_password>"

spark.conf.set(astraCatName, "com.datastax.spark.connector.datasource.CassandraCatalog")
spark.conf.set(astraCatName + ".spark.cassandra.connection.config.cloud.path", astraSecureConnFileName)
spark.conf.set(astraCatName + ".spark.cassandra.auth.username", astraUserName)
spark.conf.set(astraCatName + ".spark.cassandra.auth.password", astraPassword)

// -- read from Astra 
val tblDf_a = spark.read.table(astraClusterAlias + ".testks.testbl_astra")
println(">> [Step 2] Read from Astra: testks.testbl_astra")
tblDf_a.show()

//--------------------------
// -- Write to Astra
println(">> [Step 3] Write DSE data into Astra: testks.testbl_astra")
println
tblDf_d.writeTo(astraClusterAlias + ".testks.testbl_astra").append

//--------------------------
// -- Read from Astra again
println(">> [Step 4] Read again from Astra: testks.testbl_astra")
tblDf_a.show()
```

The result output is as below:

```
>> [Step 1] Read from DSE: testks.testbl_dse
+----+-----------+
|cola|       colb|
+----+-----------+
| 100|dse-row-100|
| 300|dse-row-300|
| 200|dse-row-200|
+----+-----------+

>> [Step 2] Read from Astra: testks.testbl_astra
+----+-----------+
|cola|       colb|
+----+-----------+
|   1|astra-row-1|
|   0|astra-row-0|
|   2|astra-row-2|
+----+-----------+

>> [Step 3] Write DSE data into Astra: testks.testbl_astra

>> [Step 4] Read again from Astra: testks.testbl_astra
+----+-----------+
|cola|       colb|
+----+-----------+
|   1|astra-row-1|
|   0|astra-row-0|
|   2|astra-row-2|
| 200|dse-row-200|
| 100|dse-row-100|
| 300|dse-row-300|
+----+-----------+
```

Looking at the program output, we can clearly see that the data is successfully migrated from the DSE cluster and landed in Astra.

## 3.2. Spark 2.4 + SCC 2.5: Multiple SCC connector 

It has to be pointed out that before **Cassandra Catalog**, SCC is already able to connect to multiple C* clusters. 

Create a Databricks cluster based on Spark 2.4. The runtime version is 
* 6.6 (Apache 2.4.5, Scala 2.11)

Download the corresponding SCC 2.5.1 assembly jar file (spark-cassandra-connector-assembly_2.11-2.5.1.jar) from [here](https://mvnrepository.com/artifact/com.datastax.spark/spark-cassandra-connector-assembly_2.11/2.5.1)

Without **Cassandra Catalog**, the code is a little bit different. The main difference is doing read/write by specifying "cluster" option, as below:

```
... ...

// -- read from DSE cluster
sqlContext.setConf(dseClusterAlias + "/spark.cassandra.connection.host", dseSrvIp)
sqlContext.setConf(dseClusterAlias + "/spark.cassandra.connection.port", dseSrvPort)

val tblDf_d = sqlContext
  .read
  .format("org.apache.spark.sql.cassandra")
  .options(Map( 
    "cluster" -> dseClusterAlias,
    "keyspace" -> "<astra_username>",
    "table" -> "<astra_password>"
    ))
  .load

... ... 

// -- read from Astra 
sqlContext.setConf(astraClusterAlias + "/spark.cassandra.connection.config.cloud.path", astraSecureConnFileName)
sqlContext.setConf(astraClusterAlias + "/spark.cassandra.auth.username", astraUserName)
sqlContext.setConf(astraClusterAlias + "/spark.cassandra.auth.password", astraPassword)
val tblDf_a = sqlContext
  .read
  .format("org.apache.spark.sql.cassandra")
  .options(Map( 
    "cluster" -> astraClusterAlias,
    "keyspace" -> "testks",
    "table" -> "testbl_astra"
    ))
  .load

.... 

// -- Write to Astra
tblDf_d.write
  .format("org.apache.spark.sql.cassandra")
  .options(Map( 
    "cluster" -> astraClusterAlias,
    "keyspace" -> "testks",
    "table" -> "testbl_astra"
    ))
  .mode("append")  
  .save

... ... 
```