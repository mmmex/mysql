#   MySQL. Backup & Restore.

##  Prepare VM for Demo
vm-otus-mysql
root / otus_1pass
```bash
cd ~/Otus/Linux/MySQL_Backup_Replication/demo
# chmod 400 ub_vks.pem
# ssh -i ub_vks.pem ubuntu@10.10.10.10
# sudo bash
ssh root@10.10.10.10     -- Master
ssh root@10.10.10.11      -- Replica
apt update && apt install -y sudo curl wget lsb-release nano mc pbzip2 && apt-get clean all
```

##  Install Installing Percona Server & XtraBackup 
```bash
cd ~/Otus/Linux/MySQL_Backup_Replication/demo
# chmod 400 otus_key.pub
# cat otus_key.pub
# ssh -i otus_key otus@10.10.10.10
# sudo apt update && sudo apt install -y mysql-server mc && sudo apt clean all

cd /tmp
sudo apt update \
    && sudo curl -O https://repo.percona.com/apt/percona-release_latest.generic_all.deb \
    && sudo apt install -y gnupg2 lsb-release ./percona-release_latest.generic_all.deb \
    && sudo apt update && sudo percona-release setup pdps-8.0 \
    && sudo apt install -y percona-server-server percona-xtrabackup-80 qpress

root / root

export MYSQL_PWD='root'
mysql -e "CREATE FUNCTION fnv1a_64 RETURNS INTEGER SONAME 'libfnv1a_udf.so'"
mysql -e "CREATE FUNCTION fnv_64 RETURNS INTEGER SONAME 'libfnv_udf.so'"
mysql -e "CREATE FUNCTION murmur_hash RETURNS INTEGER SONAME 'libmurmur_udf.so'"
```

##  Config mysql-server
### Securing the MySQL server deployment
```bash
# sudo mysql_secure_installation
```
### Config auth method
```
sudo mysql
SELECT user,authentication_string,plugin,host FROM mysql.user;
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'root';
FLUSH PRIVILEGES;
SELECT user,authentication_string,plugin,host FROM mysql.user;
exit
```

##  Connect MySQL
```bash
export MYSQL_PWD='root'
mysql -hlocalhost -P3306 -uroot -e "show databases;"
```

