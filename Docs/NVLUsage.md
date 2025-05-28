
# NVL usage for UNION ALL transformation and index scan  
Для разработчиков платформы IBSO знакома конструкция такого вида в условиях WHERE в запросах 
```sql
(:B1 is null OR A1.C_FILIAL = :B1 )
```
Такие конструкции позволяют создавать универсальный код, который работает в различных режимах в зависимости от значения BIND-переменной. 
Можно выполнять запрос либо по конкретному филиалу, либо по всему обьему данных. 
Это удобно при разработке представлений, отчетов и различных процедур, работающих по разным массивам данных.  
Но "платой" за такую универсальность - неоптимальные планы выполнения "универсальных" запросов.

Недавно я нашел один способ - как сохранить универсальность и получить план запроса, в котором оптимизатором будет внесена "вариативность" 
выполнения двух разных веток в зависимости от значения BIND-переменной.  

Рассмотрим простейший тест-кейс.  
```sql
create table nvl_test (id number, c_filial number);
alter table nvl_test add constraint pk_nvl_test primary key (id) ;
insert into nvl_test (select level, mod(level, 10) from dual connect by level <1000);
create index idx_nvl_test on nvl_test(c_filial);
```

План запроса с обычным выражением через OR выглядит предсказуемо с TABLE FULL SCAN
```sql
explain plan for select * from ibs.nvl_test where (:b1 is null or c_filial =:b1);
select * from dbms_xplan.display(format=>'Advanced');

Plan hash value: 4194980086
 
------------------------------------------------------------------------------
| Id  | Operation         | Name     | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |          |    59 |  1534 |     3   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| NVL_TEST |    59 |  1534 |     3   (0)| 00:00:01 |
------------------------------------------------------------------------------

Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------

1 - SEL$1 / NVL_TEST@SEL$1

Outline Data
-------------

/*+
BEGIN_OUTLINE_DATA
FULL(@"SEL$1" "NVL_TEST"@"SEL$1")
OUTLINE_LEAF(@"SEL$1")
ALL_ROWS
OPT_PARAM('_optimizer_nlj_hj_adaptive_join' 'false')
OPT_PARAM('_optimizer_reduce_groupby_key' 'false')
OPT_PARAM('_optimizer_aggr_groupby_elim' 'false')
OPT_PARAM('_optimizer_strans_adaptive_pruning' 'false')
OPT_PARAM('_px_adaptive_dist_method' 'off')
OPT_PARAM('_optimizer_adaptive_cursor_sharing' 'false')
OPT_PARAM('_optimizer_extended_cursor_sharing_rel' 'none')
OPT_PARAM('_optimizer_extended_cursor_sharing' 'none')
DB_VERSION('19.1.0')
OPTIMIZER_FEATURES_ENABLE('19.1.0')
IGNORE_OPTIM_EMBEDDED_HINTS
END_OUTLINE_DATA
*/

Predicate Information (identified by operation id):
---------------------------------------------------

1 - filter(:B1 IS NULL OR "C_FILIAL"=TO_NUMBER(:B1))

Column Projection Information (identified by operation id):
-----------------------------------------------------------

1 - "NVL_TEST"."ID"[NUMBER,22], "C_FILIAL"[NUMBER,22]

Note
-----
- dynamic statistics used: dynamic sampling (level=2)
```
Но если выражение (:b1 is null or c_filial =:b1) поменять на конструкцию nvl(:b1,c_filial)=c_filial, то план запроса меняется
```sql
explain plan for select * from ibs.nvl_test where nvl(:b1,c_filial)=c_filial;

select * from dbms_xplan.display(format=>'Advanced');
Plan hash value: 4019020071
 
----------------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name            | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |                 |  1009 | 26234 |     5   (0)| 00:00:01 |
|   1 |  VIEW                                  | VW_ORE_B38F0400 |  1009 | 26234 |     5   (0)| 00:00:01 |
|   2 |   UNION-ALL                            |                 |       |       |            |          |
|*  3 |    FILTER                              |                 |       |       |            |          |
|   4 |     TABLE ACCESS BY INDEX ROWID BATCHED| NVL_TEST        |    10 |   260 |     2   (0)| 00:00:01 |
|*  5 |      INDEX RANGE SCAN                  | IDX_NVL_TEST    |     4 |       |     1   (0)| 00:00:01 |
|*  6 |    FILTER                              |                 |       |       |            |          |
|*  7 |     TABLE ACCESS FULL                  | NVL_TEST        |   999 | 25974 |     3   (0)| 00:00:01 |
----------------------------------------------------------------------------------------------------------

Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------

1 - SET$2A13AF86   / VW_ORE_B38F0400@SEL$B38F0400
2 - SET$2A13AF86
3 - SET$2A13AF86_1
4 - SET$2A13AF86_1 / NVL_TEST@SET$2A13AF86_1
5 - SET$2A13AF86_1 / NVL_TEST@SET$2A13AF86_1
6 - SET$2A13AF86_2
7 - SET$2A13AF86_2 / NVL_TEST@SET$2A13AF86_2

Outline Data
-------------

/*+
BEGIN_OUTLINE_DATA
FULL(@"SET$2A13AF86_2" "NVL_TEST"@"SET$2A13AF86_2")
BATCH_TABLE_ACCESS_BY_ROWID(@"SET$2A13AF86_1" "NVL_TEST"@"SET$2A13AF86_1")
INDEX_RS_ASC(@"SET$2A13AF86_1" "NVL_TEST"@"SET$2A13AF86_1" ("NVL_TEST"."C_FILIAL"))
NO_ACCESS(@"SEL$9162BF3C" "VW_ORE_B38F0400"@"SEL$B38F0400")
OUTLINE(@"SEL$1")
OR_EXPAND(@"SEL$1" (1) (2))
OUTLINE_LEAF(@"SEL$9162BF3C")
OUTLINE_LEAF(@"SET$2A13AF86")
OUTLINE_LEAF(@"SET$2A13AF86_1")
OUTLINE_LEAF(@"SET$2A13AF86_2")
ALL_ROWS
OPT_PARAM('_optimizer_nlj_hj_adaptive_join' 'false')
OPT_PARAM('_optimizer_reduce_groupby_key' 'false')
OPT_PARAM('_optimizer_aggr_groupby_elim' 'false')
OPT_PARAM('_optimizer_strans_adaptive_pruning' 'false')
OPT_PARAM('_px_adaptive_dist_method' 'off')
OPT_PARAM('_optimizer_adaptive_cursor_sharing' 'false')
OPT_PARAM('_optimizer_extended_cursor_sharing_rel' 'none')
OPT_PARAM('_optimizer_extended_cursor_sharing' 'none')
DB_VERSION('19.1.0')
OPTIMIZER_FEATURES_ENABLE('19.1.0')
IGNORE_OPTIM_EMBEDDED_HINTS
END_OUTLINE_DATA
*/

Predicate Information (identified by operation id):
---------------------------------------------------

3 - filter(:B1 IS NOT NULL)
5 - access("C_FILIAL"=:B1)
6 - filter(:B1 IS NULL)
7 - filter("C_FILIAL" IS NOT NULL)

Column Projection Information (identified by operation id):
-----------------------------------------------------------

1 - "ITEM_1"[NUMBER,22], "ITEM_2"[NUMBER,22]
2 - STRDEF[22], STRDEF[22]
3 - "NVL_TEST".T"."ID"[NUMBER,22], "C_FILIAL"[NUMBER,22]
"ID"[NUMBER,22], "C_FILIAL"[NUMBER,22]
4 - "NVL_TEST"."ID"[NUMBER,22], "C_FILIAL"[NUMBER,22]
5 - "NVL_TEST".ROWID[ROWID,10], "C_FILIAL"[NUMBER,22]
6 - "NVL_TEST"."ID"[NUMBER,22], "C_FILIAL"[NUMBER,22]
7 - "NVL_TES
Note
-----
- dynamic statistics used: dynamic sampling (level=2)
 ```
Интересны 3 и 6 шаги плана.
В разделе Predicate Information видно, что две ветки плана выполняются каждая при определенном условии.
3 - filter(:B1 IS NOT NULL)
6 - filter(:B1 IS NULL)
Соотвественно, если мы передаем в переменную код филиала для отбора данных по одному филиалу, что будет выполнено индексное сканирование, 
а блок с TABLE FULL SCAN будет проигнорирован при выполнении
