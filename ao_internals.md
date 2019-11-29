# Внутреннее устройство AO и CO таблиц

Принципиальное устройство строковых AO (append optimized) и столбцовых CO (column oriented) таблиц весьма схоже. Ниже на картинке показано принципиальное устройство AO таблицы.

![](ao_internals.png | width=700)

## Основные компоненты

  * **Сегментные файлы** - большие файлы (более 1ГБ), в которые последовательно дописываются блоки с данными. Нулевой сегментный файл выделен для utility режима и модифицирующих операций, всего сегментных файлов в AO таблице может быть 128, а в CO - 128 * (к-во столбцов). Т.е. для CO таблицы под первую колонку будут по мере необходимости создаваться сегментные файлы 0-127 (0 зарезервирован), под вторую - 128-255 (128 зарезервирован) и т.д. Ограничение в 128 файлов станет понятно из устройства идентификатора кортежа (TID) для AO/CO таблиц. Каждый запрос эксклюзивно пишет в конец строго одному сегментному файлу в случае AO таблицы (в случае CO таблицы пишет в сегментные файлы по количеству колонок). Если параллельно возникает еще одна транзакция, то она или находит следующий незанятый сегментный файл в случае AO, например 2 (а в случае CO - 2, 129, ...) или порождает такой сегментный файл самостоятельно (они создаются "лениво", а не заранее). Итоговая параллельность на модификацию - 127 запросов. Каждый сегментный файл строится из блоков произвольной длины, содержащих заголовок и данные (если таблица со сжатием, то именно данные в блоках и сжимаются). В блоках лежат кортежи, при этом у каждого кортежа есть свой TID - идентификатор кортежа (tuple id). Для CO и AO файлов он представляет комбинацию из сегментного файла и номера строки. TID занимает 48 бит:
    * 7 левый бит под номер сегментного файла (2^7 = 128 - отсюда и ограничение на количество сегментный файлов)
    * 16 правых бит - зарезервированы на будущее и всегда 1
    * 40 оставшихся бит - номер строки (в AO/CO таблице может быть (2^40 - 1) ~ 1099 триллионов строк)
  * **Таблица состояния сегментных файлов** - heap таблица, содержащая сегментные файлы AO/CO таблицы и указатели смещения их конца. Когда мы заканчиваем писать в сегментный файл данные, мы обновляем указатель на конец файла в этой таблице. А так как таблица heap, то она подчиняется правилам MVCC PostgreSQL и мы работаем с AO/CO таблицами транзакционно.
  * **Таблица карты видимости** - heap таблица, хранящая сжатую битовую маску, скрывающую кортежи из сегментного файла. На самом деле мы не удаляем данные из сегментных файлов, выполняя команды DELETE или UPDATE (= DELETE + INSERT), а скрываем старые версии в битовой маске и добавляем в конец сегментного файла блоки с новыми значениями. Если в этой таблице грязных данных меньше некоторого значения отсечки, то VACUUM не будет трогать данный сегментный файл. Если больше - то выберет уже существующий сегментный файл и перельет в него живые значения (а старый сегментный файл будет со временем удален). VACUUM FULL же переливает данные в другой сегментный файл всегда.
  * **Таблица карты блоков** - heap таблица, содержащая карту смещений блоков сегментного файла и хранящегося в блоке диапазона TID. Данная таблица нужна для произвольного доступа к кортежу в AO/CO таблице по индексу, поэтому она создается только, если у AO/CO таблицы есть индекс. Существует механизм трансляции TID AO/CO таблиц в heap-совместимый TID для индексов и обратно для использования без изменений стандартных PostgreSQL индексов. Поэтому, при поиске по индексу в AO/CO таблице возникает дополнительный заход в таблицу карты блоков, чтобы выбрать нужный блок сегментного файла. В случае последовательного сканирования без индекса мы выбираем подходящие нам сегментные файлы (для AO таблицы - все, для CO таблицы - только относящиеся к колонкам в запросе) и получаем номер строки из TID, который мы извлекаем вместе с остальным кортежем, разжимая блок с данными (в случае CO таблицы по этим номерам строк будет склеен нужный нам кортеж). На текущий момент при последовательном сканировании с условиями мы разжимаем все блоки (адресный выбор блоков возможен только при наличие индекса).

## Примечания

Для всех вспомогательных таблиц OID основной AO/CO таблицы располагается в их имени (например, pg_aoseg.pg_aoseg_<oid>), но на это полагаться нельзя, так как некоторые модифицирующие операции на таблицы могут менять OID таблицы. Поэтому вспомогательные таблицы можно получить из pg_catalog.pg_appendonly (данные в ней хранятся на сегментах, нужно использовать gp_dist_random).