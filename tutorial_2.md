---
title: Aerospike Connect for Spark Tutorial
description: Learn how to deploy and configure Aerospike Connect for Spark.
---

This example does the following:

- Creates a data set of Australian Prime Ministers and saves it to Aerospike via Spark.
- Retrieves a subset of the data using the `aeroJoin` operation.
- Filters the data by party affiliation to get only Labor Party members.
- Writes the results to a new Aerospike set.

**Contents**

- [Initialize example object and SparkConf](#initialize-example-object-and-sparkconf)
- [Create a table of Australian PMs ](#create-a-table-of-australian-pms-)
- [Retrieve subset of data using the aeroJoin operation](#retrieve-subset-of-data-using-the-aerojoin-operation)
    - [Get results as an AeroKV extension](#get-results-as-an-aerokv-extension)
    - [Return only data via the bespoke method](#return-only-data-via-the-bespoke-method)
    - [Filter only Labor PMs ](#filter-only-labor-pms-)
- [Save the results](#save-the-results)
- [Verify the results](#verify-the-results)
- [Complete example code](#complete-example-code)

### Initialize example object and SparkConf

At the top we initialize the program's object `AerojoinExample` and set up the `SparkConf`:

```scala
object aeroJoinExample extends LazyLogging with Serializable {
  val seedHost = "172.17.0.4"
  val exampleNamespace = "device"

  // Aerospike configuration for spark connector
  val conf: SparkConf = new SparkConf()
    .set("aerospike.seedhost", seedHost)
    .set("aerospike.port",  "3000")
    .set("aerospike.namespace", exampleNamespace)
    ...
```


### Create a table of Australian PMs 

We will save the data to Aerospike using `prime_min_id` as the primary key.

| prime_min_id     | last_name | first_name | party     | years_in_office |
| ---------------- | --------- | ---------- | --------- | --------------- |
| `Fraser_Malcolm` | `Fraser`  | `Malcolm`  | `Liberal` | `7`             |
| `Hawke_Bob`      | `Hawke`   | `Bob`      | `Labor`   | `9`             |
| `Keating_Paul`   | Keating   | `Paul`     | `Labor`   | `4`             |
| `Howard_John`    | Howard    | `John`     | `Liberal` | `12`            |

```
...
val rows = Seq(
      Row("Fraser_Malcolm", "Fraser", "Malcolm", "Liberal",7),
      Row("Hawke_Bob", "Hawke", "Bob", "Labor",9),
      Row("Keating_Paul", "Keating", "Paul", "Labor",4),
      Row("Howard_John", "Howard", "John", "Liberal", 12)
    )
...
```

### Retrieve subset of data using the aeroJoin operation

We retrieve the data by way of the following primary keys. Note we omit Malcom Fraser and add Kevin Rudd. We should therefore get three results; see **[Return only data via the bespoke method](#return-only-data-via-the-bespoke-method)**.

```scala
...
   val ausPrimeMinIDs = Seq( "Hawke_Bob", "Keating_Paul","Howard_John","Rudd_Kevin")
...
    val ausPrimeMinIDsDS = ausPrimeMinIDs.toDF("prime_min_id").as[AusPrimeMinID]
...
    val aussiePMDS = ausPrimeMinIDsDS.aeroJoin[AusPrimeMinKV]("prime_min_id", "AussiePMs")
...
```

**Result**

```
Hawke_Bob,Keating_Paul,Howard_John,Rudd_Kevin
```

#### Get results as an AeroKV extension

Note the metadata `namespace`,`setName`,`digest`, and `userKey`:

```scala
...
AusPrimeMinKV(Map(namespace -> device, setName -> AussiePMs, digest -> [B@884b46e, userKey -> Map(object -> Hawke_Bob, type -> 3)),Hawke_Bob,Bob,Hawke,Labor,9)
AusPrimeMinKV(Map(namespace -> device, setName -> AussiePMs, digest -> [B@b869470, userKey -> Map(object -> Keating_Paul, type -> 3)),Keating_Paul,Paul,Keating,Labor,4)
AusPrimeMinKV(Map(namespace -> device, setName -> AussiePMs, digest -> [B@7fefc375,v userKey -> Map(object -> Howard_John, type -> 3)),Howard_John,John,Howard,Liberal,12)
...
```

#### Return only data via the bespoke method

```
ID: Keating_Paul;First Name: Paul;Last name: Keating;Party: Labor;Years in office: 4
ID: Howard_John;First Name: John;Last name: Howard;Party: Liberal;Years in office: 12
ID: Hawke_Bob;First Name: Bob;Last name: Hawke;Party: Labor;Years in office: 9
```

#### Filter only Labor PMs 

The following lines in the sample filter the data to extract only Labor party members:

```scala
...
val laborPMs = aussiePMDS.filter(pm => pm.party == "Labor")
...
```

**Results**

```
ID: Keating_Paul;First Name: Paul;Last name: Keating;Party: Labor;Years in office: 4
ID: Hawke_Bob;First Name: Bob;Last name: Hawke;Party: Labor;Years in office: 9
```

### Save the results

Finally we save the results to a specific set named `laborPMs`.

```scala
// We need to map to a type free of AeroKV metadata to remove metadata fields
    laborPMs.map(pm => new AusPrimeMin(pm.prime_min_id, pm.first_name, pm.last_name, pm.party, pm.years_in_office)).
      toDF("prime_min_id", "last_name", "first_name", "party","years_in_office").
      write.mode(SaveMode.Overwrite).
      format("com.aerospike.spark.sql").
      option("aerospike.updateByKey", "prime_min_id").
      option("aerospike.set", "laborPMs").
      save()
```

### Verify the results

To see the results, use the following statement in `aql`:

```
aql> select * from device.laborPMs
      +------------+-----------+---------+----------------+-----------------+
      | first_name | last_name | party   | prime_min_id   | years_in_office |
      +------------+-----------+---------+----------------+-----------------+
      | "Paul"     | "Keating" | "Labor" | "Keating_Paul" | 4               |
      | "Bob"      | "Hawke"   | "Labor" | "Hawke_Bob"    | 9               |
      +------------+-----------+---------+----------------+-----------------+
      2 rows in set (0.179 secs)
```

### Complete example code

```scala
package com.aerospike.spark

import com.typesafe.scalalogging.slf4j.LazyLogging
import org.apache.spark.SparkConf
import org.apache.spark.sql.types.{IntegerType, StringType, StructField, StructType}
import org.apache.spark.sql.{Row, SaveMode, SparkSession}
import org.apache.log4j.{Level, Logger}

/**
  * This example shows use of aeroJoin and filtering
  * Prereqs: Aerospike Connect for Spark and Aerospike Server
  *
  * Set seedHost below to an IP address within your Aerospike cluster
  * Set exampleNamespace to a configured namespace within your Aerospike cluster
  *
  */

object aeroJoinExample extends LazyLogging with Serializable {
  val seedHost = "172.17.0.4"
  val exampleNamespace = "device"

  // Aerospike configuration for spark connector
  val conf: SparkConf = new SparkConf()
    .set("aerospike.seedhost", seedHost)
    .set("aerospike.port",  "3000")
    .set("aerospike.namespace", exampleNamespace)

  // Programatically created Spark session - for the duration of this program only
  val session: SparkSession = SparkSession.builder()
    .config(conf)
    .master("local[*]")
    .appName("Aerospike Example")
    .config("spark.ui.enabled", "false")
    .getOrCreate()

  // Schema used to create Data Frame from Sequence data (below)
  val schema: StructType = new StructType(Array(
    StructField("prime_min_id", StringType, nullable = false),
    StructField("first_name", StringType, nullable = true),
    StructField("last_name", StringType, nullable = true),
    StructField("party",StringType,nullable = true),
    StructField("years_in_office", IntegerType, nullable = true)
  ))


  def main(args: Array[String]) {
    // Remove Spark info messages
    val rootLogger = Logger.getRootLogger()
    rootLogger.setLevel(Level.ERROR)

    import session.implicits._ // Required to convert data to Data Frames

    println("In this example we create a data set (Australian Prime Ministers) and save to Aerospike via Spark")
    println("Next we retrieve a subset of this data using the aeroJoin operation")
    println("This data is then filtered by party affiliation")
    println("and finally written back to a new Aerospike set\n")
    /*
      Create a DataFrame (Spark 'Table') of Australian PM data
      and save it to Aerospike
    */
    println("Creating a table of Australian PM data which we will save into Aerospike using prime_min_id as the primary key\n")

    // Australian PM daata
    // Cols are IDs,Last Name, First Name, Years in Office (to nearest year)
    // Data from https://en.wikipedia.org/wiki/List_of_prime_ministers_of_Australia
    val rows = Seq(
      Row("Fraser_Malcolm", "Fraser", "Malcolm", "Liberal",7),
      Row("Hawke_Bob", "Hawke", "Bob", "Labor",9),
      Row("Keating_Paul", "Keating", "Paul", "Labor",4),
      Row("Howard_John", "Howard", "John", "Liberal", 12)
    )

    // To create a data frame ( against which we can issue SQL like queries )
    // we first create an RDD ( resilient distributed dataset - basic Spark building block )
    // and then convert it into a data frame (
    val primeMinRDD = session.sparkContext.parallelize(rows)
    val primeMinDF = session.sqlContext.createDataFrame(primeMinRDD, schema)

    primeMinDF.show()

    // Save the data to Aerospike
    // Using prime_min_id as the key
    primeMinDF.write.
      // Save mode choices are ErrorIfExists, Append, Ignore, Overwrite
      // See https://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html for details
      mode(SaveMode.Overwrite).
      // Use of 'com.aerospike.spark.sql' indicates the write destination as being an Aerospike database
      format("com.aerospike.spark.sql").
      // Aerospike specific options
      // See https://www.aerospike.com/docs/connectors/enterprise/spark/reference.html for full list
      // Update by key - we use a column in our data frame as a key and identify which column to use
      option("aerospike.updateByKey", "prime_min_id").
      // Identify the set to save the data into
      option("aerospike.set", "AussiePMs").
      save()

    println("Retrieve data using the aeroJoin operation")

    val ausPrimeMinIDs = Seq( "Hawke_Bob", "Keating_Paul","Howard_John","Rudd_Kevin")

    println("using the following primary keys : "+ausPrimeMinIDs.mkString(","))
    println("\nNote we omit Malcom Fraser, and add in Kevin Rudd. We should therefore get three results\n")

    // Method is, create typed data set - in our case AusPrimeMinID which contains our required keys
    val ausPrimeMinIDsDS = ausPrimeMinIDs.toDF("prime_min_id").as[AusPrimeMinID]

    // Now do the join from our list of keys to our Aerospike set (note the namespace is already specified)
    // We specify the field in our set (ausPrimeMinIDsDS) that contains the required keys
    // We must also specify the return type, which must inherit from AeroKV
    val aussiePMDS = ausPrimeMinIDsDS.aeroJoin[AusPrimeMinKV]("prime_min_id", "AussiePMs")

    // Object returned is of type AusPrimeMinKV
    println("First the results as an AeroKV extension - note the metadata namespace,setName,digest,userKey\n")
    aussiePMDS.foreach(b => {println(b)})

    println("\nSecond :  returned data via a bespoke method showing data only\n")
    aussiePMDS.foreach(b => {println(b.dataToString)})

    println("\nUse of filters - show only Labor PMs from the list above\n")
    val laborPMs = aussiePMDS.filter(pm => pm.party == "Labor")
    laborPMs.foreach(b => println(b.dataToString))

    println("\nFinally we save the results to a specific set - laborPMs")

    // We need to map to a type free of AeroKV metadata to remove metadata fields
    laborPMs.map(pm => new AusPrimeMin(pm.prime_min_id, pm.first_name, pm.last_name, pm.party, pm.years_in_office)).
      toDF("prime_min_id", "last_name", "first_name", "party","years_in_office").
      write.mode(SaveMode.Overwrite).
      format("com.aerospike.spark.sql").
      option("aerospike.updateByKey", "prime_min_id").
      option("aerospike.set", "laborPMs").
      save()

    println("In aql do select * from "+exampleNamespace+".laborPMs to see results")

    /*
      Result will be

      aql> select * from device.laborPMs
      +------------+-----------+---------+----------------+-----------------+
      | first_name | last_name | party   | prime_min_id   | years_in_office |
      +------------+-----------+---------+----------------+-----------------+
      | "Paul"     | "Keating" | "Labor" | "Keating_Paul" | 4               |
      | "Bob"      | "Hawke"   | "Labor" | "Hawke_Bob"    | 9               |
      +------------+-----------+---------+----------------+-----------------+
      2 rows in set (0.179 secs)

    */
  }
}

/* Case classes allow easy creation of struct like classes */
case class AusPrimeMinID(prime_min_id: String)

case class AusPrimeMin(prime_min_id: String, first_name: String, last_name: String, party: String, years_in_office: Long)

/* AusPrimeMinKV is used to retrieve data from Aerospike, so must extend AeroKV
   We add in a ;'dataToString' method as the default toString method exposes object metadata
   AeroKV requires specification of a __key field */
case class AusPrimeMinKV(__key: Any, prime_min_id: String, first_name: String, last_name: String, party:String, years_in_office: Long) extends AeroKV{
  def dataToString : String ={
    "ID: "+prime_min_id + ";First Name: "+first_name+ ";Last name: "+last_name + ";Party: "+party+";Years in office: "+years_in_office
  }
}
```
