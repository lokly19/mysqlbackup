# mysqlbackup

Разворачиваеим виртуалки в Vagrant (master,slave)

```
vagrant up
```

Устнавливаем перкону оба сервера.

```
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
yum install -y Percona-Server-server-57
```

По умолчанию Percona хранит файлы в таком виде:

Основной конфиг в /etc/my.cnf
Так же инклудится директориā /etc/my.cnf.d/ - куда мы и будем складывать наши конфиги
Дата файлы в /var/lib/mysql

# Настраиваем мастер. Подключаемся
```
vagrant ssh master
```

Копируем конфиги из /vagrant/conf.d в /etc/my.cnf.d/

```
cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
```

После этого можно запустить службу

```
systemctl start mysql
```

При установке Percona автоматически генерирует пароль для пользователя root и кладет его в файл /var/log/mysqld.log

```
cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
kq!pIAqfa5JU
```

Подключаемся к mysql и меняем пароль для доступа к полному функционалу

```bash
mysql -uroot -p'kq!pIAqfa5JU'

ALTER USER USER() IDENTIFIED BY 'Lev12345678';
```

Репликацию будем настраивать с использованием GTID. Что это такое и зачем это надо можно почитать [здесь](https://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html).

Следует обратить внимание, что атрибут server-id на мастер-сервере должен обязательно отличаться от server-id слейв-сервера. Проверить какая переменная установлена в текущий момент можно следующим образом

```
SELECT @@server_id;

+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
```

Убеждаемся что GTID включен

```bash
SHOW VARIABLES LIKE 'gtid_mode';

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

Создадим тестовую базу bet и загрузим в нее дамп и проверим

```
CREATE DATABASE bet;

Query OK, 1 row affected (0.00 sec)
```

```
mysql -uroot -p -D bet < /vagrant/bet.dmp

mysql -uroot -p'iGdnT#^7H&Bs'

USE bet;
SHOW TABLES;

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
7 rows in set (0.00 sec)
```

Создадим пользователя для репликации и даем ему права на эту самую репликацию

```bash
CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
SELECT user,host FROM mysql.user where user='repl';

+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
```

Дампим базу для последующего залива на слейв и игнорируем таблицу по заданию

```
mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > /vagrant/master.sql
```

Копируем дамп на slave

```
vagrant plugin install vagrant-scp
vagrant scp master:/vagrant/master.sql 

vagrant scp master.sql slave:/vagrant/
```

На этом настройка Master-а завершена. Файл дампа нужно залить на слейв

Так же точно копируем конфиги из /vagrant/conf.d в /etc/my.cnf.d/

```
cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
```

- Правим в /etc/my.cnf.d/01-base.cnf директиву server-id = 2

```
SELECT @@server_id;

+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)
```

Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки

```bash
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event
```

Таким образом указываем таблицы которые будут игнорироваться при репликации

Заливаем дамп мастера и убеждаемся, что база есть и она без лишних таблиц

```
systemctl start mysql
cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
:mheKBQ(/74r
mysql -uroot -p':mheKBQ(/74r'

ALTER USER USER() IDENTIFIED BY 'Lev12345678';

SOURCE /vagrant/master.sql
SHOW DATABASES LIKE 'bet';

+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0.00 sec)

USE bet;
SHOW TABLES;

+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.00 sec)
```

видим что таблиц v_same_event и events_on_demand нет

Ну и собственно подключаем и запускаем слейв

```
CHANGE MASTER TO MASTER_HOST = "192.168.11.150", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2018", MASTER_AUTO_POSITION = 1;
START SLAVE;
SHOW SLAVE STATUS\G

*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.11.150
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 119864
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 119864
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes           
```

Видно что репликация работает, gtid работает и игнорятся таблички по заданию

```
               Slave_IO_State: Waiting for master to send event
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
```

Проверим репликацю в действии. На мастере

```
USE bet;
INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
SELECT * FROM bookmaker;

+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)
```

На слейве

```bash
SELECT * FROM bookmaker;

+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)
```

