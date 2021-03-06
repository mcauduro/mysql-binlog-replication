# Streaming mysql binlog replication to Snowflake/Redshift/BigQuery
[![CircleCI](https://circleci.com/gh/trainingrocket/mysql-binlog-replication.svg?style=svg)](https://circleci.com/gh/trainingrocket/mysql-binlog-replication)

```bash
docker-compose up --build
```

> Note: If you get `Can't connect to MySQL server on 'mysql' ([Errno 111] Connection refused)` error on the first run, try running it again.

We can actually merge all these images into a single image, but I personally prefer it for simplicity at the expense of code duplication.

In another terminal login into the mysql instance
```bash
docker-compose exec mysql mysql -u root -pexample
```
And execute the following
```sql
DROP DATABASE IF EXISTS testdb;
CREATE DATABASE testdb; USE testdb;
CREATE TABLE testtbl (id int, name varchar(255));
INSERT INTO testtbl VALUES (1, 'hello'), (2, 'hola'), (3, 'zdravstvuy'), (1, 'bonjour');
DELETE FROM testtbl WHERE id = 1;
SELECT * FROM testtbl;
```

Or you can just
```bash
docker-compose exec mysql mysql -u root -pexample -e "DROP DATABASE IF EXISTS testdb; CREATE DATABASE testdb; USE testdb; CREATE TABLE testtbl (id int, name varchar(255)); INSERT INTO testtbl VALUES (1, 'hello'), (2, 'hola'), (3, 'zdravstvuy'), (1, 'bonjour'); DELETE FROM testtbl WHERE id = 1; SELECT * FROM testtbl;"
```

Which will output the following to the terminal
```
+------+------------+
| id   | name       |
+------+------------+
|    2 | hola       |
|    3 | zdravstvuy |
+------+------------+
```

`docker-compose` daemon should output something like this
```sql
python_1  | {"type": "WriteRowsEvent", "row": {"values": {"name": "hello", "id": 1}}, "table": "testtbl", "schema": "testdb"}
python_1  | INSERT INTO `testdb`.`testtbl`(`name`, `id`) VALUES ('hello', 1);
python_1  | {"type": "WriteRowsEvent", "row": {"values": {"name": "hola", "id": 2}}, "table": "testtbl", "schema": "testdb"}
python_1  | INSERT INTO `testdb`.`testtbl`(`name`, `id`) VALUES ('hola', 2);
python_1  | {"type": "WriteRowsEvent", "row": {"values": {"name": "zdravstvuy", "id": 3}}, "table": "testtbl", "schema": "testdb"}
python_1  | INSERT INTO `testdb`.`testtbl`(`name`, `id`) VALUES ('zdravstvuy', 3);
python_1  | {"type": "WriteRowsEvent", "row": {"values": {"name": "bonjour", "id": 1}}, "table": "testtbl", "schema": "testdb"}
python_1  | INSERT INTO `testdb`.`testtbl`(`name`, `id`) VALUES ('bonjour', 1);
python_1  | {"type": "DeleteRowsEvent", "row": {"values": {"name": "hello", "id": 1}}, "table": "testtbl", "schema": "testdb"}
python_1  | DELETE FROM `testdb`.`testtbl` WHERE `name`='hello' AND `id`=1 LIMIT 1;
python_1  | {"type": "DeleteRowsEvent", "row": {"values": {"name": "bonjour", "id": 1}}, "table": "testtbl", "schema": "testdb"}
python_1  | DELETE FROM `testdb`.`testtbl` WHERE `name`='bonjour' AND `id`=1 LIMIT 1;
```

# Change Data Capture from RDS instance

[Update RDS parameter group](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_LogAccess.Concepts.MySQL.html#USER_LogAccess.MySQL.BinaryFormat) to make sure `log_bin` is enabled and set `binlog_format` to `ROW`.
Update the environment variables with RDS credentials, `MYSQL_HOST` should be something like `testdb.xxx.us-west-2.rds.amazonaws.com`.

```
mysql> show global variables like 'log_bin'; show global variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.64 sec)

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.61 sec)
```

# References
- https://www.alooma.com/blog/mysql-to-amazon-redshift-replication
- https://aws.amazon.com/blogs/database/streaming-changes-in-a-database-with-amazon-kinesis/
- https://github.com/danfengcao/binlog2sql
- https://www.thegeekstuff.com/2017/08/mysqlbinlog-examples/