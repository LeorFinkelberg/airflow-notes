Создать локальную базу данных Postgres для тестирования несложно с помощью Docker. Сначала нужно установить `pytest-docker-tools` в своем окружении с помощью команды 
```bash
$ pip install pytest_docker_tools
```

Извлечем контейнер
```python
from pytest_docker_tools import fetch

postgres_image = fecth(repository="postgres:11.1-alpine")
```

Функция `fetch` запускает команду `docker_pull` на компьютере, на ведется работа (и, следовательно, требуется, чтобы вы установили на нем Docker), и возвращает скаченный образ. Сама функция представляет собой _фикстуру_ pytest, а это значит, что ее нельзя вызывать напрямую. Мы должны предоставить ее в качестве параметра для теста
```python
from pytest_docker_tools import fetch

postgres_image = fetch(repository="postgres:11.1-alpine")

# postgres_image - это фикстура
def test_call_fixture(postgres_image):
    print(postgres_image.id)
```

Теперь можно использовать этот идентификатор образа для настройки и запуска контейнера Postgres [[Список литературы#^ae6dac]]<c. 249>
```python
from pytest_docker_tools import container

postgres_container = container(
	image="{postgres_image.id"}
    ports={"5432/tcp": None},
)

# postgres_container - это фикстура
def test_call_fixture(postgres_container):
    print(
        f"Running Postgres container named {postgres_container.name} "
        f"on port {postgres_container.ports['5432/tcp'][0]}."
    )
```

Обычно порт контейнера отображается в тот же порт в хост-системе (например, `docker run -p 5432:5432 posrgres`). Контейнер для тестов не должен работать до бесконечности, и, кроме того, мы не хотим конфликтовать с другими портами, используемыми в хост-системе.

Предоставляя словарь ключевому аргументу `ports`, где ключи -- это порты контейнера, а значения отображаются в хост-систему, и оставляя значения `None`, мы отображаем порт хоста в случайный открытый порт на хосте (как при запуске `docker run -P`). Если предоставить фикстуру, то это приведет к ее выполнению (то есть запуску контейнера), а pytest-docker-tools затем внутренне отобразит назначенные порты в хост-системе в атрибут `ports` в самой фикстуре. `postgres_container.ports['5432/tcp'][0]` дает нам назначенный номер порта на хосте, который мы затем можем использовать в тесте для подключения.
```python
from pytest_docker_tools import container, fetch

postgres_image = fetch(repository="postgres:11.1-alpine")
postgres = container(
	image="{postgres_image.id"}
    environment={
        "POSTGRES_USER": "testuser",
        "POSTGRES_PASSWORD": "testpass",
    },
    ports={"5432/tcp": None},
    volumes={
        os.path.join(os.path.dirname(__file__), "posrgres-init.sql"): {
            "bind": "/docker-entrypoint-initdb.d/posrgres-init.sql"
        }
    }
)
```

Структуру базы данных и данные можно инициализировать в `posrgres-init.sql`
```sql
-- postgres-init.sql
SET SCHEMA 'public';
CREATE TABLE movielens(
  movieId integer,
  rating float,
  ratingTimestamp integer,
  userId integer,
  scrapeTime timestamp
)
```

В фикстуре `container` мы предоставляем имя пользователя и пароль Postgres через переменные окружения. Еще одна функция образа docker -- возможно инициализировать контейнер сценарием запуска, поместив файл с расширением `*.sql`, `*.sql.gz` или `*.sh` в каталог `/docker-entrypoint-initdb.d`. Они выполняются при загрузке контейнера, перед запуском фактического сервиса Postgres, и их можно использовать для инициализации нашего тестового контейнера с таблицей для запроса.

Файл `postgres-init.sql` монтируется в контейнер, используя ключевое слово `volume`
```python
volumes = {
	os.path.join(os.path.dirname(__file__), "postgres-init.sql"): {
         "bind": "/docker-entrypoint-initdb.d/postgres-init.sql"
	}
}
```

Мы предоставляем ему словарь, где ключи показывают (абсолютное) местоположение в хост-системе. В данном случае мы сохранили файл `postgres-init.sql` в том же каталоге, что и наш тестовый сценарий, поэтому `os.path.join(os.path.dirname(__file__), "postgres-init.sql")` даст нам абсолютный путь к нему. Значения также являются словарем, где ключ указывает тип монтирования (bind) и значение местоположения внутри контейнера, `/docker-entrypoint-initdb.d` для запуска сценария с расширением `*.sql` во время загрузки контейнера.