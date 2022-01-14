# alluxio-codait-spark-benchmark
Run the CODAIT Spark benchmark against the Alluxio virtual filesystem

For more information regarding the CODAIT Spark benchmark framework, see: 
     https://codait.github.io/spark-bench/users-guide/installation/

# Prerequisites

- A Linux based cluster environment with tools such as git, wget and time
- A working Hadoop cluster (version 2.7+)
- A working Spark environment that accepts spark-submit jobs
- A working Alluxio virtual filesystem

# Usage

## Step 1. Install the CODAIT Spark benchmark files

Download the Spark benchmark framework file from the CODAIT git repo:

     wget https://github.com/CODAIT/spark-bench/releases/download/v99/spark-bench_2.3.0_0.4.0-RELEASE_99.tgz

Extract the tar file:

     tar -xvzf spark-bench_2.3.0_0.4.0-RELEASE_99.tgz

Change directory:

     cd spark-bench_2.3.0_0.4.0-RELEASE

Set the SPARK_HOME environment variable to point to your working spark installation:

     export SPARK_HOME=/usr/lib/spark  # <-- Change this to your spark home dir

Set the SPARK_MASTER to reference your spark master environment. If you have a full YARN setup, you may set it to "yarn" like this:

     export SPARK_MASTER=yarn          # <-- Change this to your spark master

For spark jobs to access the Alluxio virtual filesystem, they must be able to load the Alluxio client jar file. The client jar file must be loaded on each server that runs Spark drivers and executors. Set an environment variable to point to the location of the client jar file:

     export ALLUXIO_CLIENT_JAR=/opt/alluxio/client/alluxio-enterprise-2.7.0-2.4-client.jar

For spark jobs to access the Alluxio master node, the proper server or hostname must be specified. Set an environment variable to point to your Alluxio master node:

     export ALLUXIO_MASTER=localhost

## Step 2. Generate the kmeans dataset in HDFS

Create a configuration file that sets up a benchmark workload that generates about XXX GB of CSV and Parquet formated data in HDFS.

     cat <<EOT > generate-kmeans-data-in-hdfs.conf
     spark-bench = {
       spark-submit-config = [{
         spark-home = "$SPARK_HOME"
         spark-args = {
           master = "$SPARK_MASTER"
         }
         conf = {
           "spark.debug.maxToStringFields" = 100
         }
         suites-parallel = false
         workload-suites = [
           {
             descr = "Generate a dataset to be used by benchmark queries later"
             benchmark-output = "hdfs:///tmp/codait-spark-benchmark-hdfs/results-data-gen.csv"
             // We need to generate the dataset first through the data generator, then we take that dataset and convert it to Parquet.
             parallel = false
             workloads = [
               {
                 name = "data-generation-kmeans"
                 rows = 100000000
                 cols = 24
                 output = "hdfs:///tmp/codait-spark-benchmark-hdfs/kmeans-data.csv"
               },
               {
                 name = "sql"
                 query = "select * from input"
                 input = "hdfs:///tmp/codait-spark-benchmark-hdfs/kmeans-data.csv"
                 output = "hdfs:///tmp/codait-spark-benchmark-hdfs/kmeans-data.parquet"
               }
             ]
           }
         ]
       }]
     }
     EOT

Run the workload and generate the data in HDFS. Use the time program to record the amount of time it takes to run the workload:

     time ./bin/spark-bench.sh ./generate-kmeans-data-in-hdfs.conf

View the generated files in HDFS:

     hdfs dfs -ls -R /tmp/codait-spark-benchmark-hdfs

     hdfs dfs -du -h /tmp/codait-spark-benchmark-hdfs

## Step 3. Run the SQL benchmark query against the kmeans CSV dataset in HDFS

