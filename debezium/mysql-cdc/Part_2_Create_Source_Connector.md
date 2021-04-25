
# Create a Kafka Connect Source Connector

## Set up sample data MySQL CDC

```bash
$ mysql -u root -pasd
```

```sql
CREATE DATABASE company;

USE company;

CREATE TABLE emp (
empno    LONG,
ename    VARCHAR(10),
job      VARCHAR(9),
mgr      LONG,
created_date date,
sal      INT,
deptno   INT);

-- Add some intial records
INSERT INTO emp VALUES (7499,'ALLEN','SALESMAN',7698,STR_TO_DATE('20-02-1981','%d-%m-%Y'),1600,30);
INSERT INTO emp VALUES (7521,'WARD','SALESMAN',7698,STR_TO_DATE('22-02-1981','%d-%m-%Y'),1250,30);
INSERT INTO emp VALUES (7566,'JONES','MANAGER',7839,STR_TO_DATE('2-04-1981','%d-%m-%Y'),2975,20);
INSERT INTO emp VALUES (7654,'MARTIN','SALESMAN',7698,STR_TO_DATE('28-09-1981','%d-%m-%Y'),1250,30);

SELECT * FROM emp;
```

## Debezium Connector configuration

```json
{
 "name": "company-connector",
 "config": {
     "connector.class": "io.debezium.connector.mysql.MySqlConnector",
     "tasks.max": "1",
     "database.hostname": "localhost",
     "database.port": "33006",
     "database.user": "debezium",
     "database.password": "dbz",
     "database.server.id": "1234",
     "database.server.name": "mysql-v8",
     "database.ssl.mode" : "preferred",
     "database.whitelist": "company",
     "table.whitelist": "company.emp",
     "poll.interval.ms": 1000,
     "database.history.kafka.bootstrap.servers": "localhost:9092",
     "database.history.kafka.topic": "schema-changes.company"
     }
}
```

* Here we are are only capturing the CDC for one table which is `emp`. This is controlled by 
  the properties 

- `database.whitelist`
- `table.whitelist`

* Create the JDBC Source connection

```bash
# Login to Kakfa Connect worker
docker container exec -it kafka-connect-demo bash

curl -X POST \
-H "Content-Type: application/json" \
--data '{"name":"company-connector","config":{"connector.class":"io.debezium.connector.mysql.MySqlConnector","tasks.max":"1","database.hostname":"localhost","database.port":"33006","database.user":"debezium","database.password":"dbz","database.server.id":"1234","database.server.name":"mysql-v8","database.ssl.mode":"preferred","database.whitelist":"company","table.whitelist":"company.emp","poll.interval.ms":1000,"database.history.kafka.bootstrap.servers":"localhost:9092","database.history.kafka.topic":"schema-changes.company"}}' \
http://localhost:28082/connectors/
```

* Response :

```json
{
   "name":"company-connector1",
   "config":{
      "connector.class":"io.debezium.connector.mysql.MySqlConnector",
      "tasks.max":"1",
      "database.hostname":"localhost",
      "database.port":"33006",
      "database.user":"debezium",
      "database.password":"dbz",
      "database.server.id":"1234",
      "database.server.name":"mysql-v8",
      "database.ssl.mode":"preferred",
      "database.whitelist":"company",
      "table.whitelist":"company.emp",
      "include.schema.changes":"true",
      "poll.interval.ms":"1000",
      "database.history.kafka.bootstrap.servers":"localhost:9092",
      "database.history.kafka.topic":"schema-changes.company",
      "name":"company-connector1"
   },
   "tasks":[
      
   ],
   "type":"source"
}
```

* Now you'll see new topics got created in kafka

```bash
docker container exec -it kafka-standalone bash

bash-4.4# kafka-topics.sh --list --zookeeper localhost:2181
__consumer_offsets
connect-config-storage
connect-offset-storage
connect-status-storage
mysql-v8
mysql-v8.company.emp
schema-changes.company
```

## Check status of connector

```bash
# Login to Kakfa Connect worker
docker container exec -it kafka-connect-demo bash

curl -s -X GET http://localhost:28082/connectors/company-connector/status 
```

```json
{
   "name":"company-connector",
   "connector":{
      "state":"RUNNING",
      "worker_id":"localhost:28082"
   },
   "tasks":[
      {
         "id":0,
         "state":"RUNNING",
         "worker_id":"localhost:28082"
      }
   ],
   "type":"source"
}
```

## See all the existing data ingested

```bash
kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic mysql-v8.company.emp \
--from-beginning 
```

## See new data ( CDC )

* Keep a Kafka consumer open in another terminal

```bash
kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic mysql-v8.company.emp
```

* Add some new records

```sql
use company;

INSERT INTO emp VALUES (7698,'BLAKE','MANAGER',7839,STR_TO_DATE('1-05-1981','%d-%m-%Y'),2850,30);
INSERT INTO emp VALUES (7782,'CLARK',K'MANAGER',7839,STR_TO_DATE('9-06-1981','%d-%m-%Y'),2450,10);
INSERT INTO emp VALUES (7788,'SCOTT','ANALYST',7566,STR_TO_DATE('13-06-87','%d-%m-%Y')-85,3000,20);
INSERT INTO emp VALUES (7839,'KING','PRESIDENT',NULL,STR_TO_DATE('17-11-1981','%d-%m-%Y'),5000,10);
```

You'll see in the Kafka Console consumer, new records getting added

