Сощдаю таблицу:
```
CREATE TABLE test (
      IntColumn INT,
      UUIDColumn VARCHAR(100)
   );

INSERT INTO test (IntColumn, UUIDColumn)
SELECT
  generate_series(1,1000) AS RandomInteger,
  md5(random()::text) AS UniqueIdentifier;
  
 SELECT * FROM test;
```

![image](https://github.com/FedotovaVera/postgres/assets/111522773/024c0966-0183-43ff-b87b-0c4d9ecf4bc6)
