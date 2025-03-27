# PLSQL Warnings and Column Type mismatch

Многократно сталкивался  с ситуацией игнорирования индекса в плане выполнения.

Есть таблица Z#TEST1

У этой таблицы есть нужный индекс

Однако выполнение элементарного запроса приводит в FULL SCAN

```
Xplain plan
SQL_ID  0haftpw35pqd4, child number 0
-------------------------------------
SELECT A1.ID FROM Z#TEST1 A1 WHERE A1.C_DOCUM_RC_OBJ = :B1
AND A1.C_DATE_END IS NULL

Plan hash value: 1145617301
 
---------------------------------------------------------------------------------------
| Id  | Operation         | Name              | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |                   |       |       |   890 (100)|          |
|*  1 |  TABLE ACCESS FULL| Z#TEST1   |     1 |    28 |   890   (1)| 00:00:01 |
---------------------------------------------------------------------------------------

Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------

1 - SEL$1 / A1@SEL$1

Outline Data
-------------

/*+
BEGIN_OUTLINE_DATA
IGNORE_OPTIM_EMBEDDED_HINTS
OPTIMIZER_FEATURES_ENABLE('19.1.0')
DB_VERSION('19.1.0')
OPT_PARAM('_optimizer_extended_cursor_sharing' 'none')
OPT_PARAM('_optimizer_extended_cursor_sharing_rel' 'none')
OPT_PARAM('_optimizer_adaptive_cursor_sharing' 'false')
OPT_PARAM('_px_adaptive_dist_method' 'off')
OPT_PARAM('_optimizer_strans_adaptive_pruning' 'false')
OPT_PARAM('_optimizer_aggr_groupby_elim' 'false')
OPT_PARAM('_optimizer_reduce_groupby_key' 'false')
OPT_PARAM('_optimizer_nlj_hj_adaptive_join' 'false')
OPT_PARAM('optimizer_index_cost_adj' 8)
OPT_PARAM('optimizer_index_caching' 20)
ALL_ROWS
OUTLINE_LEAF(@"SEL$1")
FULL(@"SEL$1" "A1"@"SEL$1")
END_OUTLINE_DATA
*/

Peeked Binds (identified by position):
--------------------------------------

1 - :B1 (NUMBER): 55837435106

Predicate Information (identified by operation id):
---------------------------------------------------

1 - filter(("A1"."C_DATE_END" IS NULL AND
TO_NUMBER("A1"."C_DOCUM_RC_OBJ")=:B1))

Column Projection Information (identified by operation id):
-----------------------------------------------------------

1 - "A1"."ID"[NUMBER,22]
```
Причину хорошо видно в результатах выполнения функции dbms_xplan.display_cursor (результаты выше) в разделе  
```
Predicate Information (identified by operation id):
---------------------------------------------------
1 - filter(("A1"."C_DATE_END" IS NULL AND TO_NUMBER("A1"."C_DOCUM_RC_OBJ")=:B1))
```

Тип колонки преобразуется к типу bind-переменной.  
И это еще повезло, что все значения в колонке таблицы в действительности числа. Но ничто не мешает записать в колонку таблицы какой-то символ. 
И тогда на этапе выполнения в один прекрасный момент будем получать ошибку.  
Данный кейс использования полей VARCHAR2 для хранения чисел (идентификаторов) очень часто встречается к коде ЦФТ.  
Подобные проблемы легко идентифицируются, но, как правило, их замечают уже в процессе эксплуатации. Тогда, когда таблица накопит значительный обьем и full scan начнет влиять на производительность.

НО. ЕСТЬ СПОСОБ ЛЕГКО ПОЙМАТЬ ТАКИЕ КЕЙСЫ ЕЩЕ НА ЭТАПЕ РАЗРАБОТКИ.

Делается это очень просто.  
Запрос выше находится в пакете Z$TEST1_LIB в 123 строке

```
--# 145,4
declare
X number;
cursor c_obj is
select  a1.id
from Z#TEST1 a1
where a1.C_DOCUM_RC_OBJ = P_DOC_RC and a1.C_DATE_END is NULL;
begin
for plp$c_obj in c_obj loop
X := plp$c_obj.id;
--# 149,18
Z#TEST1#INTERFACE.s#date_end(X,SYSDATE);
end loop;
end;
--# 153,4
Z#TEST1#INTERFACE.init(CTRLRC,true,true);
```

Если выполнить следующую последовательность SQL-команд, то warning с кодом PLW-07204 точно укажет на проблемное место.  
При этом совсем не требуется выполнять саму операцию и анализировать планы выполнения запросов. Все сделает компилятор.  

Ниже пример выполнения в SQLPlus.  

```
SQL*Plus: Release 11.2.0.4.0 Production on Пт Апр 15 11:21:53 2022

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Присоединен к:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production

SQL> alter session set PLSQL_WARNINGS='enable:all';

Сеанс изменен.

SQL> alter package IBS.Z$TEST1_LIB compile body;

SP2-0811: Тело пакета изменено с предупреждениями компилятора

SQL> show errors;
Ошибки для PACKAGE BODY IBS.Z$TEST1_LIB:

LINE/COL ERROR
-------- -----------------------------------------------------------------
72/98    PLW-07203: параметр 'V_TYPE' можно использовать лучше, если
воспользоваться подсказкой компилятора NOCOPY

123/13   PLW-07204: результатом преобразования к типу, отличному от типа
столбца, может стать субоптимальный план очереди

166/3    PLW-06010: ключевое слово "RESULT" использовано как определенное
имя
```
Аналогичную информацию о warnings можно получить из представления USER_ERRORS или ALL_ERRORS 

