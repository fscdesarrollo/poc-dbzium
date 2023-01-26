# poc-dbzium
This is a repository to document a pc using DBZium to monitor database changes and send events via Kafka

## Support Links

- https://docs.confluent.io/kafka-connectors/debezium-sqlserver-source/current/overview.html#set-up-sql-server-using-docker-optional
- https://debezium.io/documentation/reference/connectors/sqlserver.html#setting-up-sqlserver
- https://www.paradigmadigital.com/dev/primeros-pasos-con-debezium/

## Docker Images
#### ZOOKEEPER
`docker run -d --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.5`

#### KAFKA
`docker run -d --rm --name kafka -p 9092:9092 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://host.docker.internal:9092 --link zookeeper:zookeeper debezium/kafka:1.5`

#### MYSQL
`docker run -d --rm --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.5`

#### SQL-SERVER
`docker run -d --rm -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=Admin789" -e "MSSQL_AGENT_ENABLED=true" -e "MSSQL_PID=Standard" -p 1433:1433 --name sqlserver --hostname sqlserver -d mcr.microsoft.com/mssql/server:2022-latest`

#### KAFKA-CONNECT
##### SQL SERVER
`docker run -d --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link sqlserver:sqlserver debezium/connect:1.5`

##### MY-SQL
`docker run -d --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:1.5`

## Sql Server Database Example Script

```
-- Create the test database
CREATE DATABASE TestDBZium;
GO

-- Enable CDC ---
USE TestDBZium;
EXEC sys.sp_cdc_enable_db;

-- Create some customers --
CREATE TABLE Customers (
  Id INTEGER IDENTITY(1001,1) NOT NULL PRIMARY KEY,
  FirstName VARCHAR(255) NOT NULL,
  LastName VARCHAR(255) NOT NULL,
  Email VARCHAR(255) NOT NULL UNIQUE
);
INSERT INTO Customers (FirstName, LastName, Email)
  VALUES ('Sally','Thomas','sally.thomas@acme.com');
INSERT INTO Customers (FirstName, LastName, Email)
  VALUES ('George','Bailey','gbailey@foobar.com');
INSERT INTO Customers (FirstName, LastName, Email)
  VALUES ('Edward','Walker','ed@walker.com');
INSERT INTO Customers (FirstName, LastName, Email)
  VALUES ('Anne','Kretchmar','annek@noanswer.org');

-- Enable CDC for Table ---
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', 
	@source_name = 'Customers', 
	@role_name = NULL, 
	@supports_net_changes = 0;
GO

-- List of Tables with CDC Enable --
SELECT s.name AS Schema_Name, tb.name AS Table_Name
, tb.object_id, tb.type, tb.type_desc, tb.is_tracked_by_cdc
FROM sys.tables tb
INNER JOIN sys.schemas s on s.schema_id = tb.schema_id
WHERE tb.is_tracked_by_cdc = 1
```

## Source Connectors

### Connector Sql Server

```
POST http://localhost:8083/connectors
{
 "name": "inventory-connector-sql-server",
 "config": {
     "connector.class" : "io.debezium.connector.sqlserver.SqlServerConnector",
     "tasks.max" : "1",
     "database.server.name" : "sqlserver",
     "database.hostname" : "sqlserver",
     "database.port" : "1433",
     "database.user" : "sa",
     "database.password" : "Admin789",
     "database.dbname" : "TestDBZium",
     "database.history.kafka.bootstrap.servers" : "kafka:9092",
     "database.history.kafka.topic": "schema-changes.inventory"
     }
 }
```

### Connetor MySql

```
POST http://localhost:8083/connectors
{
    "name": "inventory-connector-mysql",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "database.server.name": "dbserver1",
        "database.include.list": "inventory",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "schema-changes.inventory"
    }
}
```

