# Восстановление данных в gp_persistent_filespace_node

## Таблица gp_persistent_filespace_node
Данная таблица хранит взаимосвязь состояния объектов файловой системы и связанных с ними объектов файловых пространств. Эта информация используется для контроля целостности системных каталогов и соответствующих им объектов файлов на диске. Данная информация используется в процессе файловой репликации между праймари и зеркалом.

  * **filespace_oid** - oid файлового пространства
  * **db_id_1** - идентификатор первого узла в паре праймари-зеркало (обычно, сегмента с preferred_role='p')
  * **location_1** - расположение файлового пространства для данного узла
  * **db_id_2** - идентификатор второго узла
  * **location_2** - расположение файлового пространства для второго узла
  * **persistent_state** - 0 - free, 1 - create pending, 2 - created, 3 - drop pending, 4 - aborting create, 5 - "Just in Time" create pending, 6 - bulk load create pending
  * **mirror_existence_state** - 0 - none, 1 - not mirrored, 2 - mirror create pending, 3 - mirrorcreated, 4 - mirror down before create, 5 - mirror down during create, 6 - mirror drop pending, 7 - only mirror drop remains
  * **parent_xid** - глобальный идентификатор транзакции
  * **persistent_serial_num** - Номер позиции (LSN) в журнале транзакций для блока файла.
  * **previous_free_tid** - используется ADB для внутреннего управления постоянным представлениями объектов файловой системы.

Также данные этой таблицы используются при обращении к объектам, хранящимся на файловых пространствах. Каждый сегмент (пара праймари-зеркало), содержит данные только о своих объектах на файловой системе.

## Инициализация hash map таблицы gp_persistent_filespace_node

Экземпляры PG читают данные (1 строка) из таблицы gp_persistent_filespace_node не напрямую, а из hash map, которая строится на ее основе при запуске postmaster. Поэтому, когда мы что-то правим в gp_persistent_filespace_node мы это делаем в двух местах: в hash map и напрямую в таблице gp_persistent_filespace_node. Ниже описан процесс инициализации hash map при запуске postmaster.

  - main
  - SubPostmasterMain
  - CreateSharedMemoryAndSemaphores
  - PersistentFilespace_ShmemInit
    * Создаем пустую hash map в разделяемой памяти
  - PersistentFileSysObj_Init
    * PersistentStore_Init(
      * storeData,
      * "gp_persistent_filespace_node",
      * GpGlobalSequence_PersistentFilespace,
      * DirectOpen_GpPersistentFileSpaceNodeOpenShared,
      * DirectOpen_GpPersistentFileSpaceNodeClose,
      * scanTupleCallback,
      * PersistentFileSysObj_PrintFilespaceDir,
      * Persistent_FilespaceScanKeyInit,
      * Persistent_FilespaceAllowDuplicateEntry,
      * Natts_gp_persistent_filespace_node,
      * Anum_gp_persistent_filespace_node_persistent_serial_num);
      * 
      * При этом, например, DirectOpen_GpPersistentFileSpaceNodeOpenShared под капотом использует макросы файла cdbdirectopen.h, которые с помощью препроцессинга генерируют по шаблонам сигнатуры и тела функций. По факту DirectOpen_GpPersistentFileSpaceNodeOpenShared с помощью DirectOpenDefineExtern(DirectOpen_GpPersistentFileSpaceNode)->Relation name##OpenShared(void) вернет открытую для прямой работы таблицу gp_persistent_filespace_node
    * PersistentFilespace_ScanTupleCallback
      * GpPersistentFilespaceNode_GetValues - заполняет единственную строчку в хеш мапе из таблицы
 

## Повреждение данных

**В процессе восстановления упавших зеркал с полной перезаливкой данных с помощью утилиты gprecoverseg -F возможно повреждение хранящихся в данной таблице данных на единственном праймари, в случае, если процесс был прерван**:

filespace_oid                                | 16713
db_id_1                                      | 0
location_1                                   |
db_id_2                                      | 0
location_2                                   |
persistent_state                             | 2
create_mirror_data_loss_tracking_session_num | 0
mirror_existence_state                       | 1
reserved                                     | 0
parent_xid                                   | 0
persistent_serial_num                        | 1

При этом данные хранящиеся на данном сегменте в поврежденных файловых пространствах перестают быть доступны. При обращении к ним возвращается ошибка DTM.

gprecoverseg в процессе полного восстановления зеркал производит их удаление из конфигурации с последующим добавлением обратно. Аналогичные операции производятся с метаданными в таблицах gp_persistent_*. Процесс удаления зеркала происходит по следующему стеку вызовов:

  - GpRecoverSegmentProgram
  - def run(self):
  - def buildMirrors(self, actionName, gpEnv, gpArray):
  - def updateSystemConfig( self, gpArray, textForConfigTable, dbIdToForceMirrorRemoveAdd, useUtilityMode, allowPrimary):
  - %%def __updateSystemConfigRemoveAddMirror(self, conn, gpArray, seg, textForConfigTable):%%
  - %%def __callSegmentRemoveMirror(self, conn, seg):%%
  - gp_remove_segment_mirror(PG_FUNCTION_ARGS)
  - remove_segment(int16 pridbid, int16 mirdbid)
  - remove_segment_persistent_entries(int16 pridbid, seginfo *i)
  - gp_remove_segment_persistent_entries(PG_FUNCTION_ARGS)
  - remove_segment_persistent_entries(int16 pridbid, seginfo *i)
  - update_filespaces(int16 pridbid, seginfo *i, bool add, ArrayType *fsmap)
  - desist_all_filespaces(int16 dbid)
  - PersistentFilespace_RemoveSegment(int16 dbid, bool ismirror)
  - PersistentFileSysObj_RemoveSegment
 
