# Integrate Apache Kafka and Azure Cosmos DB Cassandra API using Kafka Connect

This is an example of how to use the open-source [DataStax Apache Kafka connector](https://docs.datastax.com/en/kafka/doc/kafka/kafkaIntro.html) that works on top of Kafka Connect framework to ingest records from a Kafka topic into rows of one or more Cassandra table(s). The example provides a re-usable setup using Docker Compose. This is really convenient since it enables you to bootstrap all the required components locally with a single command. These components include: Kafka, Zookeeper, Kafka Connect worker and the sample data generator application.

## Prerequisites

* [Provision an Azure Cosmos DB Cassandra API account](https://docs.microsoft.com/azure/cosmos-db/create-cassandra-go?WT.mc_id=data-11436-abhishgu#create-a-database-account)

* [Use cqlsh or hosted shell for validation](cassandra-support.md#hosted-cql-shell-preview)

* Install [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install)


## Create Keyspace, tables and start the integration pipeline

Using the Azure portal, create the Cassandra Keyspace and the tables required for the demo application.

> Use the same Keyspace and table names as below

```sql
CREATE KEYSPACE weather WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'datacenter1' : 1};

CREATE TABLE weather.data_by_state (station_id text, temp int, state text, ts timestamp, PRIMARY KEY (state, ts)) WITH CLUSTERING ORDER BY (ts DESC) AND cosmosdb_cell_level_timestamp=true AND cosmosdb_cell_level_timestamp_tombstones=true AND cosmosdb_cell_level_timetolive=true;

CREATE TABLE weather.data_by_station (station_id text, temp int, state text, ts timestamp, PRIMARY KEY (station_id, ts)) WITH CLUSTERING ORDER BY (ts DESC) AND cosmosdb_cell_level_timestamp=true AND cosmosdb_cell_level_timestamp_tombstones=true AND cosmosdb_cell_level_timetolive=true;
```

Clone the GitHub repo:

```bash
git clone https://github.com/Azure-Samples/cosmosdb-cassandra-kafka
cd cosmosdb-cassandra-kafka
```

Start all the services:

```shell
docker-compose --project-name kafka-cosmos-cassandra up --build
```

> It might take a while to download and start the containers: this is just a one time process.

To confirm whether all the containers have started:

```shell
docker-compose -p kafka-cosmos-cassandra ps
```

The data generator application will start pumping data into the `weather-data` topic in Kafka. You can also do quick sanity check to confirm. Peek into the Docker container running the Kafka connect worker:


```bash
docker exec -it kafka-cosmos-cassandra_cassandra-connector_1 bash
```

Once you drop into the container shell, just start the usual Kafka console consumer process and you should see weather data (in JSON format) flowing in.

```bash
cd ../bin
./kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic weather-data
```

## Cassandra Sink connector setup

Copy the JSON contents below to a file (you can name it `cassandra-sink-config.json`). You will need to update it as per your setup and the rest of this section will provide guidance around this topic.

```json
{
    "name": "kafka-cosmosdb-sink",
    "config": {
        "connector.class": "com.datastax.oss.kafka.sink.CassandraSinkConnector",
        "tasks.max": "1",
        "topics": "weather-data",
        "contactPoints": "<cosmos db account name>.cassandra.cosmos.azure.com",
        "port": 10350,
        "loadBalancing.localDc": "<cosmos db region e.g. Southeast Asia>",
        "auth.username": "<enter username for cosmosdb account>",
        "auth.password": "<enter password for cosmosdb account>",
        "ssl.hostnameValidation": true,
        "ssl.provider": "JDK",
        "ssl.keystore.path": "/etc/alternatives/jre/lib/security/cacerts/",
        "ssl.keystore.password": "changeit",
        "datastax-java-driver.advanced.connection.init-query-timeout": 5000,
        "maxConcurrentRequests": 500,
        "maxNumberOfRecordsInBatch": 32,
        "queryExecutionTimeout": 30,
        "connectionPoolLocalSize": 4,
        "topic.weather-data.weather.data_by_state.mapping": "station_id=value.stationid, temp=value.temp, state=value.state, ts=value.created",
        "topic.weather-data.weather.data_by_station.mapping": "station_id=value.stationid, temp=value.temp, state=value.state, ts=value.created",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": false,
        "offset.flush.interval.ms": 10000
    }
}
```

### Install the connector

Install the connector using the Kafka Connect REST endpoint:

```shell
curl -X POST -H "Content-Type: application/json" --data @cassandra-sink-config.json http://localhost:8083/connectors
```

To check the status:

```
curl http://localhost:8080/connectors/kafka-cosmosdb-sink/status
```

If all goes well, the connector should start weaving its magic. It should authenticate to Azure Cosmos DB and start ingesting data from the Kafka topic (`weather-data`) into Cassandra tables - `weather.data_by_state` and `weather.data_by_station`

You can now query data in the tables. Head over to the Azure portal, bring up the hosted CQL Shell for your Azure Cosmos DB account.

## Query Azure Cosmos DB

Check the `data_by_state` and `data_by_station` tables. Here is some sample queries to get you started:

```sql
select * from weather.data_by_state where state = 'state-1';
select * from weather.data_by_state where state IN ('state-1', 'state-2');
select * from weather.data_by_state where state = 'state-3' and ts > toTimeStamp('2020-11-26');

select * from weather.data_by_station where station_id = 'station-1';
select * from weather.data_by_station where station_id IN ('station-1', 'station-2');
select * from weather.data_by_station where station_id IN ('station-2', 'station-3') and ts > toTimeStamp('2020-11-26');
```

## Clean up resources

[Delete the Azure Cosmos DB account](https://docs.microsoft.com/azure/cosmos-db/create-cassandra-go?WT.mc_id=data-11436-abhishgu#clean-up-resources) once you've finished.