Create a configuration file that sets up a benchmark workload that queries the CSV kmeans dataset in HDFS:

     cat <<EOT > run-csv-sql-query-hdfs.conf
     spark-bench = {
       spark-submit-config = [{
         spark-home = "$SPARK_HOME"
         spark-args = {
           master = "$SPARK_MASTER"
         }
         conf = {
           // Any configuration you need for your setup goes here, like:
           // "spark.dynamicAllocation.enabled" = "false"
         }
         suites-parallel = false
         workload-suites = [
           {
             descr = "Run SQL queries over the CSV dataset in HDFS"
             benchmark-output = "console"
             parallel = false
             workloads = [
               {
                 name = "sql"
                 input = "hdfs:///tmp/codait-spark-benchmark-hdfs/kmeans-data.csv"
                 query = "select c0, c22 from input where c0 < -0.9"
                 cache = false
               }
             ]
           }
         ]
       }]
     }
     EOT

Run the workload to query the data in HDFS. Use the time program to record the amount of time it takes to run the workload:

     time ./bin/spark-bench.sh run-csv-sql-query-hdfs.conf

## Step 4. Generate the kmeans dataset in HDFS

Create a configuration file that sets up a benchmark workload that generates about XXX GB of CSV and Parquet formated data in the Alluxio virtual filesystem.

     cat <<EOT > generate-kmeans-data-in-alluxio.conf
     spark-bench = {
       spark-submit-config = [{
         spark-home = "$SPARK_HOME"
         spark-args = {
           master = "$SPARK_MASTER"
         }
         conf = {
           "spark.driver.extraClassPath" = "$ALLUXIO_CLIENT_JAR"
           "spark.executor.extraClassPath" = "$ALLUXIO_CLIENT_JAR"
           "spark.driver.extraJavaOptions" = " -Dalluxio.user.file.read.default=CACHE -Dalluxio.user.file.writetype.default=CACHE_THROUGH -Dalluxio.user.ufs.block.read.location.policy=alluxio.client.block.policy.DeterministicHashPolicy -Dalluxio.user.ufs.block.read.location.policy.deterministic.hash.shards=2"
           "spark.executor.extraJavaOptions" = " -Dalluxio.user.file.read.default=CACHE -Dalluxio.user.file.writetype.default=CACHE_THROUGH -Dalluxio.user.ufs.block.read.location.policy=alluxio.client.block.policy.DeterministicHashPolicy -Dalluxio.user.ufs.block.read.location.policy.deterministic.hash.shards=2"
         }
         suites-parallel = false
         workload-suites = [
           {
             descr = "Generate a dataset to be used by benchmark queries later"
             benchmark-output = "alluxio://$ALLUXIO_MASTER:19998/tmp/codait-spark-benchmark-alluxio/results-data-gen.csv"
             // We need to generate the dataset first through the data generator, then we take that dataset and convert it to Parquet.
             parallel = false
             workloads = [
               {
                 name = "data-generation-kmeans"
                 rows = 10000000
                 cols = 24
                 output = "alluxio://$ALLUXIO_MASTER:19998/tmp/codait-spark-benchmark-alluxio/kmeans-data.csv"
               },
               {
                 name = "sql"
                 query = "select * from input"
                 input = "alluxio://$ALLUXIO_MASTER:19998/tmp/codait-spark-benchmark-alluxio/kmeans-data.csv"
                 output = "alluxio://$ALLUXIO_MASTER:19998/tmp/codait-spark-benchmark-alluxio/kmeans-data.parquet"
               }
             ]
           }
         ]
       }]
     }
     EOT

Run the workload and generate the data in Alluxio. Use the time program to record the amount of time it takes to run the workload:

     time ./bin/spark-bench.sh ./generate-kmeans-data-in-alluxio.conf

Use the Alluxio filesystem command to see the generated files:

     alluxio fs ls -R /tmp/codait-spark-benchmark-alluxio

## Step 4. Run the SQL Query against the CSV dataset in Alluxio

