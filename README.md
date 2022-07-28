# How to use DataStax's CQL Proxy tool

Learn how to take a program that is using an old python cassandra driver version to talk to dockerized Cassandra 3.11 and connect it to DataStax Astra via the CQL Proxy tool! This demo was built to run on Gitpod on the browser, so hit the button below to get started!

## Prerequisites
- Free DataStax Astra Account and Token
  - [Find out how to generate a token here.](https://docs.datastax.com/en/astra/docs/manage/org/manage-tokens.html) (for this demo, we created an admin token)

## Click below to get started!

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/Anant/example-cql-proxy)

### 1. Confirm the installs completed and Dockerized Cassandra has started
Each task in the `.gitpod.yml` file run in their own terminals that will start on Gitpod load. If CQLSH has not started, give it a few more seconds and re-run this command:
```bash
docker exec -it cassandra cqlsh
```


### 2. Run `data_importer.py` for dockerized Cassandra
#### 2.1 Run the below command with your desired keyspace and table names after replacing the placeholders
```bash
python data_importer.py $(hostname -I | awk '{print $2}') <keyspace_name> <table_name>
```

#### 2.2 Confirm your keyspace and table were created and data was loaded in CQLSH
```bash
select count(*) from <keyspace_name>.<table_name>
```

This should return 10

### 3. Start CQL Proxy
#### 3.1 Kill Cassandra Container
The reason we are doing this is because the proxy container utilized the 9042 port that the Dockerized Cassandra is using. 

```bash
docker kill cassandra
```

#### 3.2 Start CQL Proxy Container (Replace the placeholder with your token)
```bash
docker run -p 9042:9042 datastax/cql-proxy:v0.1.3 --astra-token <your_token>
```

### 4. Re-run `data_imported.py` using the CQL Proxy

#### 4.1 Run the below command with your keyspace and table name
```bash
python data_importer.py $(hostname -I | awk '{print $1}') <keyspace_name> <table_name>
```

#### 4.2 Wait, why didn't that work?
You may see an error such as the one below:
```console
Traceback (most recent call last):
  File "data_importer.py", line 24, in <module>
    session.execute(exec_command)
  File "cassandra/cluster.py", line 2289, in cassandra.cluster.Session.execute
  File "cassandra/cluster.py", line 4242, in cassandra.cluster.ResponseFuture.result
cassandra.Unauthorized: Error from server: code=2100 [Unauthorized] message="Missing correct permission on demo."
```
This is because we can't create a keyspace using CQL on Astra even with an admin token. You will either need to create a new keyspace for this demo using the UI, or use an already existing keyspace that we can add a table to.

#### 4.3 Comment out lines 20-24 in `data_importer.py' and re-run the command from 4.1 with a new keyspace in Astra or an existing one
```bash
sed -i "19,23 {s/^/#/}" data_importer.py
```

then

```bash
python data_importer.py $(hostname -I | awk '{print $1}') <keyspace_name> <table_name>
```

#### 4.4 Hey, wait, there's another error!
```console
Traceback (most recent call last):
  File "data_importer.py", line 147, in <module>
    insert_query = session.execute(
  File "cassandra/cluster.py", line 2289, in cassandra.cluster.Session.execute
  File "cassandra/cluster.py", line 4242, in cassandra.cluster.ResponseFuture.result
cassandra.InvalidRequest: Error from server: code=2200 [Invalid query] message="Provided value LOCAL_ONE is not allowed for Write Consistency Level (disallowed values are: [ANY, ONE, LOCAL_ONE])"
```

The reason we are getting this error is because the python cassandra driver has a default consistency level setting ([see for reference](https://docs.datastax.com/en/developer/python-driver/3.19/api/cassandra/#cassandra.ConsistencyLevel)). We can fix this by importing a `ConsistencyLevel` and setting the session's consistency level to something like LOCAL_QUORUM.

#### 4.5 Update Session Consistency Level by running the command below
```bash
sed -i 's/# session.default_consistency_level = ConsistencyLevel.LOCAL_QUORUM/session.default_consistency_level = ConsistencyLevel.LOCAL_QUORUM/' data_importer.py 
```

#### 4.6 Comment out Table creation
```bash
sed -i "26,30 {s/^/#/}" data_importer.py
```

#### 4.7 Re-run `data_importer.py` with the command below
```bash
python data_importer.py $(hostname -I | awk '{print $1}') <keyspace_name> <table_name>
```

### 5. Confirm CQL Proxy worked

#### 5.1 Run a count command in the Astra CQLSH terminal
```bash
select count(*) from <keyspace_name>.<table_name>
```

#### 5.2 Update number of records imported from 10 to 1000
```bash
sed -i 's/rows = 10/rows = 1000/' data_importer.py 
```

#### 5.3 Re-run `data_importer.py` with the command below
```bash
python data_importer.py $(hostname -I | awk '{print $1}') <keyspace_name> <table_name>
```

#### 5.4 Confirm new count with command in the Astra CQLSH terminal
```bash
select count(*) from <keyspace_name>.<table_name>
```
