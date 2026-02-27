# Домашнее задание к занятию "`Репликация и масштабирование. Часть 1`" - `Дёмин Максим`


### Задание 1

Что нужно сделать:

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

### Решение:

## Master-Slave (Source-Replica)

Один сервер (master) принимает все записи (INSERT/UPDATE/DELETE) и пишет изменения в binary log.

Второй сервер (slave) читает binary log мастера и повторяет изменения у себя.

Обычно на slave включают read_only, и используют его для чтения (отчёты, витрины, разгрузка мастера).

Плюсы: проще настраивать и поддерживать, почти нет конфликтов.

Минусы: возможна задержка репликации (данные на slave появляются позже), при падении master нужен failover (переключение).

## Master-Master (Active-Active)

Два сервера одновременно: каждый и master, и slave (репликация идёт в обе стороны).

Запись можно делать на оба, но появляется риск конфликтов:

одинаковые ключи (особенно AUTO_INCREMENT),

одновременные изменения одной строки на обоих серверах.

Чтобы уменьшить риск конфликтов автоинкремента, обычно задают:

auto_increment_increment=2

auto_increment_offset=1 на первом и =2 на втором.

Плюсы: можно строить схемы высокой доступности, двусторонняя синхронизация.

Минусы: сложнее, риск конфликтов, нужны правила “куда пишем”.

### Задание 2

Что нужно сделать:

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

### Решение:

Установка MySQL

```
sudo apt update
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
```
Настройка MASTER

/etc/mysql/mysql.conf.d/mysqld.cnf
```
[mysqld]
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_format           = ROW
binlog_row_image        = FULL

gtid_mode               = ON
enforce_gtid_consistency= ON
```
Перезапуск:
```
sudo systemctl restart mysql
sudo systemctl status mysql --no-pager
```
Пользователь для репликации на MASTER
```
sudo mysql

CREATE USER 'repl'@'192.168.88.%' IDENTIFIED BY 'RPass_123';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.88.%';
FLUSH PRIVILEGES;

SHOW GRANTS FOR 'repl'@'192.168.88.%';
EXIT;
```
Создадим тестовую БД demo
```
sudo mysql -e "
CREATE DATABASE IF NOT EXISTS demo;
USE demo;
CREATE TABLE IF NOT EXISTS messages (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  txt VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO messages(txt) VALUES ('hello from master');
SELECT * FROM messages;
"
```
Делаем дамп:
```
mysqldump --single-transaction --set-gtid-purged=OFF demo > /tmp/demo.sql
ls -lh /tmp/demo.sql
```
Копируем на SLAVE
```
scp /tmp/demo.sql demin@192.162.88.107:/tmp/demo.sql
```
Настройка SLAVE
```
[mysqld]
server-id               = 2
relay_log               = /var/log/mysql/mysql-relay-bin.log
read_only               = ON
super_read_only         = ON

gtid_mode               = ON
enforce_gtid_consistency= ON
```
Перезапуск:
```
sudo systemctl restart mysql
sudo systemctl status mysql --no-pager
```
Импорт дампа на SLAVE
```
sudo mysql < /tmp/demo.sql
sudo mysql -e "SELECT * FROM demo.messages;"
```
Подключение репликации на SLAVE
```
sudo mysql

STOP REPLICA;
RESET REPLICA ALL;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.88.106',
  SOURCE_PORT=3306,
  SOURCE_USER='repl',
  SOURCE_PASSWORD='RPass_123',
  SOURCE_AUTO_POSITION=1;

START REPLICA;

SHOW REPLICA STATUS\G
EXIT;
```
Проверка master-slave
На MASTER:
```
sudo mysql -e "INSERT INTO demo.messages(txt) VALUES ('row2 from master'); SELECT * FROM demo.messages;"
```
На SLAVE:
```
sudo mysql -e "SELECT * FROM demo.messages;"
```



### Задание 3

Что нужно сделать:

Выполните конфигурацию master-master репликации. Произведите проверку.

### Решение:

На обоих серверах включаем
```
[mysqld]
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_format           = ROW

gtid_mode               = ON
enforce_gtid_consistency= ON

log_replica_updates      = ON

auto_increment_increment = 2
auto_increment_offset  = 1
# auto_increment_offset  = 2 на втором сервере
```
+++++++++

<img width="1920" height="525" alt="image_2026-02-27_17-45-58" src="https://github.com/user-attachments/assets/41e61f7a-7598-4fb8-a5d6-49e75b1c98f4" />

+++++++++

<img width="1920" height="730" alt="image_2026-02-27_18-05-12" src="https://github.com/user-attachments/assets/34bc7500-0206-4745-a157-440e0af3124a" />

+++++++++

Перезапуск:
```
sudo systemctl restart mysql
```
Пользователь репликации на A и B
На A:
```
CREATE USER 'repl'@'192.168.88.%' IDENTIFIED BY 'RPass_123';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.88.%;
FLUSH PRIVILEGES;
```
+++++++++

<img width="1920" height="214" alt="image_2026-02-27_18-06-26" src="https://github.com/user-attachments/assets/b179f896-25aa-40b7-b84b-7c9f0602eb3e" />

+++++++++

На B — то же самое.

Репликация в обе стороны
На A настраиваем источник = B:
```
STOP REPLICA;
RESET REPLICA ALL;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.88.107',
  SOURCE_PORT=3306,
  SOURCE_USER='repl',
  SOURCE_PASSWORD='RPass_123',
  SOURCE_AUTO_POSITION=1;

START REPLICA;
SHOW REPLICA STATUS\G
```
На B настраиваем источник = A:
```

STOP REPLICA;
RESET REPLICA ALL;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.88.106',
  SOURCE_PORT=3306,
  SOURCE_USER='repl',
  SOURCE_PASSWORD='RPass_123',
  SOURCE_AUTO_POSITION=1;

START REPLICA;
SHOW REPLICA STATUS\G
```
+++++++++

<img width="1904" height="1119" alt="image_2026-02-27_18-34-14" src="https://github.com/user-attachments/assets/3288a9b0-3ce5-4724-aa99-9f3670d69e09" />

+++++++++

Проверка master-master
Тест A → B:
```
# На A:
sudo mysql -e "CREATE DATABASE IF NOT EXISTS mm; USE mm;
CREATE TABLE IF NOT EXISTS t (id BIGINT PRIMARY KEY AUTO_INCREMENT, v VARCHAR(50));
INSERT INTO t(v) VALUES ('from A');
SELECT * FROM t;"
￼
# На B:
sudo mysql -e "SELECT * FROM mm.t;"
```
+++++++++

<img width="1904" height="198" alt="image_2026-02-27_18-35-02" src="https://github.com/user-attachments/assets/d5097a2c-6dfb-43a2-906c-c7a5284130a5" />

+++++++++

Тест B → A:
```￼
# На B:
sudo mysql -e "INSERT INTO mm.t(v) VALUES ('from B'); SELECT * FROM mm.t;"
￼
# На A:
sudo mysql -e "SELECT * FROM mm.t;"
```
+++++++++

<img width="1904" height="198" alt="image_2026-02-27_18-35-45" src="https://github.com/user-attachments/assets/952a713f-eaa9-4262-a350-06f9daad9140" />

+++++++++
