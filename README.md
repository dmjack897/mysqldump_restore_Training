# mysqldump와 restore 실습 내용을 기록합니다.

우선 실습에 사용할 대용량의 데이터가 들어있는 Table을 생성합니다.
```
mysql> create table person (id varchar(30), first_name varchar(30),last_name varchar(30),email varchar(30),gender varchar(30),ip_address varchar(30));
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+---------------+
| Tables_in_sim |
+---------------+
| person        |
+---------------+
1 row in set (0.00 sec)

mysql> select count(*) from person;
+----------+
| count(*) |
+----------+
|        0 |
+----------+
1 row in set (0.00 sec)

mysql> DELIMITER $$
mysql> DROP PROCEDURE IF EXISTS loopInsert$$
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql>  
mysql> CREATE PROCEDURE loopInsert()
    -> BEGIN
    ->     DECLARE i INT DEFAULT 1;
    ->         
    ->     WHILE i <= 5000000 DO
    ->         INSERT INTO person(id , first_name, last_name , email , gender , ip_address)
    ->           VALUES(concat('id',i), concat('성',i), concat('이름',i),concat('이메일',i),concat('성별',i),concat('ip주소',i));
    ->         SET i = i + 1;
    ->     END WHILE;
    -> END$$
Query OK, 0 rows affected (0.00 sec)

mysql> DELIMITER ;
mysql> CALL loopInsert;
Query OK, 1 row affected (25 min 25.80 sec)

mysql> 
mysql> select count(*) from person;
+----------+
| count(*) |
+----------+
|  5000000 |
+----------+
1 row in set (1.38 sec)
```

총 3가지 패턴의 backup을 진행해보고 비교해보겠습니다.

1. sim DB를 backup
- mysqldump -u root -p sim > total.db

2. sim DB의 person table만 backup
- mysqldump -u root -p school student > total.db

3. sim DB의 table 구조만 backup
- mysqldump -u root -p -d sim > total.db

