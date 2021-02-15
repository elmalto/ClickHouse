---
toc_priority: 36
toc_title: Merge
---

# Merge {#merge}

Движок `Merge` (не путайте с движком `MergeTree`) не хранит данные самостоятельно, а позволяет читать одновременно из произвольного количества других таблиц.
Чтение автоматически распараллеливается. Запись в таблицу не поддерживается. При чтении будут использованы индексы тех таблиц, из которых реально идёт чтение, если они существуют.
Движок `Merge` принимает параметры: имя базы данных и регулярное выражение для таблиц.

Пример:

``` sql
Merge(hits, '^WatchLog')
```

Данные будут читаться из таблиц в базе `hits`, имена которых соответствуют регулярному выражению ‘`^WatchLog`’.

Вместо имени базы данных может использоваться константное выражение, возвращающее строку. Например, `currentDatabase()`.

Регулярные выражения — [re2](https://github.com/google/re2) (поддерживает подмножество PCRE), регистрозависимые.
Смотрите замечание об экранировании в регулярных выражениях в разделе «match».

При выборе таблиц для чтения, сама `Merge`-таблица не будет выбрана, даже если попадает под регулярное выражение, чтобы не возникло циклов.
Впрочем, вы можете создать две `Merge`-таблицы, которые будут пытаться бесконечно читать данные друг друга, но делать этого не нужно.

Типичный способ использования движка `Merge` — работа с большим количеством таблиц типа `TinyLog`, как с одной.

Пример 2:

Пусть есть старая таблица `WatchLog_old`. Необходимо изменить партиционирование без перемещения данных в новую таблицу `WatchLog_new`. При этом в выборке должны участвовать данные обеих таблиц.

``` sql
CREATE TABLE WatchLog_old(date Date, UserId Int64, EventType String, Cnt UInt64)
ENGINE=MergeTree(date, (UserId, EventType), 8192);
INSERT INTO WatchLog_old VALUES ('2018-01-01', 1, 'hit', 3);

CREATE TABLE WatchLog_new(date Date, UserId Int64, EventType String, Cnt UInt64)
ENGINE=MergeTree PARTITION BY date ORDER BY (UserId, EventType) SETTINGS index_granularity=8192;
INSERT INTO WatchLog_new VALUES ('2018-01-02', 2, 'hit', 3);

CREATE TABLE WatchLog as WatchLog_old ENGINE=Merge(currentDatabase(), '^WatchLog');

SELECT *
FROM WatchLog
```

``` text
┌───────date─┬─UserId─┬─EventType─┬─Cnt─┐
│ 2018-01-01 │      1 │ hit       │   3 │
└────────────┴────────┴───────────┴─────┘
┌───────date─┬─UserId─┬─EventType─┬─Cnt─┐
│ 2018-01-02 │      2 │ hit       │   3 │
└────────────┴────────┴───────────┴─────┘
```

## Виртуальные столбцы {#virtualnye-stolbtsy}

-   `_table` — содержит имя таблицы, из которой данные были прочитаны. Тип — [String](../../../engines/table-engines/special/merge.md).

        В секции `WHERE/PREWHERE` можно установить константное условие на столбец `_table` (например, `WHERE _table='xyz'`). В этом случае операции чтения выполняются только для тех таблиц, для которых выполняется условие на значение `_table`, таким образом, столбец `_table` работает как индекс.

**Смотрите также**

-   [Виртуальные столбцы](index.md#table_engines-virtual_columns)

[Оригинальная статья](https://clickhouse.tech/docs/ru/operations/table_engines/merge/) <!--hide-->