# ETL with Spark

* Launch a spark shell locally

```bash
spark-shell --master local 
```

## Do ETL to see CDC data in tabular format

* There are three record types which is specified in the column `op` :
   - c (create)
   - u (update)
   - d (delete)

```java
val file = "/Users/nilanjan1.sarkar/volumes/kafka-connect/file/emp.json"

val df = spark.read.json(file).select("payload.*")

df.createOrReplaceTempView("emp_source")

val iuQuery = """
select after.empno,after.ename,after.job,after.mgr,after.sal,after.deptno,after.created_date,source.pos,op,ts_ms from emp_source
where op IS NOT NULL
and after.empno IS NOT NULL 
"""

// Insert and update records
val iuDf = spark.sql(iuQuery)

+-----+------+--------+----+----+------+------------+----+---+-------------+
|empno|ename |job     |mgr |sal |deptno|created_date|pos |op |ts_ms        |
+-----+------+--------+----+----+------+------------+----+---+-------------+
|7499 |ALLEN |SALESMAN|7698|1600|30    |4068        |5487|c  |1619362972540|
|7521 |WARD  |SALESMAN|7698|1250|30    |4070        |5820|c  |1619362972546|
|7566 |JONES |MANAGER |7839|2975|20    |4109        |6152|c  |1619362972552|
|7654 |MARTIN|SALESMAN|7698|1250|30    |4288        |6484|c  |1619362972556|
|7844 |TURNER|SALESMAN|7698|1500|30    |4268        |6818|c  |1619362994801|
|7900 |JAMES |CLERK   |7698|950 |30    |4354        |7152|c  |1619362994809|
|7902 |FORD  |ANALYST |7566|3000|20    |4354        |7482|c  |1619362994815|
|7934 |MILLER|CLERK   |7782|1300|10    |4678        |7813|c  |1619362994819|
|7902 |FORD  |ANALYST |7566|3010|20    |4354        |8484|u  |1619363214503|
|7902 |FORD  |ANALYST |7566|3020|20    |4354        |8864|u  |1619363214509|
+-----+------+--------+----+----+------+------------+----+---+-------------+

val deleteQuery = """
select before.empno,before.ename,before.job,before.mgr,before.sal,before.deptno,before.created_date,source.pos,op,ts_ms from emp_source
where 
after IS NULL
and before.empno IS NOT NULL 
"""

val delDf = spark.sql(deleteQuery)

delDf.show(false)

+-----+------+-----+----+----+------+------------+----+---+-------------+
|empno|ename |job  |mgr |sal |deptno|created_date|pos |op |ts_ms        |
+-----+------+-----+----+----+------+------------+----+---+-------------+
|7934 |MILLER|CLERK|7782|1300|10    |4678        |8144|d  |1619363017427|
+-----+------+-----+----+----+------+------------+----+---+-------------+

// Get a uniform view of all create,update and delete records
val logstateDf = iuDf.unionAll(delDf)

logstateDf.show(false)

+-----+------+--------+----+----+------+------------+----+---+-------------+
|empno|ename |job     |mgr |sal |deptno|created_date|pos |op |ts_ms        |
+-----+------+--------+----+----+------+------------+----+---+-------------+
|7499 |ALLEN |SALESMAN|7698|1600|30    |4068        |5487|c  |1619362972540|
|7521 |WARD  |SALESMAN|7698|1250|30    |4070        |5820|c  |1619362972546|
|7566 |JONES |MANAGER |7839|2975|20    |4109        |6152|c  |1619362972552|
|7654 |MARTIN|SALESMAN|7698|1250|30    |4288        |6484|c  |1619362972556|
|7844 |TURNER|SALESMAN|7698|1500|30    |4268        |6818|c  |1619362994801|
|7900 |JAMES |CLERK   |7698|950 |30    |4354        |7152|c  |1619362994809|
|7902 |FORD  |ANALYST |7566|3000|20    |4354        |7482|c  |1619362994815|
|7934 |MILLER|CLERK   |7782|1300|10    |4678        |7813|c  |1619362994819|
|7902 |FORD  |ANALYST |7566|3010|20    |4354        |8484|u  |1619363214503|
|7902 |FORD  |ANALYST |7566|3020|20    |4354        |8864|u  |1619363214509|
|7934 |MILLER|CLERK   |7782|1300|10    |4678        |8144|d  |1619363017427|
+-----+------+--------+----+----+------+------------+----+---+-------------+

logstateDf.createOrReplaceTempView("emp_log")

// Take the latest entry for each employeee
val latestRecordQuery = """
WITH latest as (
select *, RANK() OVER ( PARTITION BY empno ORDER BY pos desc ) as rnk
from emp_log
)
select * from latest where rnk=1
and op <> 'd'
"""

// Drop Debezium specific columns
val debCols = Seq("pos","op","ts_ms","rnk")
val latestDf = spark.sql(latestRecordQuery).drop(debCols:_*)

latestDf.show(false)

+-----+------+--------+----+----+------+------------+
|empno|ename |job     |mgr |sal |deptno|created_date|
+-----+------+--------+----+----+------+------------+
|7521 |WARD  |SALESMAN|7698|1250|30    |4070        |
|7499 |ALLEN |SALESMAN|7698|1600|30    |4068        |
|7654 |MARTIN|SALESMAN|7698|1250|30    |4288        |
|7844 |TURNER|SALESMAN|7698|1500|30    |4268        |
|7566 |JONES |MANAGER |7839|2975|20    |4109        |
|7900 |JAMES |CLERK   |7698|950 |30    |4354        |
|7902 |FORD  |ANALYST |7566|3020|20    |4354        | <- Reflects latest version 
+-----+------+--------+----+----+------+------------+
```

Also notice that record with empno 7934 has been dropped. 

Thus we have managed to create the exact version of the data as it exists at source using
a log based CDC approach 

References
===========
https://www.startdataengineering.com/post/change-data-capture-using-debezium-kafka-and-pg/