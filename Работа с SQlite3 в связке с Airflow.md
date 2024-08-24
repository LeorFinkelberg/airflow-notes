После инициализации базы метаданных с помощью команды `airflow db init`, будет создан файл базы данных `airflow.db` по пути `~/airflow/airflow.db`.

Содержимое базы данных можно изучить, например, с помощью Python-сессии
```python
import sqlite3

con = sqlite3.connect("./airflow.db")
cur = con.cursor()
```
Вывести список таблиц можно так
```python
import typing as t

SqliteMasterTable = t.Tuple[str, str, str, int, str]

sqlite_master_tables: t.List[SqliteMasterTable] = cur.execute(
	"select * from sqlite_master"
    "where type = 'table';"
).fetchall()
```
Получим список имен таблиц
```python
table_names: t.List[str] = [table[1] for table in sqlite_master_tables]
```
Получить описание схемы таблицы можно так
```python
cur.execution("PRAGMA table_info('dag')").fetchall()
# Output
[(0, 'dag_id', 'VARCHAR(250)', 1, None, 1),
 ...
 (27, 'next_dagrun_create_after', 'TIMESTAMP', 0, None, 0)]
```