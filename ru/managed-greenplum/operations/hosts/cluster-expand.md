# Расширение кластера

Вы можете добавить хосты-сегменты в кластер {{ mgp-name }}. Данные перераспределяются между существующими и добавленными сегментами. Число добавляемых сегментов не может быть меньше двух.

Перераспределение данных бывает двух типов:

* Автоматическое — после обновления кластера выполняется перенос части данных в новые сегменты последовательно для каждой таблицы. Во время переноса таблица недоступна для операций чтения и записи.
* Ручное — выполняется после добавления новых сегментов в течение таймаута, указанного в настройках. После завершения таймаута прекращается.

## Добавить хосты-сегменты {#add-hosts}

{% list tabs %}


- API

    Воспользуйтесь методом API [expand](../../api-ref/Cluster/expand.md) и передайте в запросе:

    * Идентификатор кластера в параметре `clusterId`.
    * Число добавляемых хостов-сегментов в параметре `segment_host_count`.
    * Число сегментов на хост в параметре `add_segments_per_host_count`.
    * Таймаут перераспределения данных (в секундах) в параметре `duration`. Минимальное значение и значение по умолчанию — `0` (не выполнять перераспределение).

    Идентификатор кластера можно получить со [списком кластеров в каталоге](../cluster-list.md#list-clusters).

{% endlist %}

{% note warning %}

Минимальное или малое значение таймаута перераспределения может снизить производительность кластера. В этом случае выполните повторный запуск перераспределения вручную (см. [Пример](#redistribute-by-hand)).

{% endnote %}

## Мониторинг перераспределения данных {#redistribute-monitoring}

Чтобы следить за ходом перераспределения данных по новым сегментам, подключитесь к базе `postgres` и выполните запрос от имени пользователя с ролью `mdb_admin`:

```sql
SELECT dbname, fq_name, status, expansion_started, source_bytes FROM gpexpand.status_detail;
```

```text
  dbname   |               fq_name               |   status    |     expansion_started      | source_bytes
-----------+-------------------------------------+-------------+----------------------------+-------------
 diskquota | diskquota_namespace.database_list   | NOT STARTED |                            |            0
 postgres  | public.rnd_nocomp_distrnd_ao_res3   | NOT STARTED |                            |  52558742480
 postgres  | public.rnd_nocomp_distrnd_ao1       | COMPLETED   | 2022-09-06 12:44:36.71759  |     13013536
 postgres  | public.rnd_nocomp_distrnd_ao_res2   | IN PROGRESS | 2022-09-06 13:03:29.231359 |  63070490912
(4 rows)
```

Текущий статус перераспределения будет указан в колонке `status`.

## Пример ручного перераспределения данных {#redistribute-by-hand}

Если при расширении кластера {{ GP }} явно указан таймаут перераспределения, то необходимо выполнить его вручную.

* Для обычных таблиц выполните запрос:

  ```sql
  ALTER TABLE ONLY <имя_таблицы> EXPAND TABLE;
  ```

* Для партиционированных таблиц выполните запрос:

  ```sql
  ALTER TABLE <имя_таблицы> SET WITH (REORGANIZE=true) <distribution_policy>;
  ```

  где `<distribution_policy>` – это строка, обозначающая политику распределения {{ GP }} для партиции выбранной таблицы. Ее можно получить в результате вызова встроенной функции {{ GP }}:
  
  ```sql
  SELECT pg_get_table_distributedby(<OID партиции>) AS distribution_policy;
  ```

Найти таблицы, которые перераспределились не полностью, можно с помощью запроса:

```sql
SELECT * FROM gp_toolkit.gp_skew_coefficients ORDER BY skccoeff DESC;
```

{% include [greenplum-trademark](../../../_includes/mdb/mgp/trademark.md) %}
