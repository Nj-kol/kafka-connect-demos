# MySQL 

## Launch a MySQL Instance

```bash
docker container run -p 33006:3306 -d \
--name mysql-v8 \
--network sandbox \
-e MYSQL_ROOT_PASSWORD=asd \
-v mysql-v8-data:/var/lib/mysql \
-v ${HOME}/volumes/mysql_v8:/opt/mysql \
mysql:8.0.23

docker container exec -it mysql-v8 bash

Ex - mysql -u root -pasd
```

## MySQL config

* Debezium uses MySQL’s `bin-log` to track all the events, hence the bin-log should be enabled
  in MySql. Here are the steps to enable bin-log in MySQL. Check the current state of bin-log activation:

```bash
$ mysqladmin variables -uroot -pasd|grep log_bin

| log_bin                                                  | ON                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| log_bin_basename                                         | /var/lib/mysql/binlog                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| log_bin_index                                            | /var/lib/mysql/binlog.index                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| log_bin_trust_function_creators                          | OFF                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| log_bin_use_v1_row_events                                | OFF                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
```

* In order for Debezium to track all the changes in `bin-log`, it needs a user with 
  SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT permissions. 

   Grant them to the user you intend to use for Debezium.

```bash
$ mysql -u root -pasd
mysql> CREATE USER 'debezium'@'%' IDENTIFIED BY '‘dbz’';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'debezium'@'%';
```

## Kafka Connect Configuration

* Download `debezium-connector-mysql` plugin

```bash
# Download in the mounted path of container
cd ${HOME}/volumes/kafka-connect/jars

wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.5.0.Final/debezium-connector-mysql-1.5.0.Final-plugin.tar.gz
```

* Unpack the `.tar.gz` file

```bash
tar -xvzf debezium-connector-mysql-1.5.0.Final-plugin.tar.gz
```

* The exanded directory should look like the following

```bash
└── debezium-connector-mysql
    ├── CHANGELOG.md
    ├── CONTRIBUTE.md
    ├── COPYRIGHT.txt
    ├── LICENSE-3rd-PARTIES.txt
    ├── LICENSE.txt
    ├── README.md
    ├── README_ZH.md
    ├── antlr4-runtime-4.7.2.jar
    ├── debezium-api-1.5.0.Final.jar
    ├── debezium-connector-mysql-1.5.0.Final.jar
    ├── debezium-core-1.5.0.Final.jar
    ├── debezium-ddl-parser-1.5.0.Final.jar
    ├── failureaccess-1.0.1.jar
    ├── guava-30.0-jre.jar
    ├── mysql-binlog-connector-java-0.25.0.jar
    └── mysql-connector-java-8.0.21.jar
```

References
==========
https://blog.clairvoyantsoft.com/mysql-cdc-with-apache-kafka-and-debezium-3d45c00762e4

https://wecode.wepay.com/posts/streaming-databases-in-realtime-with-mysql-debezium-kafka

https://mvnrepository.com/artifact/io.debezium/debezium-connector-mysql 