##  Import Example Databases
(https://dev.mysql.com/doc/index-other.html)

```bash
cd /tmp
curl -O https://downloads.mysql.com/docs/world-db.zip
unzip world-db.zip
less /tmp/world-db/world.sql

# mysql -hlocalhost -P3306 -uroot
# source /tmp/world-db/world.sql;
mysql < /tmp/world-db/world.sql
mysql -e "show databases;use world;show tables;"

mysql world -e "select count(*) from city;select * from city limit 0,10;" -- 4079

```

##  MySQLDump
(https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html)
```bash
sudo mkdir -p /tmp/backups/mysqldump/base
sudo chmod -R 777 /tmp/backups/
cd /tmp/backups/mysqldump
#mysqldump -d world --host=localhost --port=3306 --user=root --password | more
```
### Backup

#### Dump examples
```bash
mysqldump -hlocalhost -P3306 -uroot --databases mysql world > base/backup_base_1.sql
less base/backup_base_1.sql

mysqldump -hlocalhost -P3306 -uroot world > base/backup_base_2.sql
less base/backup_base_2.sql

mysqldump -hlocalhost -P3306 -uroot world city > base/backup_base_3.sql
less base/backup_base_3.sql

mysqldump -hlocalhost -P3306 -uroot -d world city country > base/backup_base_4.sql
less base/backup_base_4.sql

mysqldump -hlocalhost -P3306 -uroot -d world | less //Withuot data
mysqldump -hlocalhost -P3306 -uroot -R -d world | less //Dump stored routines (functions and procedures)

mysqldump -hlocalhost -P3306 -uroot world --tab=base //Delimited-Text Format
mysql -hlocalhost -P3306 -uroot -e "select @@secure_file_priv;"
# SHOW VARIABLES LIKE "secure_file_priv";

sudo su
export MYSQL_PWD='root'
mysqldump -hlocalhost -P3306 -uroot world --tab=/var/lib/mysql-files/ --fields-terminated-by=';'
sudo ls -lh /var/lib/mysql-files/
sudo less /var/lib/mysql-files/city.sql
sudo less /var/lib/mysql-files/city.txt

# Dump только триггеров, процедур и событий
mysqldump -hlocalhost -P3306 -uroot --no-create-info --no-data --triggers --routines --events world > base/backup_base_5.sql

# Dump части таблицы
mysqldump -hlocalhost -P3306 -uroot world city --where="id > 3500" > base/backup_base_6.sql
less base/backup_base_6.sql
```

### Restore

#### Restore Database
```sql
mysql -hlocalhost -P3306 -uroot -e "create database world2; show databases;"
mysql -hlocalhost -P3306 -uroot -e "use world2; show tables;"
mysql -hlocalhost -P3306 -uroot world2 < base/backup_base_2.sql

mysql -hlocalhost -P3306 -uroot -e "select count(*) from world.city;"
mysql -hlocalhost -P3306 -uroot -e "select count(*) from world2.city;"
mysql -hlocalhost -P3306 -uroot -e "use world2; show tables;"

-- mysql> create database if not exists world2;
-- mysql> use world2;
-- mysql> source base/backup_base_2.sql;
```

#### Simple way restore one table
```
sudo mysql world < base/backup_base_3.sql
```
#### Correct way restore one table
```sql
mysql -hlocalhost -P3306 -uroot world -e "show tables;"
mysql -hlocalhost -P3306 -uroot world -e "select count(*) from city;" //4079
mysql -hlocalhost -P3306 -uroot world -e "drop table city; show tables;"

sed -n -e '/DROP TABLE.*`city`/,/UNLOCK TABLES/p' base/backup_base_2.sql > base/table_city.sql
less base/table_city.sql
mysql -hlocalhost -P3306 -uroot world < base/table_city.sql

mysql -hlocalhost -P3306 -uroot world -e "show tables; select count(*) from city;"
```

#### Full backup
(https://dev.mysql.com/doc/refman/8.0/en/backup-policy.html)
--single-transaction - консистентный дамп, Никакие изменения, которые происходят с таблицами InnoDB во время дампа, не будут включены в дамп. Таким образом, дамп - это моментальный снимок баз данных в момент запуска дампа, независимо от того, сколько времени занимает процесс дампа.
--flush-logs ротация bin-логов
```bash
sudo rm -rf /tmp/backups/mysqldump/base/*
ls -lh /tmp/backups/mysqldump/base/

mysqldump -hlocalhost -P3306 -uroot --all-databases --single-transaction \
    > base/backup_base_1.sql
less base/backup_base_1.sql
sudo ls -lh /var/lib/mysql

mysqldump -hlocalhost -P3306 -uroot --all-databases --single-transaction --flush-logs \
    > base/backup_base_2.sql
less base/backup_base_2.sql
sudo ls -lh /var/lib/mysql

mysqldump -hlocalhost -P3306 -uroot --all-databases --single-transaction --flush-logs \
    --delete-source-logs > base/backup_base_3.sql
sudo ls -lh /var/lib/mysql

mysqldump -hlocalhost -P3306 -uroot world --single-transaction \
    > base/backup_base_1.sql
mysqldump -hlocalhost -P3306 -uroot world --single-transaction | gzip \
    > base/backup_base_1.sql.gz
sudo ls -lh base
# gunzip < base/backup_base_1.sql.gz | mysql -hlocalhost -P3306 -uroot world2
# mysql -hlocalhost -P3306 -uroot world2 -e "show tables;"
```

#### MySQLBinlog
```bash
sudo ls -lh /var/lib/mysql
sudo mysqlbinlog -v --base64-output=DECODE-ROWS /var/lib/mysql/binlog.000004
mysql -hlocalhost -P3306 -uroot world -e "select * from city where id = 3665;"
mysql -hlocalhost -P3306 -uroot world -e "update city set name = 'Shahty12345' where id = 3665;"

sudo mysqlbinlog -v --base64-output=DECODE-ROWS /var/lib/mysql/binlog.000004 | less
sudo cp /var/lib/mysql/binlog.000004 base

mysqldump -hlocalhost -P3306 -uroot --all-databases --single-transaction --flush-logs \
    --delete-source-logs > base/backup_base_4.sql

sudo mysqlbinlog -v --base64-output=DECODE-ROWS /var/lib/mysql/binlog.000005 | more

sudo ls -lh /var/lib/mysql
sudo mysqlbinlog -v base/binlog.000004 | more //at 156

mysql -hlocalhost -P3306 -uroot world -e "update city set name = 'Shahty67890' where id = 3665;"
mysql -hlocalhost -P3306 -uroot world -e "select * from city where id = 3665;"

# sudo mysqlbinlog base/binlog.000004 > base/bl_city.sql
# less base/bl_city.sql

sudo mysqlbinlog base/binlog.000004 | mysql -hlocalhost -P3306 -uroot world
mysql -hlocalhost -P3306 -uroot world -e "select * from city where id = 3665;"

# sudo mysqlbinlog base/binlog.000007 \
#     --start-position=235 \
#     --stop-position=497 \
#     --start-datetime="2021-11-01 01:00:00"
#     --stop-datetime="2021-11-10 01:00:00"
#     > base/bl_city.sql

sudo mysql
-- SHOW VARIABLES LIKE "binlog%";
-- binlog_format | ROW
```

##  MySQLPump
mysqlpump

##  xtrabackup
(https://www.percona.com/doc/percona-xtrabackup/8.0/backup_scenarios/full_backup.html)

### Backup
```bash
rm -rf /tmp/backups/xtrabackup
mkdir -p /tmp/backups/xtrabackup/{base,inc1,inc2,partial,stream}
sudo chmod -R 777 /tmp/backups/
cd /tmp/backups/xtrabackup

MYSQL_PWD='root'
# --no-server-version-check
sudo xtrabackup --host=localhost --port=3306 --user=root --password=$MYSQL_PWD \
    --backup --target-dir=/tmp/backups/xtrabackup/base

sudo du -sh /var/lib/mysql
sudo du -sh /tmp/backups/xtrabackup/base

sudo ls -lh /var/lib/mysql
sudo ls -lh /tmp/backups/xtrabackup/base/
sudo more /tmp/backups/xtrabackup/base/xtrabackup_binlog_info
sudo more /tmp/backups/xtrabackup/base/xtrabackup_info
# sudo more /tmp/backups/xtrabackup/base/xtrabackup_logfile
sudo more /tmp/backups/xtrabackup/base/xtrabackup_checkpoints

# select * from city where CountryCode = 'RUS';
mysql -uroot world -e "select * from city where id in (3629,3704);
    update city set name = 'Sochi' where id = 3629;
    update city set name = 'Balashikha' where id = 3704;
    select * from city where id in (3629,3704);"

sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --backup --target-dir=/tmp/backups/xtrabackup/inc1 \
    --incremental-basedir=/tmp/backups/xtrabackup/base

sudo du -sh /tmp/backups/xtrabackup/*
sudo more /tmp/backups/xtrabackup/inc1/xtrabackup_info

mysql -uroot world -e "select * from city where id = 3665;
update city set name = 'Shahty' where id = 3665;
select * from city where id in (3629,3665,3704);"

sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --backup --target-dir=/tmp/backups/xtrabackup/inc2 \
    --incremental-basedir=/tmp/backups/xtrabackup/inc1

sudo du -sh /tmp/backups/xtrabackup/*
sudo more /tmp/backups/xtrabackup/inc2/xtrabackup_info

ls -lh /tmp/backups/xtrabackup/inc1/
ls -lh /tmp/backups/xtrabackup/inc2/
```

### Restore

#### Ooops
```bash
mysql -uroot world -e "show databases; select count(*) from world.city;" //4079

sudo systemctl stop mysql

rm -rf /var/lib/mysql/*
ls -lh /var/lib/mysql
```

<!-- sudo systemctl start mysql
sudo systemctl status mysql -->

#### Prepare
!!! Copy all xtrabackup files !!!
mkdir -p /tmp/backups/xtrabackup2
cp -r /tmp/backups/xtrabackup/* /tmp/backups/xtrabackup2

```bash
sudo ls -lh /tmp/backups/xtrabackup/base
sudo xtrabackup --prepare --apply-log-only --target-dir=/tmp/backups/xtrabackup/base
sudo ls -lh /tmp/backups/xtrabackup/base
sudo xtrabackup --prepare --apply-log-only --target-dir=/tmp/backups/xtrabackup/base \
    --incremental-dir=/tmp/backups/xtrabackup/inc1
sudo ls -lh /tmp/backups/xtrabackup/base
```

#### Restore
```bash
sudo xtrabackup --copy-back --target-dir=/tmp/backups/xtrabackup/base \
    --datadir=/var/lib/mysql

sudo xtrabackup --prepare --target-dir=/tmp/backups/xtrabackup/base

sudo xtrabackup --copy-back --target-dir=/tmp/backups/xtrabackup/base \
    --datadir=/var/lib/mysql

sudo ls -lh /var/lib/mysql

sudo systemctl start mysql
sudo tail -n 10 /var/log/mysql/error.log

sudo chown -R mysql.mysql /var/lib/mysql
sudo ls -lh /var/lib/mysql
sudo systemctl start mysql && sudo systemctl status mysql

mysql -uroot world -e "show databases;
    select count(*) from world.city; 
    select * from world.city where id in (3629,3665,3704);" //4079
```

#### Restore Incremental
```bash
sudo xtrabackup --prepare --apply-log-only --target-dir=/tmp/backups/xtrabackup2/base
sudo xtrabackup --prepare --apply-log-only --target-dir=/tmp/backups/xtrabackup2/base \
    --incremental-dir=/tmp/backups/xtrabackup2/inc1
sudo xtrabackup --prepare --apply-log-only --target-dir=/tmp/backups/xtrabackup2/base \
    --incremental-dir=/tmp/backups/xtrabackup2/inc2
sudo xtrabackup --prepare --target-dir=/tmp/backups/xtrabackup2/base
sudo ls -lh /tmp/backups/xtrabackup2/base

sudo systemctl stop mysql
rm -rf /var/lib/mysql/*
ls -lh /var/lib/mysql

sudo xtrabackup --copy-back --target-dir=/tmp/backups/xtrabackup2/base \
    --datadir=/var/lib/mysql

sudo ls -lh /var/lib/mysql
sudo chown -R mysql.mysql /var/lib/mysql
sudo systemctl start mysql && sudo systemctl status mysql

mysql -uroot world -e "select * from world.city where id in (3629,3665,3704);"
```

#### Partial Backup & Restore

##### Table
```bash
cd /tmp/backups/xtrabackup
sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --backup --datadir=/var/lib/mysql \
    --target-dir=/tmp/backups/xtrabackup/partial --tables="^world.ci.*"
# sudo xtrabackup --backup --tables-file=/tmp/tables.txt
# sudo xtrabackup --backup --databases="mysql world"

sudo ls -lh partial
sudo ls -lh partial/world

MYSQL_PWD='root'
mysqldump -uroot -d world city > partial/city.sql
less partial/city.sql

mysql -uroot world -e "select count(*) from city; 
    drop table city;
    show tables;" //4079


sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --prepare --export --target-dir=/tmp/backups/xtrabackup/partial
sudo ls -lh partial/world

mysql -uroot world < partial/city.sql
mysql -uroot world -e "desc city;
    select count(*) from city;"

mysql -uroot world -e "alter table city discard tablespace;"

sudo cp partial/world/city.ibd /var/lib/mysql/world
sudo chown -R mysql.mysql /var/lib/mysql/world/city.ibd

mysql -uroot world -e "alter table city import tablespace;"
mysql -uroot world -e "select count(*) from city;
    select * from world.city where id in (3629,3665,3704);"
```

##### Single DataBase
```bash
sudo mkdir -p /tmp/backups/world
sudo chmod -R 777 /tmp/backups/world
cd /tmp/backups/
```
###### Backup
```bash
mysqldump -uroot -R -d world > world/backup_world-db.sql

mysql -uroot -N -B <<'EOF' > world/backup_world-db_alter_discard_tables.sql
SELECT 'SET FOREIGN_KEY_CHECKS=0;' UNION SELECT CONCAT('ALTER TABLE ', table_name, ' DISCARD TABLESPACE;') AS _ddl FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='world';
EOF

mysql -uroot -N -B <<'EOF' > world/backup_world-db_alter_import_tables.sql
SELECT CONCAT('ALTER TABLE ', table_name, ' IMPORT TABLESPACE;') AS _ddl FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='world';
EOF

sudo xtrabackup --backup --no-server-version-check \
    --databases="world" \
    --datadir=/var/lib/mysql \
    --target-dir=/tmp/backups/world 

sudo ls -lh world
```
###### Ooops
```bash
sudo mysqladmin drop world
sudo mysql world -e "show databases;"
sudo ls -lh /var/lib/mysql/
```
###### Restore
```bash
sudo xtrabackup --prepare --export --target-dir=/tmp/backups/world

sudo mysqladmin create world
sudo mysql world -e "show databases;"
sudo ls -lh /var/lib/mysql/

sudo mysql world < world/backup_world-db.sql
sudo mysql world -e "show tables;"
sudo mysql world -e "select count(*) from city;"

#ALTER TABLES DISCARD TABLESPACE;
sudo mysql world < world/backup_world-db_alter_discard_tables.sql

sudo cp -a world/world/. /var/lib/mysql/world/
sudo chown -R mysql.mysql /var/lib/mysql/world
sudo ls -lh /var/lib/mysql/world

#ALTER TABLES IMPORT TABLESPACE;
sudo mysql world < world/backup_world-db_alter_import_tables.sql

sudo mysql world -e "show tables;"
sudo mysql world -e "select count(*) from city;"
sudo mysql world -e "select count(*) from country;"
sudo mysql world -e "select count(*) from countrylanguage;"
```

#### Stream into file
```bash
sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --backup --stream=xbstream \
    --target-dir=/tmp/backups/xtrabackup/stream \
    > stream/backup.xbstream

sudo ls -lh stream

sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --backup --stream=xbstream --compress \
    --target-dir=/tmp/backups/xtrabackup/stream \
    > stream/backup_cmprs.xbstream

sudo ls -lh stream

sudo xtrabackup --user=root --password=$MYSQL_PWD \
    --backup --stream=xbstream \
    --target-dir=/tmp/backups/xtrabackup/stream \
    | gzip - | openssl des3 -salt -k "password" \
    > stream/backup_des.xbstream.gz.des3

sudo ls -lh stream

openssl des3 -salt -k "password" -d -in stream/backup_des.xbstream.gz.des3 \
    -out stream/backup_des.xbstream.gz
gzip -d stream/backup_des.xbstream.gz 

cd stream
xbstream -x < backup_des.xbstream
cd ..
sudo ls -lh stream
```

#### Stream on another server
##### 1
```bash
$MYSQL_PWD="password"
ssh root@10.10.10.10 "xtrabackup --host=127.0.0.1 --port=3306 --user=backup --password=$MYSQL_PWD \
    --backup --stream=xbstream" | xbstream -x -C /var/lib/mysql
```

##### 2
```bash
cd ~/Otus/Linux/MySQL_Backup_Replication/demo/
ssh -i ub_vks.pem ubuntu@10.80.1.2
on listener
sudo mkdir /tmp/backups
sudo chmod -R 777 /tmp/backups/
cd /tmp/backups/
ls -lh 
nc -l 9999 | cat - > ./backup.xbstream

# on MySQL Server
sudo xtrabackup --backup --no-server-version-check \
    --stream=xbstream \
    --target-dir=/tmp/backups/xtrabackup/stream \
    | nc 10.80.1.2 9999

# on listener
ls -lh 
xbstream -x < backup.xbstream
```

#   MySQL. Replication.
(https://docs.percona.com/percona-xtrabackup/2.4/howtos/setting_up_replication.html)

## On Master
mysql -e "show variables like '%log_bin%'"
nano /etc/mysql/mysql.conf.d/mysqld.cnf
-------------------------------------------------------------------------------------
server_id                = 1
read_only                = OFF
-------------------------------------------------------------------------------------
mysql -e "SHOW BINLOG EVENTS;"
mysql -e "SET SQL_LOG_BIN=0;"

# mysql -e "DROP USER 'replica'@'%'"
mysql -e "CREATE USER 'replica'@'%' IDENTIFIED WITH mysql_native_password BY 'replica'; GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%'; FLUSH PRIVILEGES;"
mysql -e "SHOW GRANTS FOR 'replica'@'%';"
mysql -e "select * from mysql.user where user = 'replica' \G"
mysql -e "show master status \G";

## On Replica
nano /etc/mysql/mysql.conf.d/mysqld.cnf
-------------------------------------------------------------------------------------
server_id                = 2
read_only                = ON
-------------------------------------------------------------------------------------

```bash
ls -lh /var/lib/mysql
systemctl stop mysql && rm -rf /var/lib/mysql/* && ls -lh /var/lib/mysql

MYSQL_PWD='Otus2022!' && export MYSQL_PWD

# otus_1pass
ssh root@10.10.10.10 "xtrabackup --host=127.0.0.1 --port=3306 --user=root --password=$MYSQL_PWD \
    --backup --stream=xbstream" | xbstream -x -C /var/lib/mysql
xtrabackup --prepare --target-dir=/var/lib/mysql
ls -lh /var/lib/mysql
sudo chown -R mysql.mysql /var/lib/mysql
systemctl start mysql && systemctl status mysql

mysql -e "show replica status \G"

cat /var/lib/mysql/xtrabackup_binlog_info
BL_FILE=$(cat /var/lib/mysql/xtrabackup_binlog_info | awk '{ print $1 }')
BL_POS=$(cat /var/lib/mysql/xtrabackup_binlog_info | awk '{ print $2 }')
echo $BL_FILE $BL_POS

mysql -e "CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='mysqlmaster',
    SOURCE_USER = 'replica',
    SOURCE_PASSWORD = 'Otus2022!',
    SOURCE_LOG_FILE = '$BL_FILE',
    SOURCE_LOG_POS = $BL_POS;"

mysql -e "SHOW REPLICA STATUS \G"
mysql -e "START REPLICA;"
mysql -e "SHOW REPLICA STATUS \G"
```

## On Master
```bash
mysql -e "show databases";
# mysql -e "drop database otus;"
mysql -e "create database otus; use otus; 
    create table test_1 (id int, txt varchar(10)); 
    insert into test_1 values (1, 'txt_1');"
```

## On Replica
```bash
mysql -e "show databases";
mysql -e "select * from world.city where id in (3629,3665,3704);
    select * from otus.test_1";
```

## On Master
```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
-------------------------------------------------------------------------------------
gtid_mode                = ON
enforce_gtid_consistency = ON
-------------------------------------------------------------------------------------
systemctl restart mysql && systemctl status mysql
tail /var/log/mysql/error.log
mysql -e "SHOW MASTER STATUS \G"
mysql -e "insert into otus.test_1 values (2, 'txt_2');"
mysql -e "SHOW MASTER STATUS \G"
```

## On Replica
```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
-------------------------------------------------------------------------------------
gtid_mode                = ON
enforce_gtid_consistency = ON
-------------------------------------------------------------------------------------
systemctl restart mysql && systemctl status mysql
tail /var/log/mysql/error.log
mysql -e "SHOW REPLICA STATUS \G"

mysql -e "STOP REPLICA;"
mysql -e "insert into otus.test_1 values (3, 'txt_3');"     -- On Master
mysql -e "SHOW MASTER STATUS \G"                            -- On Master
mysql -e "show replica hosts;"                              -- On Master
mysql -e "show replicas;"                                   -- On Master

mysql -e "select * from otus.test_1";

mysql -e "CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='10.10.10.10',
    SOURCE_USER = 'replica',
    SOURCE_PASSWORD = 'replica',
    SOURCE_AUTO_POSITION = 1;"

mysql -e "START REPLICA;"
mysql -e "SHOW REPLICA STATUS \G"
mysql -e "select * from otus.test_1";
```

-- Пропустить ошибку репликации без GTID
stop slave; 
set global sql_slave_skip_counter=1; 
start slave;
show slave status \G

-- Пропустить ошибку репликации GTID
# mysql 8.0
SHOW RELAYLOG EVENTS;
SHOW BINLOG EVENTS;
SHOW BINLOG EVENTS IN 'my3306_binlog.000052';

SHOW REPLICA STATUS \G; 
select * from performance_schema.replication_applier_status_by_worker \G; -- d71944c0-a563-11ec-a9fa-fa163ea1b0d3:1-27
STOP REPLICA;
SET GTID_NEXT='d71944c0-a563-11ec-a9fa-fa163ea1b0d3:1-27';
BEGIN;COMMIT;
SET GTID_NEXT='AUTOMATIC';
START REPLICA;

# mysql 5.7
select * from performance_schema.replication_applier_status_by_worker \G; -- 06273d84-48bf-11ed-b13d-fa163e159df2:1
SHOW SLAVE STATUS \G;
STOP SLAVE;
SET GTID_NEXT='06273d84-48bf-11ed-b13d-fa163e159df2:1';
BEGIN;COMMIT;
SET GTID_NEXT='AUTOMATIC';
START SLAVE;