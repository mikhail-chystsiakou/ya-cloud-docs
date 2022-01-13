---
sourcePath: core/maintenance/_includes/backup_and_recovery/06_s3_6_forget.md
---
### Завершение операции резервного копирования {#s3_forget}

Для минимизации влияния процесса резервного копирования на показатели пользовательской нагрузки отправка данных осуществляется с копий таблиц. Перед отправкой данных из YDB процесс резервного копирования создаёт консистентные копии всех отправляемых таблиц в YDB. Так как это copy-on-write копии, то в момент их создания размер базы практически не меняется. После завершения отправки данных созданные копии остаются в базе.

Чтобы удалить созданные копии таблиц из базы и завершённую операцию из списка операций, нужно выполнить команду

```
ydb -e $YDB_ENDPOINT -d $YDB_DB_PATH operation forget 'ydb://export/6?id=283824558378666&kind=s3'
```