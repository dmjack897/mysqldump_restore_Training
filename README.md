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

##백업_TEST

총 3가지 패턴의 backup을 진행해보고 비교해보겠습니다.
확실한 시간 조사를 위해 짧은 shell script를 작성해서 실행했습니다.

1. sim DB를 backup
- mysqldump -u root -p sim > total.db
- 총 16초의 시간이 걸렸습니다.
```
#!/bin/sh
LOG="./`basename $0`_`date "+%Y%m%d"`.log"
{
    echo "start:"`date`
    mysqldump -u root -p -A > total_backup.db
    echo "end:"`date`
} 2>&1 | 
while read msg;
do
    echo "$msg" >> ${LOG}
done

///결과
MacBook-Pro-5:mysql_study simdongmok$ cat backup.sh_20231011.log
start:2023年 10月11日 水曜日 06時37分42秒 JST
end:2023年 10月11日 水曜日 06時37分58秒 JST

MacBook-Pro-5:mysql_study simdongmok$ ls -lh total_backup.db
-rw-r--r--  1 simdongmok  staff   452M 10 11 06:37 total_backup.db

```

2. sim DB의 person table만 backup
- mysqldump -u root -p school student > total.db
- 총 15초의 시간이 걸렸습니다.
```
#!/bin/sh
LOG="./`basename $0`_`date "+%Y%m%d"`.log"
{
    echo "start:"`date`
    mysqldump -u root -p sim person > table_backup.db
    echo "end:"`date`
} 2>&1 | 
while read msg;
do
    echo "$msg" > ${LOG}
done

///결과
MacBook-Pro-5:mysql_study simdongmok$ cat backup.sh_20231011.log 
start:2023年 10月11日 水曜日 06時45分58秒 JST
end:2023年 10月11日 水曜日 06時46分13秒 JST

MacBook-Pro-5:mysql_study simdongmok$ ls -lh backup.sh_20231011.log 
-rw-r--r--  1 simdongmok  staff   108B 10 11 06:46 backup.sh_20231011.log
MacBook-Pro-5:mysql_study simdongmok$ 
```

3. sim DB의 table 구조만 backup
- mysqldump -u root -p -d sim > total.db
- 총 2초의 시간이 걸렸습니다.
```
#!/bin/sh
LOG="./`basename $0`_`date "+%Y%m%d"`.log"
{
    echo "start:"`date`
    mysqldump -u root -p -d sim > structure_backup.db    
    echo "end:"`date`
} 2>&1 | 
while read msg;
do
    echo "$msg" >> ${LOG}
done

///결과
MacBook-Pro-5:mysql_study simdongmok$ cat backup.sh_20231011.log
start:2023年 10月11日 水曜日 06時51分14秒 JST
end:2023年 10月11日 水曜日 06時51分16秒 JST

MacBook-Pro-5:mysql_study simdongmok$ ls -lh backup.sh_20231011.log
-rw-r--r--  1 simdongmok  staff   108B 10 11 06:51 backup.sh_20231011.log
```

결론 : -d옵션을 이용해서 table의 구조만을 backup하는 방식이 확실히 빠르긴 하지만 백업시 데이터의 복구가 안되기 때문에 이러한 단점을 보안할 방법을 더 생각해 봐야 할것 같습니다.

##복구_TEST

우선 backup한 데이터를 삭제하겠습니다.
```
mysql> use sim
Database changed
mysql> show tables;
+---------------+
| Tables_in_sim |
+---------------+
| person        |
+---------------+
1 row in set (0.02 sec)

mysql> drop table person;
Query OK, 0 rows affected (0.32 sec)

mysql> show tables;
Empty set (0.00 sec)
```

아래 script를 통해 백업을 실행하겠습니다.
```
#!/bin/sh
LOG="./`basename $0`_`date "+%Y%m%d"`.log"
{
    echo "start:"`date`
    mysql -u root -p sim < table_backup.db
    echo "end:"`date`
} 2>&1 | 
while read msg;
do
    echo "$msg" >> ${LOG}
done

///결과
MacBook-Pro-5:mysql_study simdongmok$ cat restore.sh_20231011.log
start:2023年 10月11日 水曜日 19時36分52秒 JST
end:2023年 10月11日 水曜日 19時37分55秒 JST
```
복구후 데이터 확인입니다.
```
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
|  5000000 |
+----------+
1 row in set (1.02 sec)

mysql> 
```
총 복구에 1분3초의 시간이 걸렸습니다.

##결론
5000000개 데이터가 있는 table하나 복구에 1분 3초의 시간이 소요되었습니다. 
더 많은 데이터를 생성해보고 한번더 실행해 볼 필요가 있다고 생각이 들었습니다. 
또한 시간에 중점을 둔 다른 백업 복구 방식에 대해서 조사 할 필요가 있다고 생각합니다.
