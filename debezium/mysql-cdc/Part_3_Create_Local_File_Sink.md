
# Create a Local File Sink

* Create a local file sink for Kafka topic

```json
{
    "name": "local-file-sink",
    "config": {
        "connector.class": "FileStreamSink",
        "tasks.max": 1,
        "file": "/tmp/quickstart/emp.json",
        "topics": "mysql-v8.company.emp",
        "key.converter" : "org.apache.kafka.connect.storage.StringConverter",
        "value.converter" : "org.apache.kafka.connect.storage.StringConverter"
    }
}
```

* Create the File Sink connection

```bash
# Login to Kakfa Connect worker
docker container exec -it kafka-connect-demo bash

curl -X POST \
-H "Content-Type: application/json" \
--data '{"name":"local-file-sink","config":{"connector.class":"FileStreamSink","tasks.max":1,"file":"/tmp/quickstart/emp.json","topics":"mysql-v8.company.emp","key.converter":"org.apache.kafka.connect.storage.StringConverter","value.converter":"org.apache.kafka.connect.storage.StringConverter"}}' \
http://localhost:28082/connectors/
```

* Response

```json
{
   "name":"local-file-sink",
   "config":{
      "connector.class":"FileStreamSink",
      "tasks.max":"1",
      "file":"/tmp/quickstart/emp.json",
      "topics":"mysql-v8.company.emp",
      "name":"local-file-sink"
   },
   "tasks":[
      
   ],
   "type":"sink"
}
```

## Check status of connector

```bash
# Login to Kakfa Connect worker
docker container exec -it kafka-connect-demo bash

curl -s -X GET http://localhost:28082/connectors/local-file-sink/status 
```

## Check whether sink is working

* Add some new records

```sql
use company;

INSERT INTO emp VALUES (7844,'TURNER','SALESMAN',7698,STR_TO_DATE('8-09-1981','%d-%m-%Y'),1500,30);
INSERT INTO emp VALUES (7876,'ADAMS','CLERK',7788,STR_TO_DATE('13-06-87', '%d-%m-%Y')-51,1100,20);
INSERT INTO emp VALUES (7900,'JAMES','CLERK',7698,STR_TO_DATE('3-12-1981','%d-%m-%Y'),950,30);
INSERT INTO emp VALUES (7902,'FORD','ANALYST',7566,STR_TO_DATE('3-12-1981','%d-%m-%Y'),3000,20);
INSERT INTO emp VALUES (7934,'MILLER','CLERK',7782,STR_TO_DATE('23-10-1982','%d-%m-%Y'),1300,10);

-- Add other operations types
UPDATE emp SET sal=3010 WHERE empno=7902;
UPDATE emp SET sal=3020 WHERE empno=7902;

DELETE FROM emp WHERE empno=7934;
```

* Check the file

```bash
docker container exec -it kafka-connect-demo bash

root@docker-desktop:/tmp/quickstart# head -1 emp.json
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"string","optional":true,"field":"empno"},{"type":"string","optional":true,"field":"ename"},{"type":"string","optional":true,"field":"job"},{"type":"string","optional":true,"field":"mgr"},{"type":"int32","optional":true,"name":"io.debezium.time.Date","version":1,"field":"updated_date"},{"type":"int32","optional":true,"field":"sal"},{"type":"int32","optional":true,"field":"deptno"}],"
...
```

References
===========
https://www.baeldung.com/kafka-connectors-guide