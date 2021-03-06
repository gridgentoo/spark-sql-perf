# [CPU-нагрузки в Apache Spark] Тестирование [CERN]

https://docs.google.com/document/d/1ZOUElxEjKRccw6alcZu2LWon90XUTyywAOXdWj1gVvw/

Статья из ЦЕРН на английском

https://db-blog.web.cern.ch/blog/luca-canali/2017-09-performance-analysis-cpu-intensive-workload-apache-spark

# Hadoop and Spark services at CERN

Эксперименты по физике высоких энергий в ЦЕРНе производят и собирают больше данных, чем когда-либо прежде, недавно нарушив рекорд 12,3 ПБ в месяц. CERN предоставляет услуги Hadoop и Spark и тесно сотрудничает с научными сообществами в их стремлении проанализировать и понять эти огромные объемы данных физики высоких энергий и Океана Хиггса. В этом докладе будет представлен обзор текущей инфраструктуры на основе Spark и других ключевых компонентов экосистемы Hadoop. 

![Image alt](https://image.slidesharecdn.com/hadoopandsparkservicesatcernfinal-180427232551/95/hadoop-and-spark-services-at-cern-1-638.jpg)

Hadoop and Spark services at CERN

https://www.youtube.com/watch?v=Z7l3sV4dF4g

Hadoop and Spark services at CERN

https://www.slideshare.net/Hadoop_Summit/hadoop-and-spark-services-at-cern

![Image alt](https://i.ytimg.com/vi/3SiGujnfDVc/hqdefault.jpg)

Океана Хиггса : : Dark Matter vs Dark Energy - Leonard Susskind

https://www.youtube.com/watch?v=3SiGujnfDVc

![Image alt](https://pi.tedcdn.com/r/pe.tedcdn.com/images/ted/5efc8dbe4963321ede8d64c6398b94eb5a9797a0_800x600.jpg)

TEDxCaltech - Leonard Susskind - Richard Feynman

https://www.youtube.com/watch?v=hpjwotips7E

# Spark SQL Performance Tests

[![Build Status](https://travis-ci.org/databricks/spark-sql-perf.svg)](https://travis-ci.org/databricks/spark-sql-perf)

Это framework для тестирования производительности для [Spark SQL](https://spark.apache.org/sql/) in [Apache Spark](https://spark.apache.org/) 

This is a performance testing framework for [Spark SQL](https://spark.apache.org/sql/) in [Apache Spark](https://spark.apache.org/) 2.2+.

**Note: This README is still under development. Please also check our source code for more information.**

# Quick Start

## Running from command line.

```
$ bin/run --help

spark-sql-perf 0.2.0
Usage: spark-sql-perf [options]

  -b <value> | --benchmark <value>
        the name of the benchmark to run
  -f <value> | --filter <value>
        a filter on the name of the queries to run
  -i <value> | --iterations <value>
        the number of iterations to run
  --help
        prints this usage text
        
$ bin/run --benchmark DatasetPerformance
```

The first run of `bin/run` will build the library.

## Build

Use `sbt package` or `sbt assembly` to build the library jar.

# MLlib tests

To run MLlib tests, run `/bin/run-ml yamlfile`, where `yamlfile` is the path to a YAML configuration
file describing tests to run and their parameters.

# TPC-DS

## Setup a benchmark

Before running any query, a dataset needs to be setup by creating a `Benchmark` object. Generating
the TPCDS data requires dsdgen built and available on the machines. We have a fork of dsdgen that
you will need. The fork includes changes to generate TPCDS data to stdout, so that this library can
pipe them directly to Spark, without intermediate files. Therefore, this library will not work with
the vanilla TPCDS kit.

TPCDS kit needs to be installed on all cluster executor nodes under the same path!

It can be found [here](https://github.com/databricks/tpcds-kit).  

```
import com.databricks.spark.sql.perf.tpcds.TPCDSTables

// Set:
val rootDir = ... // root directory of location to create data in.
val databaseName = ... // name of database to create.
val scaleFactor = ... // scaleFactor defines the size of the dataset to generate (in GB).
val format = ... // valid spark format like parquet "parquet".
// Run:
val tables = new TPCDSTables(sqlContext,
    dsdgenDir = "/tmp/tpcds-kit/tools", // location of dsdgen
    scaleFactor = scaleFactor,
    useDoubleForDecimal = false, // true to replace DecimalType with DoubleType
    useStringForDate = false) // true to replace DateType with StringType


tables.genData(
    location = rootDir,
    format = format,
    overwrite = true, // overwrite the data that is already there
    partitionTables = true, // create the partitioned fact tables 
    clusterByPartitionColumns = true, // shuffle to get partitions coalesced into single files. 
    filterOutNullPartitionValues = false, // true to filter out the partition with NULL key value
    tableFilter = "", // "" means generate all tables
    numPartitions = 100) // how many dsdgen partitions to run - number of input tasks.

// Create the specified database
sql(s"create database $databaseName")
// Create metastore tables in a specified database for your data.
// Once tables are created, the current database will be switched to the specified database.
tables.createExternalTables(rootDir, "parquet", databaseName, overwrite = true, discoverPartitions = true)
// Or, if you want to create temporary tables
// tables.createTemporaryTables(location, format)

// For CBO only, gather statistics on all columns:
tables.analyzeTables(databaseName, analyzeColumns = true) 
```

## Run benchmarking queries
After setup, users can use `runExperiment` function to run benchmarking queries and record query execution time. Taking TPC-DS as an example, you can start an experiment by using

```
import com.databricks.spark.sql.perf.tpcds.TPCDS

val tpcds = new TPCDS (sqlContext = sqlContext)
// Set:
val databaseName = ... // name of database with TPCDS data.
val resultLocation = ... // place to write results
val iterations = 1 // how many iterations of queries to run.
val queries = tpcds.tpcds2_4Queries // queries to run.
val timeout = 24*60*60 // timeout, in seconds.
// Run:
sql(s"use $databaseName")
val experiment = tpcds.runExperiment(
  queries, 
  iterations = iterations,
  resultLocation = resultLocation,
  forkThread = true)
experiment.waitForFinish(timeout)
```

By default, experiment will be started in a background thread.
For every experiment run (i.e. every call of `runExperiment`), Spark SQL Perf will use the timestamp of the start time to identify this experiment. Performance results will be stored in the sub-dir named by the timestamp in the given `spark.sql.perf.results` (for example `/tmp/results/timestamp=1429213883272`). The performance results are stored in the JSON format.

## Retrieve results
While the experiment is running you can use `experiment.html` to get a summary, or `experiment.getCurrentResults` to get complete current results.
Once the experiment is complete, you can still access `experiment.getCurrentResults`, or you can load the results from disk.

```
// Get all experiments results.
val resultTable = spark.read.json(resultLocation)
resultTable.createOrReplaceTempView("sqlPerformance")
sqlContext.table("sqlPerformance")
// Get the result of a particular run by specifying the timestamp of that run.
sqlContext.table("sqlPerformance").filter("timestamp = 1429132621024")
// or
val specificResultTable = spark.read.json(experiment.resultPath)
```

You can get a basic summary by running:
```
experiment.getCurrentResults // or: spark.read.json(resultLocation).filter("timestamp = 1429132621024")
  .withColumn("Name", substring(col("name"), 2, 100))
  .withColumn("Runtime", (col("parsingTime") + col("analysisTime") + col("optimizationTime") + col("planningTime") + col("executionTime")) / 1000.0)
  .select('Name, 'Runtime)
```

## Running in Databricks.

There are example notebooks in `src/main/notebooks` for running TPCDS in the Databricks environment.

### tpcds_datagen notebook

This notebook can be used to install dsdgen on all worker nodes, run data generation, and create the TPCDS database.
Note that because of the way dsdgen is installed, it will not work on an autoscaling cluster, and `num_workers` has
to be updated to the number of worker instances on the cluster.
Data generation may also break if any of the workers is killed - the restarted worker container will not have `dsdgen` anymore.
 
### tpcds_run notebook

This notebook can be used to run TPCDS queries.

For running parallel TPCDS streams:
* Create a Cluster and attach the spark-sql-perf library to it.
* Create a Job using the notebook and attaching to the created cluster as "existing cluster".
* Allow concurrent runs of the created job.
* Launch appriopriate number of Runs of the Job to run in parallel on the cluster.