В последней функции (а также в аналогичной add-функции) содержится занятный код:

```c
if (DatumGetInt16(values[dbid_1_attnum - 1]) == dbid)
{
    dbidattnum = Anum_gp_persistent_filespace_node_db_id_1;
    locattnum = Anum_gp_persistent_filespace_node_location_1;
}
else
{
    dbidattnum = Anum_gp_persistent_filespace_node_db_id_2;
    locattnum = Anum_gp_persistent_filespace_node_location_2;
}

replaces[dbidattnum - 1] = true;
newValues[dbidattnum - 1] = Int16GetDatum(0);

/*
* Although ep is allocated on the stack, CStringGetTextDatum()
* duplicates this and puts the result in palloc() allocated memory.
*/
replaces[locattnum - 1] = true;
newValues[locattnum - 1] = CStringGetTextDatum(ep);
```

dbid содержит идентификатор удаляемого зеркала. Проблема данного кода в том, что если идентификатор удаляемого зеркала не принадлежит первой записи (db_id_1), то всегда сбрасывается вторая без проверки соответствия ее идентификатору.

Мной был проведен следующий эксперимент. В код функции __updateSystemConfigRemoveAddMirror между удалением и добавлением зеркала был внедрен таймаут в 120 секунд. Если убить gprecoverseg в этой фазе, запись для первичного хоста будет утеряна. Если повторить запуск gprecoverseg и убить его еще раз, будет уничтожена и вторая запись (из-за особенности кода выше). Что с большой долей вероятности и произошло в случае с кластером X5. При этом кластер будет работать до перезапуска за счет данных в хэш-мапе (привет, отказ после REINDEX).

Первопричиной повреждения метаданных явился искомая нами взаимоблокировка fts и gprecoverseg. gp_remove_segment_mirror сожержит занятный комментарий:

```c
/*
 * avoid races - no one else should be allowed to update
 * gp_segment_configuration when we read and add entry to this table. Use
 * ExclusiveLock to still allow FTS probe to proceed, because we ourselves
 * might request a fts probe in case of failure or timeout when QD polls
 * results of add_segment_persistent_entries() from QEs. Note that FTS
 * acquires an AccessShareLock when it reads from gp_segment_configuration,
 * and only acquires another RowExclusiveLock if it decides to modify it.
 *
 * FIXME: there exists an edge case that if any other segment failed during
 * the FTS probe requested by us, deadlock will still happen.
 */
```

## Восстановление

Прямая запись в данную таблице недоступна даже с установленным GUCом:
```
allow_system_table_mods='dml'
```
Для управления данными предоставлено несколько функций, доступных для запуска через sql.

### gp_update_persistent_filespace_node_entry

Функция заглушка с макросом Not Implemented Yet (верно для версий по актуальную 5.23).

### gp_add_segment_persistent_entries

Данная функция принимает в качестве аргументов идентификаторы праймари, зеркала и список пар - имя файлового пространства, путь для хранения данных. Функция может быть запущена только локально на сегментах и только супер-пользователем. Если первый аргумент не совпадает с идентификатором сегмента, на котором происходит выполнение запроса, запрос будет завершен с успехом:
```c
if (dbid != GpIdentity.dbid)
    PG_RETURN_BOOL(true);
```
Данная функция вызывает: add_segment_persistent_entries(dbid, &seg, fsmap) => update_filespaces(pridbid, i, true, fsmap) => persist_all_filespaces(pridbid, i->db.dbid, fsmap) => PersistentFilespace_AddMirror(fsoid, path, pridbid, mirdbid, set_mirror_existence), которая в свою очередь ищет в таблице gp_persistent_filespace_node запись, соответствующую текущему dbid, и обновляет другую запись (запись зеркала, данного сегмента).

Так как обе записи в приведенном выше примере были обнулены, при работе в составе кластера применение этой функции не внесет необходимых изменений. Необходимо предварительно зафиксировать:
  * номер контента **C**
  * номера dbid двух сегментов из пары **P** и **M**
  * количество контентов в кластере **N**
  * путь к дата-каталогу оставшегося в живых сегмента, на котором будет производиться восстановление **PATH** и порт **PORT**
  * пути для имени каждого файлового пространства для праймари и для зеркала.

Затем необходимо остановить работу кластера и получить доступ к консоли хоста, на котором расположен текущий праймари из пары. Затем необходимо произвести запуск сегмента в dbid=0:
```
/usr/lib/gpdb/bin/postgres -D PATH -p PORT --gp_dbid=0 --gp_num_contents_in_cluster=N --silent-mode=true -i --gp_contentid=C
```
Подключиться к нему в режиме utility:
```
PGOPTIONS='-c gp_session_role=utility' psql -p PORT postgres
```
Восстановить информацию для **зеркала**:
```sql
select gp_add_segment_persistent_entries(0::int2,M::int2, array [['data1', 'путь к файловому пространству для зеркала']]);
```
Остановить сегмент:
```
pg_ctl stop -D PATH
```
Запустить с dbid зеркала
```
/usr/lib/gpdb/bin/postgres -D PATH -p PORT --gp_dbid=M --gp_num_contents_in_cluster=N --silent-mode=true -i --gp_contentid=C
```
Восстановить информацию праймари:
```sql
select gp_add_segment_persistent_entries(M::int2,P::int2, array [['data1', 'путь к файловому пространству для праймари']]);
```
Убедиться, что таблица gp_persistent_filespace_node содержит корректные данные.
Остановить сегмент:
```
pg_ctl stop -D PATH
```
Запустить кластер.