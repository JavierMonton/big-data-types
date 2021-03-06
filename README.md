# Big Data Types
![CI Tests](https://github.com/data-tools/big-data-types/workflows/ci-tests/badge.svg)
[![codecov](https://codecov.io/gh/data-tools/big-data-types/branch/main/graph/badge.svg)](https://codecov.io/gh/data-tools/big-data-types)
![Maven Central](https://img.shields.io/maven-central/v/io.github.data-tools/big-data-types-core_2.13)
![Scala 2.12](https://img.shields.io/badge/Scala-2.12-red)
![Scala_2.13](https://img.shields.io/badge/Scala-2.13-red)

A library to transform Case Classes into Database schemas

This is a type safe library that converts basic Scala types and Case Classes into different database types and schemas using Shapeless, 
making possible to extract a database schema from a Case Class and to work with Case Classes when writing,
reading or creating tables in different databases. 

For now, it supports BigQuery and Spark.


- [Big Data Types](#big-data-types)
- [Quick Start](#quick-start)
- [BigQuery](#bigquery)
  * [Create BigQuery Tables](#create-bigquery-tables)
    + [Transform field names](#transform-field-names)
    + [Time Partitioned tables](#time-partitioned-tables)
    + [Create a table with more than one Case Class](#create-a-table-with-more-than-one-case-class)
  * [Create BigQuery schema from a Case Class](#create-bigquery-schema-from-a-case-class)
  * [From a Case Class instance](#from-a-case-class-instance)
  * [Connecting to your BigQuery environment](#connecting-to-your-bigquery-environment)
- [Spark](#spark)
  * [Spark Schema from Case Class](#spark-schema-from-case-class)
    + [Spark Schema from Multiple Case Classes](#spark-schema-from-multiple-case-classes)
  * [Field transformations](#field-transformations)
- [Implicit Formats](#implicit-formats)
  * [DefaultFormats](#defaultformats)
  * [SnakifyFormats](#snakifyformats)
  * [Creating a custom Formats](#creating-a-custom-formats)

# Quick Start
The library has different modules that can be imported separately
- BigQuery
```
libraryDependencies += "io.github.data-tools" % "big-data-types-bigquery_2.13" % "{version}"
```
- Spark
```
libraryDependencies += "io.github.data-tools" % "big-data-types-spark_2.12" % "{version}"
```
- Core
    - To get support for abstract SqlTypes, it is included in the others, so it is not needed if you are using one of the others
```
libraryDependencies += "io.github.data-tools" % "big-data-types-core_2.13" % "{version}"
```

Versions for Scala ![Scala 2.12](https://img.shields.io/badge/Scala-2.12-red)
 and ![Scala_2.13](https://img.shields.io/badge/Scala-2.13-red) are available in Maven


 
# BigQuery

## Create BigQuery Tables

```scala
import org.datatools.bigdatatypes.bigquery.BigQueryTable
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats

case class MyTable(field1: Int, field2: String)
BigQueryTable.createTable[MyTable]("dataset_name", "table_name")
```
This also works with Structs, Lists and Options.
See more examples in [Tests](https://github.com/data-tools/big-data-types/blob/main/src/it/scala/org/datatools/bigdatatypes/bigquery/BigQueryTableSpec.scala)

### Transform field names
There is a `Format` object that allows us to decide how to transform field names, for example, changing CamelCase for snake case
```scala
import org.datatools.bigdatatypes.bigquery.BigQueryTable
import org.datatools.bigdatatypes.formats.Formats.implicitSnakifyFormats

case class MyTable(myIntField: Int, myStringField: String)
BigQueryTable.createTable[MyTable]("dataset_name", "table_name")
//This table will have my_int_field and my_string_field fields
```

### Time Partitioned tables
Using a `Timestamp` or `Date` field, tables can be partitioned in BigQuery using a [Time Partition Column](https://cloud.google.com/bigquery/docs/creating-column-partitions)
```scala
import org.datatools.bigdatatypes.bigquery.BigQueryTable
import org.datatools.bigdatatypes.formats.Formats.implicitSnakifyFormats

case class MyTable(field1: Int, field2: String, myPartitionField: java.sql.Timestamp)
BigQueryTable.createTable[MyTable]("dataset_name", "table_name", "my_partition_field")
```
### Create a table with more than one Case Class
In many cases we work with a Case Class that represents our data but we also want to add 
some metadata fields like `updated_at`, `received_at`, `version` and so on.
In these cases we can work with multiple Case Classes and fields will be concatenated:

```scala
import org.datatools.bigdatatypes.bigquery.BigQueryTable
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats

case class MyData(field1: Int, field2: String)
case class MyMetadata(updatedAt: Long, version: Int)
BigQueryTable.createTable[MyData, MyMetadata]("dataset_name", "table_name")
```
This can be done up to 5 concatenated classes


## Create BigQuery schema from a Case Class
```scala
import com.google.cloud.bigquery.{Field, Schema}
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats
import org.datatools.bigdatatypes.bigquery.BigQueryTypes

case class MyTable(field1: Int, field2: String)
//List of BigQuery Fields, it can be used to construct an Schema
val fields: List[Field] = BigQueryTypes[MyTable].bigQueryFields
//BigQuery Schema, it can be used to create a table
val schema: Schema = Schema.of(fields.asJava)
```

## From a Case Class instance
```scala
import com.google.cloud.bigquery.Field
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats
import org.datatools.bigdatatypes.bigquery.BigQueryTypes._

case class MyTable(field1: Int, field2: String)
val data = MyTable(1, "test")
val fields: List[Field] = data.getBigQueryFields
```

See more info about [creating tables on BigQuery](https://cloud.google.com/bigquery/docs/tables#java) in the official documentation

## Connecting to your BigQuery environment
If you want to create tables using the library you will need to connect to your BigQuery environment 
through any of the GCloud options. 
Probably the most common will be to specify a service account and a project id.
It can be added on environment variables. The library expects:
- PROJECT_ID: <your_project_id>
- GOOGLE_APPLICATION_CREDENTIAL: <path_to_your_service_account_json_file>

---

# Spark

## Spark Schema from Case Class

With Spark module, Spark Schemas can be created from Case Classes.
```scala
import org.apache.spark.sql.types.StructType
import org.datatools.bigdatatypes.spark.SparkSchemas
//an implicit Formats class is needed, defaultFormats does no transformations
//it can be created as implicit val instead of using this import
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats

case class MyModel(myInt: Integer, myString: String)
val schema: StructType = SparkSchemas.schema[MyModel]
```
It works for Options, Sequences and any level of nested objects

Also, a Spark Schema can be extracted from a Case Class instance
```scala
val model = MyModel(1, "test")
model.sparkSchema
```

### Spark Schema from Multiple Case Classes
Also, an schema can be created from multiple case classes. 
As an example, it could be useful for those cases where we read data using a Case Class, 
and we want to append some metadata fields, but we don't want to create another Case Class with exactly the same fields plus a few more.
```scala
import java.sql.Timestamp
import org.apache.spark.sql.types.StructType
import org.datatools.bigdatatypes.spark.SparkSchemas
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats
 
case class MyModel(myInt: Integer, myString: String)
case class MyMetadata(updatedAt: Timestamp, version: Int)
val schema: StructType = SparkSchemas.schema[MyModel, MyMetadata]
/*
schema =
 List(
    StructField(myInt, IntegerType, false), 
    StructField(myString, StringType, false)
    StructField(updatedAt, TimestampType, false)
    StructField(version, IntegerType, false)
   )
*/
```


## Field transformations
Also, custom transformations can be applied to field names, something that usually is quite hard to do with Spark Datasets.
For example, working with CamelCase Case Classes but using snake_case field names in Spark Schema.

```scala
import org.apache.spark.sql.types.StructType
import org.datatools.bigdatatypes.spark.SparkSchemas
//implicit formats for transform keys to snake_case
import org.datatools.bigdatatypes.formats.Formats.implicitSnakifyFormats

case class MyModel(myInt: Integer, myString: String)
val schema: StructType = SparkSchemas.schema[MyModel]
/*
schema =
 List(
    StructField(my_int, IntegerType, false), 
    StructField(my_string, StringType, false)
   )
*/
```

---

# Implicit Formats
Formats can handle different configurations that we want to apply to schemas, like transforming field names, 
defining precision for numeric types and so on.

For now, it only contains a possibility for field names transformations.

They can be used by creating an implicit val with a Formats class or by importing one of the available implicit vals in `Formats` object 

## DefaultFormats
`DefaultFormats` is a trait that applies no transformation to field names
To use it, you can create an implicit val:
```scala
import org.datatools.bigdatatypes.formats.{Formats, DefaultFormats}
implicit val formats: Formats = DefaultFormats
```
or just import the one available:
```scala
import org.datatools.bigdatatypes.formats.Formats.implicitDefaultFormats
```


## SnakifyFormats
`SnakifyFormats` is a trait that converts camelCase field names to snake_case names
To use it, you can create an implicit val:
```scala
import org.datatools.bigdatatypes.formats.{Formats, SnakifyFormats}
implicit val formats: Formats = SnakifyFormats
```
or just import the one available:
```scala
import org.datatools.bigdatatypes.formats.Formats.implicitSnakifyFormats
```

## Creating a custom Formats
Formats can be extended, so if we want to transform keys differently, for example adding a suffix to all of our fields
```scala
import org.datatools.bigdatatypes.formats.Formats
trait SuffixFormats extends Formats {
  override def transformKeys(key: String): String = key + "_at"
}
object SuffixFormats extends SuffixFormats
```
All your field names will have "_at" at the end
