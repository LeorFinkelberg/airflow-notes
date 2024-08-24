С помощью `BashOperator` (и всех других операторов Airflow) вы предоставляете строку аргументу `bash_command` (или любому другому аргументу в других операторах), который _автоматически шаблонизируется во время выполнения_. ==`PythonOperator` является исключением из этого стандарта, потому что он не принимает аргументы, которые можно шаблонизировать, используя контекст среды выполнения==. Вместо этого он принимает аргумент `python_callable`, в котором можно применить данные контекст [[Список литературы#^ae6dac]]<c. 97>. Вместо этого он принимает аргумент `python_callable`, в котором можно применить данный контекст.

Функции -- это объекты первого класса в Python, и предоставляем _вызываемый объект_ (функция -- это вызываемый объект) аргументу `python_callable` оператора `PythonOperator`. При выполнении `PythonOperator` выполняет предоставленный вызываемый объект, которым может быть любая функция. ==Поскольку эту функция, а не строка, как у всех других операторов, код внутри функции нельзя шаблонизировать автоматически==. Вместо этого в данной функции можно указать и использовать переменные контекста задачи.

Чтобы сообщить себе в будущем и другим специалистам, которые будут читать код, о своих намерениях собирать переменные контекста задачи в ключевых аргументах, рекомендуется присвоить этому аргументу соответствующее имя (например, `context`)
```python
def _print_context(**context):
    print(context)

print_context = PythonOperator(
	task_id = "print_context",
	python_callable=_print_context,
	dag=dag,
)
```
Контекстная переменная -- это словарь всех переменных контекста, что позволяет нам задавать задаче различное поведение для интервала, в котором она выполняется, например для вывода даты и времени начала и окончания текущего интервала
```python
def _print_context(**context):
    start = context["execution_date"]
    end = context["next_execution_date"]
    print(f"Start: {start}, end: {end}")

print_context = PythonOperator(
	task_id="print_context",
	python_callable=_print_context,
	dag=dag,
)
```

Оператор `PythonOperator`, скачивающий ежечасные просмотры страниц "Википедии"
```python
def _get_data(**context):
    year, month, day, hour, *_ = context["execution_date"].timetuple()
    url = (
        "https://dumps.wikimedia.org/other/pageviews/"
        f"{year}/{year}-{month:0>2}/pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz"
    )
    output_path = "/tmp/wikipageviews.gz"
    request.urlretrieve(url, output_path)  # получаем данные
```

В Python есть еще один способ принимать ключевые аргументы
```python
# Здесь мы сообщаем, что ожидаем получить аргумент `execution_date`;
# Он не будет собран в аргумент `context`
def _get_data(
	execution_date,  # явно указываем
	**context
):
    year, month, day, hour, *_ = execution_date.timetuple()
    ...
```
Конечно результат этого примера -- ключевое слово `execution_date` передается аргументу `execution_date`, а все другие переменные передаются `**context`, поскольку они явно не ожидаются в сигнатуре функции.

Теперь мы можем напрямую использовать переменную `execution_date`, вместо того чтобы извлекать ее из `**context` с помощью `context["execution_date"]`.

Пример
```python
def _get_data(output_path, **context):
    year, month, day, hour, *_ = context["execution_date"].timetuple()
    url = (
        "https://dumps.wikimedia.org/other/pageviews/"
        f"{year}/{year}-{month:0>2}/pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz"
    )
    output_path = "/tmp/wikipageviews.gz"
    request.urlretrieve(url, output_path)
```

Значение `output_path` можно указать двумя способами:
- через аргумент `op_args`,
- через аргумент `op_kwargs`.

То есть либо
```python
get_data = PythonOperator(
	task_id="get_data",
	python_callable=_get_data,
	op_args=["/tmp/wikipageviews.gz"],
	dag=dag,
)
```
При выполнении оператора каждое значение в списке, предоставленное аргументу `op_args`, передается в вызываемую функцию, то есть: `_get_data("/tmp/wikipageviews.gz")`.

Либо так
```python
get_data = PythonOperator(
	task_id="get_data",
	python_callable=_get_data,
	op_kwargs={"output_path": "/tmp/wikipageviews.gz"},
	dag=dag,
)
```
Эти значения могут содержать строки, а значит _их можно шаблонизировать_! Это означает, что мы могли бы избежать извлечения компонентов `datetime` внутри самой вызываемой функции и вместо этого передать шаблонные строки вызываемой функции
```python
def _get_data(year, month, day, hour, output_path, **_):
    url = (
        "https://dumps.wikimedia.org/other/pageviews/"
        f"{year}/{year}-{month:0>2}/"
        f"pageviews-{year}{month:0>2}{day:0>2}-{hour:0>2}0000.gz"
    )
    request.urlretrieve(url, output_path)

get_data = PythonOperator(
	task_id="get_data",
    python_callable=_get_data,
    op_kwargs={
        "year": "{{ execution_date.year }}",
        "month": "{{ execution_date.month }}",
        "day": "{{ execution_date.day }}",
        "hour": "{{ execution_date.hour }}",
        "output_path": "/tmp/wikipageviews.gz",
    },
    dag=dag,
)
```