Create a configuration file that sets up a benchmark workload that queries the CSV kmeans dataset in Alluxio:

     cat <<EOT > run-csv-sql-query-alluxio.conf
     spark-bench = {
       spark-submit-config = [{
         spark-home = "$SPARK_HOME"
         spark-args = {
           master = "$SPARK_MASTER"
         }
         conf = {
           "spark.driver.extraClassPath" = "$ALLUXIO_CLIENT_JAR"
           "spark.executor.extraClassPath" = "$ALLUXIO_CLIENT_JAR"
           "spark.driver.extraJavaOptions" = " -Dalluxio.user.file.read.default=CACHE -Dalluxio.user.file.writetype.default=CACHE_THROUGH -Dalluxio.user.ufs.block.read.location.policy=alluxio.client.block.policy.DeterministicHashPolicy -Dalluxio.user.ufs.block.read.location.policy.deterministic.hash.shards=2"
           "spark.executor.extraJavaOptions" = " -Dalluxio.user.file.read.default=CACHE -Dalluxio.user.file.writetype.default=CACHE_THROUGH -Dalluxio.user.ufs.block.read.location.policy=alluxio.client.block.policy.DeterministicHashPolicy -Dalluxio.user.ufs.block.read.location.policy.deterministic.hash.shards=2"
         }
         suites-parallel = false
         workload-suites = [
           {
             descr = "Run SQL queries over the CSV dataset in Alluxio"
             benchmark-output = "console"
             parallel = false
             workloads = [
               {
                 name = "sql"
                 input = "alluxio://$ALLUXIO_MASTER:19998/tmp/codait-spark-benchmark-alluxio/kmeans-data.csv"
                 query = "select c0, c22 from input where c0 < -0.9"
                 cache = false
               }
             ]
           }
         ]
       }]
     }
     EOT

Run the workload to query the data in Alluxio. Use the time program to record the amount of time it takes to run the workload:

     time ./bin/spark-bench.sh run-csv-sql-query-alluxio.conf

## Step 5. Remove the data files from HDFS and the Alluxio virtual filesystem

Run the HDFS command to remove the files in HDFS:

     hdfs dfs -rm -R /tmp/codait-spark-benchmark-hdfs

Run the Alluxio command to remove the files from the Alluxio virtual filesystem:

     alluxio fs rm -R /tmp/codait-spark-benchmark-alluxio

## Optional Step 6. Enabled detailed debug logging

To enable more detailed client-side logging, first create a log4j properties file, like this:

     cat <<EOT > /tmp/log4j.properties 
     log4j.rootCategory=debug,console
     log4j.logger.com.demo.package=debug,console
     log4j.additivity.com.demo.package=false
     log4j.appender.console=org.apache.log4j.ConsoleAppender
     log4j.appender.console.target=System.out
     log4j.appender.console.immediateFlush=true
     log4j.appender.console.encoding=UTF-8
     log4j.appender.console.layout=org.apache.log4j.PatternLayout
     log4j.appender.console.layout.conversionPattern=%d [%t] %-5p %c - %m%n
     EOT

Then, in the benchmark conf file (like run-csv-sql-query-alluxio.conf), add the log4j.configuration options to the extraJavaOptions setting. Like this:

     conf = {
       "spark.driver.extraClassPath" = "$ALLUXIO_CLIENT_JAR"
       "spark.executor.extraClassPath" = "$ALLUXIO_CLIENT_JAR"
       "spark.driver.extraJavaOptions" = " -Dalluxio.user.file.read.default=CACHE -Dalluxio.user.file.writetype.default=CACHE_THROUGH -Dalluxio.user.ufs.block.read.location.policy=alluxio.client.block.policy.DeterministicHashPolicy -Dalluxio.user.ufs.block.read.location.policy.deterministic.hash.shards=2 -Dlog4j.configuration=log4j.properties -Dlog4j.configuration=file:/tmp/log4j.properties"
       "spark.executor.extraJavaOptions" = " -Dalluxio.user.file.read.default=CACHE -Dalluxio.user.file.writetype.default=CACHE_THROUGH -Dalluxio.user.ufs.block.read.location.policy=alluxio.client.block.policy.DeterministicHashPolicy -Dalluxio.user.ufs.block.read.location.policy.deterministic.hash.shards=2 -Dlog4j.configuration=log4j.properties -Dlog4j.configuration=file:/tmp/log4j.properties"
     }

---

Please direct comments and questions to greg.palmer@alluxio.com

