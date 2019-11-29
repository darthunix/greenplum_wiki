# Этапы обработки запросов

Ниже показан стек этапов выполнения запроса в GP и его ключевые функции. Так же, в местах, где мы можем встроиться с помощью хука query_info_collect_hook, можно увидеть пометки вида METRICS_XXX_YYY.

```
main.c->main
* postgres.c->PostgresMain
  * ReadCommand (Read command from socket)
    * SocketBackend
      * pq_getbyte
  * exec_parse_message (Parse text to query tree)
    * pg_parse_query
      * raw_parser
    * pg_rewrite_query
      * QueryRewrite
        * RewriteQuery
  * exec_bind_message
    * GetCachedPlan
      * BuildCachedPlan (Generate query plan)
        * pg_plan_queries
          * pg_plan_query
            * planner
              * standard_planner
                * subquery_planner
  * exec_mpp_query (Executed on segments after CdbDispatchPlan from master)
    * pquery.c->PortalRun
      * PortalRunMulti
        * ProcessQuery -> METRICS_QUERY_SUBMIT
          * CreateQueryDesc
          * ResourceManagerGetQueryMemoryLimit (Calculate the amount of memory reserved for the query) + ResLockPortal
          * execMain.c->ExecutorStart
            * standard_ExecutorStart -> METRICS_QUERY_START
              * execMain.c->InitPlan -> acquire locks + METRICS_PLAN_NODE_INITIALIZE
              * cdbdisp_query.c->CdbDispatchPlan (Create connections, gangs and dispatch a plan to segments)
                * cdbdisp_dispatchX
                  * AssignGangs
                    * InventorySliceTree
                      * AllocateGang
                      * cdbgang_createGang
                        * pCreateGangFunc -points-> cdbgang_createGang_async
                          * cdbconn_doConnectStart
                          * cdbconn_doConnectComplete
                * cdbdisp_dispatchToGang
                * cdbdisp_waitDispatchFinish
              * execUtils.c->mppExecutorCleanup -> METRICS_QUERY_CANCELING, METRICS_QUERY_CANCELED/METRICS_QUERY_ERROR
          * execMain.c->ExecutorRun
            * standard_ExecutorRun
              * ExecutePlan
                * execProcnode.c->ExecProcNode -> METRICS_PLAN_NODE_EXECUTING
              * execUtils.c->mppExecutorCleanup -> METRICS_QUERY_CANCELING, METRICS_QUERY_CANCELED/METRICS_QUERY_ERROR
          * execMain.c->ExecutorFinish
            * standard_ExecutorFinish
          * execMain.c->ExecutorEnd
            * standard_ExecutorEnd -> METRICS_INNER_QUERY_DONE/METRICS_QUERY_DONE
              * execMain.c->ExecEndPlan
                * execProcnode.c->ExecEndNode -> METRICS_PLAN_NODE_FINISHED
              * execUtils.c->mppExecutorCleanup -> METRICS_QUERY_CANCELING, METRICS_QUERY_CANCELED/METRICS_QUERY_ERROR
```