## Принципы
Основным средством выражения команд к YDB является декларативный язык запросов [YQL](/yql/). YQL является диалектом SQL, который считается стандартом де факто для общения с базами данных. Поддержка YQL предлагает пользователю мощный и в то же время привычный способ взаимодействия с YDB.
Помимо YQL YDB поддерживает набор специфичных RPC, например, для работы с древовидной схемой, или для управления кластером.

## Уровни изоляции транзакций
Базам данных свойственно поддерживать разные уровни [изоляции транзакций](https://en.wikipedia.org/wiki/Isolation_(database_systems)). Ядро YDB реализует самый строгий уровень изоляции называемый serializable. Это означает, что параллельно выполняющиеся транзакции,которые были закоммичены, не подвержены различным аномалиям, а эффект от параллельного выполнения транзакций такой, какой бы был эффект при определенном (не говорим каком) порядке выполнения этих транзакций.
Тем не менее, пользователь имеет возможность ослабить уровень изоляции, выставив значения ReadCommitted, ReadUncommitted и ReadStale при помощи соответствующей [pragma](../yql/syntax/pragma.md).

## Язык YQL

Реализованные конструкции YQL можно разделить на два класса [DDL](https://en.wikipedia.org/wiki/Data_definition_language) и [DML](https://en.wikipedia.org/wiki/Data_manipulation_language).

К DDL относятся такие конструкции как CREATE/ALTER/DROP TABLE.

К DML относятся конструкции UPDATE/INSERT/DELETE/SELECT и другие.

Подробнее про поддерживаемые конструкции YQL можно почитать [здесь](/yql/).

Ниже перечислены возможности и ограничения поддержки YQL в YDB, которые могут быть неочевидны на первый взгляд и на которые стоит обратить внимание.

 * Допускаются multistatement transactions, то есть транзакции состоящие из последовательности стейтментов YQL, которые заканчиваются COMMIT. При этом при выполнении транзакции допускается взаимодействие с клиентской программой, иначе говоря взаимодействие клиента с базой может выглядеть следующим образом: `НАЧАТЬ ТРАНЗАКЦИЮ; выполнить SELECT; проанализировать результаты SELECT на клиенте; ...; выполнить UPDATE; COMMIT`. Данный паттерн взаимодействия с базой данных является стандартным для традиционных реляционных БД, мы лишь подчеркиваем, что работу с YDB можно организовать таким же образом. Стоит отметить, что если есть возможность полностью сформировать тело транзакции до обращения к базе данных, то такой запрос может обрабатываться эффективнее.
 * В YDB не поддерживается возможность смешивать DDL и DML запросы в одной транзакции. Традиционное понятие [ACID](https://ru.wikipedia.org/wiki/ACID) транзакции применимо именно к DML запросам, то есть к запросам, которые меняют данные. DDL запросы должны быть идемпотентными, то есть повторяемы в случае ошибки. Если необходимо выполнить действие со схемой, то каждое из действий будет транзакционно, а набор действий нет.
 * Реализация YQL в YDB использует механизм [Optimistic Concurrency Control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control). На затронутые в ходе транзакции сущности ставятся оптимистичные блокировки, при коммите транзакции проверяется, что блокировки не были инвалидированы. Оптимистичность блокировок выливается в важное для пользователя свойство -- более поздняя транзакция выигрывает у более ранней транзакции в случае конфликта. Используемый в YDB механизм реализации транзакций позволяет обеспечить уровень изоляции serializable (при условии, что уровень изоляции не понижен пользователем намеренно).
 * Важно отметить, что текущая реализация поддержки YQL имеет некоторые ограничения. Дело в том, что все изменения, производимые в рамках транзакции, накапливаются в памяти сервера базы данных и применяются в момент коммита транзакции. Если взятые блокировки не были инвалидированы, то все накопленные изменения применяются атомарно, если хотя бы одна блокировка была инвалидирована, то ни одно из изменений не будет применено. Описанная схема диктует некоторые ограничения:
  * Объем изменений осуществляемых в рамках одной транзакции не может быть очень большим, так как должен умещаться в память.
  * В настоящее время, если транзакция выполняет некоторое чтение, то это чтение выполняется из головы, т.е. из текущего хранимого (и зафиксированного) состояния базы данных.
    * Это означает, что чтению вообще говоря нельзя доверять до момента коммита (в этот момент будут проверены локи).
    * Это означает, что транзакция, вообще говоря, не видит своих изменений. Текущая реализация обнаруживает такие чтения после изменений и выдает ошибку, поэтому уровень сериализации транзакций не уменьшается, однако приводит к откату транзакций. Для того, чтобы желаемое пользователем изменение таки было выполнено, рекомендуется формировать транзакцию таким образом, чтобы в первой части транзакции выполнялись только чтения, а во второй части транзакции только необходимые модификации. Структура запросы тогда выглядит следующим образом:
```sql
SELECT ...;
....
SELECT ...;
UPDATE/INSERT/DELETE ...;
COMMIT;
```
Подробнее про поддержку YQL в YDB можно прочитать в соответствующем [разделе](/yql/).


## Распределенные транзакции


## OLAP vs OLTP
В зависимости от пользовательских сценариев паттерны нагрузки на базу данных могут существенно отличаться. Принято выделять два паттерна -- [OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing) и [OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing). OLTP характеризуется большим потоком относительно легких транзакций таких как, например, покупка билетов. В ходе такой транзакции меняются несколько таблиц, например, количество свободных мест уменьшается, и добавляется запись о том, что человек с определенным именем совершил покупку. Каждая OLTP-транзакция обычно "трогает" относительно мало данных (используя ключи и/или индексы), но одновременно может работать много таких транзакций, в том числе конкурирующих между собой.
OLAP-транзакции характеризуются тем, что выполняют аналитику над большими данными, что, в свою очередь, выражается в full scan таблиц(ы). OLAP-транзакции зачастую выполняют сортировку и группировку данных. Типичным способом выполнения аналитики над большими данными в распределенной системе является парадигма [MapReduce](https://en.wikipedia.org/wiki/MapReduce).

YDB как платформа позволяет потенциально поддержать оба паттерна нагрузки. Однако в настоящее время YDB является скорее OLTP базой данных. Отличительной чертой YDB является реализация высокопроизводительного механизма выполнения распределенных транзакций на основе детерминистических транзакций (подробнее можно почитать [здесь](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)). YDB была изначально спроектирована и построена с прицелом на распределенные транзакции, поэтому именно OLTP сценарий позволяет системе "раскрыться".

### Ограничения OLTP сценария
На самом деле разница между OLTP и OLAP условна. Например, full scan таблицы из 10 строк это OLAP? А запрос затрагивающий 0.1% петабайтной базы это OLTP? Формально YDB при выполнении запроса руководствуется другими критериями: участвующие в запросе данные должны быть загружены в память целиком, тогда запрос может быть выполнен с уровнем изоляции serializable. Если же оценка загружаемых данных превысит заданный порог, то запрос будет отвергнут, транзакция не будет выполнена и состояние базы не поменяется.

### Текущие возможности для аналитики
Все подходы к выполнению аналитических запросов в YDB сводятся к тому, чтобы взять снэпшот необходимых данных, и запустить асинхронную обработку не блокируя модификацию исходных таблиц. На данных момент существует два способа достичь такого поведения:

1. _CopyTable_. Данная операция выполняет копирование таблицы при помощи механизма [Copy-on-Write](https://en.wikipedia.org/wiki/Copy-on-write). При этом создается новая таблица, которая содержит консистентный снепшот данных исходной таблицы. Новая таблица может быть использована для разных целей, например, для бэкапа в сторонние системы, в том числе в [YT](https://wiki.yandex-team.ru/yt/). Бэкап в YT получается построчным, то есть результирующая таблица содержит исходные данные в формате YT и может быть использована для аналитики. Несмотря на то, что CopyTable копирует только метаданные, ее выполнение может занимать единицы секунд, на это время другие транзакции к исходной таблице будут заблокированы.
2. _ReadTable_. Данная операция позволяет консистентно прочитать таблицу. В ходе выполнения ReadTable фиксируется некоторое консистентное состояние, которое и читается пользователем. Чтение данных пользователем может быть блокирующее (все последующие записи встают в очередь) и неблокирующее. Наибольший интерес в подавляющем большинстве случаев представляет неблокирующее чтение, записи при этом продолжают выполняться, пользователю же доступен неменяющийся стейт. ReadTable является более дешевой операцией с точки зрения накладных расходов по сравнению с CopyTable, поэтому является предпочтительной. Также стоит отметить, что в отличие от CopyTable, результатом которой является изменяемая таблица, ReadTable "снимает" немодифицируемое состояние. Ограничения: ReadTable является более сложной операцией и в настоящее время реализована не в полном объеме.

Обе операции могут быть использованы для подключения сторонних систем выполнения запросов, которые будут использовать YDB как хранилище данных.








Этот раздел описывает особенности реализации [YQL](/yql/) для YDB транзакций. 
YDB предоставляет [YQL](/yql/) в качестве высокоуровнего языка транзакций. 

YDB транзакции могут состоять из нескольких YQL запросов, текущая транзакция заканчивается операцией `COMMIT`. На каждый из запросов YDB отвечает результатами чтения для выражений из этого запроса.

Текущая реализация YQL транзакций использует механизм оптимистичных блокировок для обеспечения уровня изоляции транзакций serializable.

Все модификации данных таблиц и проверка консистентности всех произведенных чтений откладываются до момента коммита транзакции. В случае если консистентность транзакции нарушена, происходит откат (`ROLLBACK`) транзакции, с соответствующим сообщением об ошибке. Откат транзакции может произойти до момента коммита, в случае если нарушение консистентности обнаружено заранее.

Транзакции не видят собственных изменений, все чтения происходят как на момент начала транзакции. Порядок применения модификаций данных соотвествует их порядку в YQL запросе.


### Data Definition Language

| Название  | Описание  | Краткие примеры |
|---|---|---|
| `CREATE TABLE` | Создает таблицу с заданными именем и схемой. Указание первичного ключа обязательно. В случае если таблица с таким именем уже существует не производит никакого действия. |  ```CREATE TABLE [Root/Tmp/Table1] (```<br>```Key1 Uint64,```<br>```Key2 String,```<br>```Value1 String,```<br>```Value2 Int32,```<br>```PRIMARY KEY (Key1, Key2)```<br>```);``` |
| `DROP TABLE` | Удаляет таблицу с заданным именем. Если таблицы с таким именем не существует, возвращается ошибка. | ```DROP TABLE [Root/Tmp/Table1];```  |
| `ALTER TABLE` | Позволяет добавлять/удалять неключевой столбец в существующую таблицу | ```ALTER TABLE [Root/Tmp/Table1] ADD COLUMN deptno Uint32; ```<br>```ALTER TABLE [Root/Tmp/Table1] DROP COLUMN deptno;```  |


### Data Manipulation Language

| Название  | Описание  | Краткие примеры |
|---|---|---|
| INSERT INTO | Вставка данных в существующую таблицу.<br> При попытке вставить строку в таблицу с уже существующим значением первичного ключа, KiKiMR вернёт ошибку с сообщением  Transaction rolled back due to constraint violation: insert_pk |  ``` INSERT INTO [Root/Tmp/Table1] (Key1,Key2, Value1, Value2) ``` <br> ```VALUES (345987,'kikimr', 'Яблочный край', 1414); ``` <br> ```COMMIT; ```  |
| INSERT OR REVERT INTO | Вставка данных в существующую таблицу. При попытке вставить строку в таблицу с уже существующим значением первичного ключа, вся операция  INSERT будет откачена, при этом транзакция останется активной и выполнение запроса будет продолжено |  `INSERT OR REVERT INTO [Root/Tmp/Table1] (Key1,Key2, Value1, Value2) `<br>`VALUES (345987,'kikimr', 'Яблочный край', 1414);`<br>`COMMIT;` |
| UPSERT INTO | Производит добавление/обновление строки таблицы по заданному значению первичного ключа. Если в таблице не было строки с заданным первичным ключем, работает как  INSERT INTO , добавляя новую строку с заданными значениями колонок. В противном случае обновляет существующую строку новыми значениями колонок, при этом значения колонок не участвующих в операции не меняются. | `UPSERT INTO [Root/Tmp/Table1] (Key1, Key2, Value2) VALUES`<br>`(1u, "One", 101),`<br>`(2u, "Two", 102);`<br>`UPSERT INTO [Root/Tmp/Table1]`<br>`SELECT Key AS Key1, "Empty" AS Key2, Value AS Value1`<br>`FROM [Root/Tmp/Table2];`|
| REPLACE INTO | Поведение совпадает с  UPSERT INTO в случае если в таблице не было строки с заданным первичным ключем. <br>В противном случае заменяет существующую строку на новую с заданными значениями колонок, при этом значения колонок не участвующих в операции сбрасывается в значения по умолчанию. | `REPLACE INTO [Root/Tmp/Table1] (Key1, Key2, Value2) VALUES`<br>`(1u, "One", 101),`<br>`(2u, "Two", 102);`<br>`REPLACE INTO [Root/Tmp/Table1]`<br>`SELECT Key AS Key1, "Empty" AS Key2, Value AS Value1 `<br>`FROM [Root/Tmp/Table2];`|
| UPDATE | Получает список строк таблицы по предикату из  WHERE (на момент начала транзакции) и применяет к ним операцию  UPSERT со значениями колонок посчитанным по выражениям из  SET . <br>Не может менять значение первичного ключа. | `UPDATE [Root/Tmp/Table1] `<br>`SET Value1 = YQL::ToString(Value2 + 1), Value2 = Value2 - 1`<br>`WHERE Key1 > 1;`| 
| DELETE | Получает список строк таблицы по предикату из  WHERE (на момент начала транзакции) и удаляет их. | `DELETE FROM [Root/Tmp/Table1] WHERE Key1 == 1 AND Key2 >= "One";` | 
| UPDATE ON <br> DELETE ON |  Невозможность видеть изменения в рамках одной транзакции, описанная выше, приводит к тому, что нельзя делать в рамках одной транзакции операции  UPDATE ,  DELETE или  INSERT над таблицами, которые уже были изменены ранее в этой транзакции. <br>Чтобы иметь возможность менять несколько строк в одной таблице, или сначала читать, потом вставлять, а удалять что-то из одной таблицы, добавлена поддержка конструкций  UPDATE ON и  DELETE ON . Принцип работы констркуций лучше всего ясен из примеров | `USE yctest;`<br>`$to_update = (`<br>`SELECT Key, SubKey, "Updated" AS Value FROM [Root/Home/spuchin/test]`<br>`WHERE Key = 1`<br>`);`<br>``<br>`$to_delete = (`<br>`SELECT Key, SubKey FROM [Root/Home/spuchin/test] WHERE Value = "ToDelete"`<br>`);`<br>``<br>`SELECT * FROM [Root/Home/spuchin/test];`<br>`UPDATE [Root/Home/spuchin/test] ON `<br>`SELECT * FROM $to_update;`<br>`DELETE FROM [Root/Home/spuchin/test] ON `<br>`SELECT * FROM $to_delete;`|




## Transaction Control Language

| Название  | Описание  | Краткие примеры |
|---|---|---|
| COMMIT  | Проверка консистентности и применение всех модификаций данных транзакцией.  |   COMMIT; |
|  ROLLBACK |  Явный откат транзакции | ROLLBACK; |








## Udf

В транзакциях YDB поддерживается ограниченный список предустановленных UDF:

* [DateTime](/yql/udf/list/#datetime)
* [Digest](/yql/udf/list/#digest)
* [Histogram](/yql/udf/list/#histogram)
* HyperLogLog
* [Ip](/yql/udf/list/#ip)
* [Json](/yql/udf/list/#json)
* [Math](/yql/udf/list/#math)
* [Pcre / Pire](/yql/udf/list/#pcre/pire)
* [Re2](/yql/udf/list/#re2)
* Stat
* [String](/yql/udf/list/#string)
* [Unicode](/yql/udf/list/#unicode)
* [Url](/yql/udf/list/#url)

## Не поддерживаемые возможности YQL
* [ParseFile](/yql/userguide/#parsefile), [FileContent/FilePath](/yql/userguide/#filecontent/filepath)
* Python и Javascript UDFs
* `YQL::Unwrap`