Stream Processing with Apache Flink 
------------------------------------

<p align="center">
    <img src="assets/cover.png" width="600" height="800">
</p>

This repository contains the code for the book **[Stream Processing: Hands-on with Apache Flink](https://leanpub.com/streamprocessingwithapacheflink)**.


### Table of Contents
1. [Environment Setup](#environment-setup)
2. [Register UDF](#register-udf)
3. [Deploy a JAR file](#deploy-a-jar-file)


### Environment Setup
In order to run the code samples we will need a Kafka and Flink cluster up and running.
You can also run the Flink examples from within your favorite IDE in which case you don't need a Flink Cluster.

If you want to run the examples inside a Flink Cluster run to start the Pulsar and Flink clusters.
```shell
docker-compose up
```

When the cluster is up and running successfully run the following command for redpanda:
```shell
./redpanda-setup.sh

```

or this command for kafka setup
```shell
./kafka-setup.sh
```


### Register UDF
```shell
CREATE FUNCTION maskfn  AS 'io.streamingledger.udfs.MaskingFn'      LANGUAGE JAVA USING JAR '/opt/flink/jars/spf-0.1.0.jar';
CREATE FUNCTION splitfn AS 'io.streamingledger.udfs.SplitFn'        LANGUAGE JAVA USING JAR '/opt/flink/jars/spf-0.1.0.jar';
CREATE FUNCTION lookup  AS 'io.streamingledger.udfs.AsyncLookupFn'  LANGUAGE JAVA USING JAR '/opt/flink/jars/spf-0.1.0.jar';

CREATE TEMPORARY VIEW sample AS
SELECT * 
FROM transactions 
LIMIT 10;

SELECT transactionId, maskfn(UUID()) AS maskedCN FROM sample;
SELECT * FROM transactions, LATERAL TABLE(splitfn(operation));

SELECT 
  transactionId,
  serviceResponse, 
  responseTime 
FROM sample, LATERAL TABLE(lookup(transactionId));
```

### Deploy a JAR file
1. Package the application and create an executable jar file
```shell
mvn clan package
```
2. Copy it under the jar files to be included in the custom Flink images

3. Start the cluster to build the new images by running
```shell
docker-compose up
```

4. Deploy the flink job
```shell
docker exec -it jobmanager ./bin/flink run \
  --class io.streamingledger.datastream.BufferingStream \
  jars/spf-0.1.0.jar
```

### Flink Table Setup

The transactions table:

```sql
CREATE TABLE transactions (
    transactionId STRING,
    accountId STRING,
    customerId STRING,
    eventTime BIGINT,
    eventTime_ltz AS TO_TIMESTAMP_LTZ(eventTime, 3),
    eventTimeFormatted STRING,
    type STRING,
    operation STRING,
    amount DOUBLE,
    balance DOUBLE,
    WATERMARK FOR eventTime_ltz AS eventTime_ltz
) WITH (
    'connector' = 'kafka',
    'topic' = 'transactions',
    'properties.bootstrap.servers' = 'redpanda:9092',
    'properties.group.id' = 'group.transactions',
    'format' = 'json',
    'scan.startup.mode' = 'earliest-offset'
);
```

The customers table:

```sql
CREATE TABLE customers (
    customerId STRING,
    sex STRING,
    social STRING,
    fullName STRING,
    phone STRING,
    email STRING,
    address1 STRING,
    address2 STRING,
    city STRING,
    state STRING,
    zipcode STRING,
    districtId STRING,
    birthDate STRING,
    updateTime BIGINT,
    eventTime_ltz AS TO_TIMESTAMP_LTZ(updateTime, 3),
    WATERMARK FOR eventTime_ltz AS eventTime_ltz,
    PRIMARY KEY (customerId) NOT ENFORCED
) WITH (
    'connector' = 'upsert-kafka',
    'topic' = 'customers',
    'properties.bootstrap.servers' = 'redpanda:9092',
    'key.format' = 'raw',
    'value.format' = 'json',
    'properties.group.id' = 'group.customers'
);
```

The account tables:

```sql
CREATE TABLE accounts (
    accountId STRING,
    districtId INT,
    frequency STRING,
    creationDate STRING,
    updateTime BIGINT,
    eventTime_ltz AS TO_TIMESTAMP_LTZ(updateTime, 3),
    WATERMARK FOR eventTime_ltz AS eventTime_ltz,
    PRIMARY KEY (accountId) NOT ENFORCED
) WITH (
    'connector' = 'upsert-kafka',
    'topic' = 'accounts',
    'properties.bootstrap.servers' = 'redpanda:9092',
    'key.format' = 'raw',
    'value.format' = 'json',
    'properties.group.id' = 'group.accounts'
);
```

### Loading Data

To load data into the systems run `StateProducer` and `TransactionsProducer` with your IntelliJ.

### SQL Client

You can use the Flink SQL client by using the following command:

```shell
docker exec -it jobmanager ./bin/sql-client.sh
```

### Setup Databases

```sql
-- create database
CREATE DATABASE bank;
USE bank;

-- see above or queries package for the sql
```