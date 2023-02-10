## MySQL

Задачи:

В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp
- [X] Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:
   - bookmaker
   - competition
   - market
   - odds
   - outcome

- [X] Настроить GTID репликацию

Варианты которые принимаются к сдаче:
- [X] [рабочий вагрантафайл](/Vagrantfile)
- [X] [скрины или логи SHOW TABLES](#проверка)
- [X] [конфиги](#листинг-файлов-конфигурации)
- [X] [пример в логе изменения строки и появления строки на реплике](#проверка)

---

### Запуск проекта

1. Клонируем репозиторий: `git clone https://github.com/mmmex/mysql.git`
2. Выполним вход в папку: `cd mysql`
3. Запускаем проект: `vagrant up`

Будет поднят кластер mysql v8.0.31-23 (Percona Server) в составе двух узлов (master, slave). Так же будут импортированы таблицы из [дампа предоставленного из ТЗ](ansible/files/bet-195395-623055.dmp) в БД `bet`.

 Хост       | IP-адрес       | Роль
------------|----------------|-------------------------
mysqlmaster | 192.168.100.11 | mysql-master
mysqlslave  | 192.168.100.12 | mysql-slave with filters

### Проверка

1. Выполним вход на узел mysqlmaster: `vagrant ssh mysqlmaster`
2. Поднимаем привелегии: `sudo -i`
3. Смотрим таблицы БД `bet` на master: `mysql -e "use bet; show tables;"`
4. Смотрим таблицы БД `bet` на slave: `ssh mysqlslave 'mysql -e "use bet; show tables;"'`
5. Можно посмотреть разницу между БД `bet` на mysqlmaster и mysqlslave: `diff -y <(mysql -e "use bet; show tables;") <(ssh mysqlslave 'mysql -e "use bet; show tables;"')`

```bash
test@test-virtual-machine:~/Otus/mysql$ vagrant ssh mysqlmaster
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat 11 Feb 2023 12:06:37 AM MSK

  System load:  0.0                Processes:             116
  Usage of /:   15.9% of 30.34GB   Users logged in:       0
  Memory usage: 54%                IPv4 address for eth0: 10.0.2.15
  Swap usage:   17%                IPv4 address for eth1: 192.168.100.11


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Fri Feb 10 23:28:33 2023 from 10.0.2.2
vagrant@mysqlmaster:~$ sudo -i
root@mysqlmaster:~# mysql -e "use bet; show tables;"
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
root@mysqlmaster:~# ssh mysqlslave 'mysql -e "use bet; show tables;"'
Tables_in_bet
bookmaker
competition
market
odds
outcome
root@mysqlmaster:~# diff -y <(mysql -e "use bet; show tables;") <(ssh mysqlslave 'mysql -e "use bet; show tables;"')
Tables_in_bet                                                   Tables_in_bet
bookmaker                                                       bookmaker
competition                                                     competition
events_on_demand                                              <
market                                                          market
odds                                                            odds
outcome                                                         outcome
v_same_event                                                  <
```

В последнем выводе результата командой diff видно таблицы БД `bet`: справа slave, слева master.

Также проверим работу репликации явно:

1. Выполним команду на master: `mysql -e "create database otus; use otus; create table test_1 (id int, txt varchar(10)); insert into test_1 values (1, 'txt_1');"`
2. Проверим на slave: `ssh mysqlslave 'mysql -e "show databases; use otus; show tables; select * from otus.test_1;"'`

```bash
root@mysqlmaster:~# ssh mysqlslave 'mysql -e "show databases; use otus; show tables; select * from otus.test_1;"'
Database
bet
information_schema
mysql
otus
performance_schema
sys
Tables_in_otus
test_1
id      txt
1       txt_1
```

### Листинг файлов конфигурации

##### master: /etc/mysql/mysql.conf.d/mysqld.cnf

```bash
root@mysqlmaster:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf
#
# The Percona Server 8.0 configuration file.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
log-error       = /var/log/mysql/error.log
server_id = 1
read_only = OFF
gtid_mode = ON
enforce_gtid_consistency = ON
```

##### slave: /etc/mysql/mysql.conf.d/mysqld.cnf

```bash
root@mysqlslave:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf
#
# The Percona Server 8.0 configuration file.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
log-error       = /var/log/mysql/error.log
server_id = 2
read_only = ON
gtid_mode = ON
enforce_gtid_consistency = ON
```

### GTID репликация

Глобальный идентификатор транзакции (GTID) — это уникальный идентификатор, созданный и связанный с каждой транзакцией, совершенной на исходном сервере (источнике). Этот идентификатор уникален не только для сервера, на котором он был создан, но и для всех серверов в данной топологии репликации.

Назначение GTID различает клиентские транзакции, которые фиксируются в источнике, и реплицированные транзакции, которые воспроизводятся в реплике. Когда клиентская транзакция фиксируется в источнике, ей присваивается новый GTID при условии, что транзакция была записана в двоичный журнал. Клиентские транзакции гарантированно имеют монотонно возрастающие GTID без пробелов между сгенерированными числами. Если клиентская транзакция не записывается в двоичный журнал (например, из-за того, что транзакция была отфильтрована или транзакция была доступна только для чтения), ей не присваивается GTID на исходном сервере.

Реплицированные транзакции сохраняют тот же GTID, который был назначен транзакции на исходном сервере. GTID присутствует до того, как реплицированная транзакция начнет выполняться, и сохраняется, даже если реплицированная транзакция не записывается в двоичный журнал реплики или отфильтровывается на реплике. Системная таблица MySQL mysql.gtid_executed используется для сохранения назначенных GTID всех транзакций, применяемых на сервере MySQL, за исключением тех, которые хранятся в активном в данный момент двоичном файле журнала.

Функция автоматического пропуска для GTID означает, что транзакция, зафиксированная в источнике, может быть применена к реплике не более одного раза, что помогает гарантировать согласованность. Как только транзакция с данным GTID зафиксирована на данном сервере, любая попытка выполнить последующую транзакцию с тем же GTID игнорируется этим сервером. Никакой ошибки не возникает, и ни один оператор в транзакции не выполняется.

Если транзакция с заданным GTID начала выполняться на сервере, но еще не зафиксирована или не откатана, любая попытка запустить параллельную транзакцию на сервере с таким же GTID блокируется. Сервер не начинает выполнять параллельную транзакцию и не возвращает управление клиенту. После фиксации или отката первой попытки транзакции могут быть продолжены параллельные сеансы, которые были заблокированы для одного и того же GTID. Если первая попытка откатывается, один параллельный сеанс переходит к попытке транзакции, а любые другие параллельные сеансы, которые были заблокированы на том же самом GTID, остаются заблокированными. Если первая попытка совершена, все параллельные сеансы перестают блокироваться и автоматически пропускаются все операторы транзакции.

[_Примерный перевод официального руководства_](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-concepts.html)

###### Полезные ссылки

https://dev.mysql.com/doc/refman/8.0/en/password-security-user.html

https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-concepts.html

https://dev.mysql.com/doc/refman/5.7/en/change-replication-filter.html

https://github.com/dincho/ansible-percona-server