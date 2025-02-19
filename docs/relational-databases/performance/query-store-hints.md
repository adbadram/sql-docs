---
title: "Query Store hints"
description: "Learn about the Query Store hints feature, which can be used to shape query plans without changing application code."
ms.custom:
- event-tier1-build-2022
ms.date: "08/01/2022"
ms.prod: sql
ms.prod_service: "database-engine, sql-database"
ms.technology: performance
ms.topic: conceptual
helpviewer_keywords: 
  - "Query Store hints"
dev_langs:
 - "TSQL"
author: WilliamDAssafMSFT
ms.author: wiassaf
monikerRange: "=azuresqldb-current||=azuresqldb-mi-current||>=sql-server-ver16||>=sql-server-linux-ver16"
---

# Query Store hints
[!INCLUDE [sqlserver2022-asdb-asdbmi](../../includes/applies-to-version/sqlserver2022-asdb-asmi.md)]

This article outlines how to apply query hints using the Query Store. Query Store hints provide an easy-to-use method for shaping query plans without changing application code. 

Query Store hints are available in [!INCLUDE[ssSDSfull](../../includes/sssdsfull-md.md)] and [!INCLUDE[ssazuremi_md](../../includes/ssazuremi_md.md)]. Query Store hints are also a feature introduced to SQL Server in [!INCLUDE[sssql22-md](../../includes/sssql22-md.md)].

- For more information on configuring and administering with the Query Store, see [Monitoring performance by using the Query Store](monitoring-performance-by-using-the-query-store.md).
- For information on discovering actionable information and tune performance with the Query Store, see [Tuning performance by using the Query Store](tune-performance-with-the-query-store.md).
- For information about operating the Query Store in Azure [!INCLUDE[ssSDS](../../includes/sssds-md.md)], see [Operating the Query Store in Azure SQL Database](best-practice-with-the-query-store.md#Insight).

> [!CAUTION]
> Because the SQL Server Query Optimizer typically selects the best execution plan for a query, we recommend only using hints as a last resort for experienced developers and database administrators. For more information, see [Query Hints](../../t-sql/queries/hints-transact-sql-query.md).

## Overview

Ideally the Query Optimizer selects an optimal execution plan for a query. When this doesn't happen, a developer or DBA may wish to manually optimize for specific conditions. Query hints are specified via the OPTION clause and can be used to affect query execution behavior. While query hints help provide localized solutions to various performance-related issues, they do require a rewrite of the original query text. Database administrators and developers may not always be able to make changes directly to [!INCLUDE[tsql](../../includes/tsql-md.md)] code to inject a query hint. The [!INCLUDE[tsql](../../includes/tsql-md.md)] may be hard-coded into an application or automatically generated by the application. Previously, a developer may have to rely on [plan guides](plan-guides.md), which can be complex to use.

For information on which query hints can be applied, see [Supported query hints](../system-stored-procedures/sys-sp-query-store-set-hints-transact-sql.md#supported-query-hints).

## When to use Query Store hints

As the name suggests, this feature extends and depends on the [Query Store](monitoring-performance-by-using-the-query-store.md). Query Store enables the capturing of queries, execution plans, and associated runtime statistics. Introduced in [!INCLUDE[sssql16-md](../../includes/sssql16-md.md)] and on-by-default in [!INCLUDE[ssSDSfull](../../includes/sssdsfull-md.md)], Query Store greatly simplifies the overall performance tuning customer experience.  

:::image type="complex" source="media/query-store-hints.png" alt-text="The workflow for Query Store Hints.":::
      First the query is executed, then captured by Query Store. Then the DBA creates a Query Store hint on a query. Thereafter, the query is executed using the Query Store hint.
:::image-end:::

Examples where Query Store hints can help with query-level performance issues:
*    Recompile a query on each execution.
*    Cap the memory grant size for a bulk insert operation.
*    Limit the maximum degree of parallelism for a statistics update operation.
*    Use a Hash join instead of a Nested Loops join.
*    Use [compatibility level](../databases/view-or-change-the-compatibility-level-of-a-database.md) 110 for a specific query while keeping everything else in the database at compatibility level 150.
*    Disable row goal optimization for a SELECT TOP query.

To use Query Store hints:
1.    Identify the Query Store `query_id` of the query statement you wish to modify. You can do this in various ways: 
    1.1. Querying the [Query Store catalog views](../system-catalog-views/query-store-catalog-views-transact-sql.md).
    1.2. Using [!INCLUDE[ssManStudioFull](../../includes/ssmanstudiofull-md.md)] built-in Query Store reports.
    1.3. Using Azure portal Query Performance Insight for [!INCLUDE[ssSDSfull](../../includes/sssdsfull-md.md)].
1.    Execute `sys.sp_query_store_set_hints` with the `query_id` and query hint string you wish to apply to the query.  This string can contain one or more query hints. For complete information, see [sys.sp_query_store_set_hints](../system-stored-procedures/sys-sp-query-store-set-hints-transact-sql.md).

Once created, Query Store hints are persisted and survive restarts and failovers. Query Store hints override hard-coded statement-level hints and existing plan guide hints. 

If a query hint contradicts what is possible for query optimization, the hint won't block query execution and the hint won't be applied. In the cases where a hint would cause a query to fail, the hint is ignored and the latest failure details can be viewed in [sys.query_store_query_hints](../system-catalog-views/sys-query-store-query-hints-transact-sql.md).

Watch this video for an overview of Query Store hints:

> [!VIDEO https://channel9.msdn.com/Shows/Data-Exposed/Query-Store-Hints-in-Azure-SQL-Database/player?WT.mc_id=dataexposed-c9-niner]

## Query Store hints system stored procedures

To create or update hints, use [sys.sp_query_store_set_hints](../system-stored-procedures/sys-sp-query-store-set-hints-transact-sql.md). Hints are specified in a valid string format N'OPTION (...)'. 

* When creating a Query Store hint, if no Query Store hint exists for a specific `query_id`, a new Query Store hint will be created.
* When creating or updating a Query Store hint, if a Query Store hint already exists for a specific `query_id`, the last value provided will override previously specified values for the associated query.
* If a `query_id` doesn't exist, an error will be raised. 

> [!Note]
> For a complete list of hints that are supported, see [sys.sp_query_store_set_hints](../system-stored-procedures/sys-sp-query-store-set-hints-transact-sql.md).

To remove hints associated with a `query_id`, use [sys.sp_query_store_clear_hints](../system-stored-procedures/sys-sp-query-store-clear-hints-transact-sql.md).

## Execution Plan XML attributes

When hints are applied, the following result set appears in the `StmtSimple` element of the [Execution Plan](execution-plans.md) in [XML format](save-an-execution-plan-in-xml-format.md):

|**Attribute**| **Description**|
|--|--|
|`QueryStoreStatementHintText`|Actual Query Store hint(s) applied to the query|
|`QueryStoreStatementHintId`|Unique identifier of a query hint|
|`QueryStoreStatementHintSource`|Source of Query Store hint (ex: "User")|

> [!Note]
> These XML elements are available via the output of the [!INCLUDE[tsql](../../includes/tsql-md.md)] commands [SET STATISTICS XML](../../t-sql/statements/set-statistics-xml-transact-sql.md) and [SET SHOWPLAN XML](../../t-sql/statements/set-showplan-xml-transact-sql.md).


## Query Store hints and feature interoperability

*   Query Store hints will override other hard-coded statement level hints and plan guides.
*   Queries will always execute where any opposing Query Store hints, that would otherwise cause an error, will be ignored.
*   If Query Store hints contradict, SQL Server will not block query execution and Query Store hint will not be applied.
*   Simple parameterization - Query Store hints are not supported for statements that qualify for simple parameterization.
*   Forced parameterization - The RECOMPILE hint is not compatible with forced parameterization set at the database level. If the database has forced parameterization set, and the RECOMPILE hint is part of the hints string set in Query Store for a query, SQL Server will ignore the RECOMPILE hint and will apply any other hints if they are leveraged.
    *    Additionally, SQL Server will issue a warning (error code 12461) stating that the RECOMPILE hint was ignored.
    *    For more information on forced parameterization use case considerations, see [Guidelines for Using Forced Parameterization](../query-processing-architecture-guide.md#forced-parameterization).
*   Currently, Query Store hints can be applied against the primary replica of an Always On availability group.

## Query Store hints best practices

*    Complete index and statistics maintenance before evaluating queries for potential new Query Store hints.
*    Test your application database on the latest [compatibility level](../../t-sql/statements/alter-database-transact-sql-compatibility-level.md), before leveraging Query Store hints.
     * For example, Parameter Sensitive Plan (PSP) optimization was introduced in SQL Server 2022 (compatibility level 160), which leverages multiple active plans per query to address non-uniform data distributions. If your environment cannot use the latest compatibility level, Query Store hints using the RECOMPILE hint can be leveraged on any supporting compatibility level.
*    Query Store hints override SQL Server query plan behavior. It is recommended to only leverage Query Store hints when it is necessary to address performance related issues.
*    It is recommended to reevaluate Query Store hints, statement level hints, plan guides, and Query Store forced plans any time data distributions change and during database migrations projects. Changes in data distribution may cause Query Store hints to generate suboptimal execution plans.

## Examples  

### A. Query Store hints demo
The following walk-through of Query Store hints in Azure SQL Database uses an imported database via a BACPAC file (.bacpac). Learn how to import a new database to an Azure SQL Database server, see [Quickstart: Import a BACPAC file to a database](/azure/azure-sql/database/database-import).

:::code language="tsql" source="../../../sql-server-samples/samples/features/query-store/query_store_hints_demo.sql":::

### B. Identify a query in Query Store

The following example queries [sys.query_store_query_text](../system-catalog-views/sys-query-store-query-text-transact-sql.md) and [sys.query_store_query](../system-catalog-views/sys-query-store-query-transact-sql.md) to return the `query_id` for an executed query text fragment. 

In this demo, the query we're attempting to tune is in the `SalesLT` sample database: 

```sql
SELECT * FROM SalesLT.Address as A 
INNER JOIN SalesLT.CustomerAddress as CA
on A.AddressID = CA.AddressID
WHERE PostalCode = '98052' ORDER BY A.ModifiedDate DESC;
```

Note that Query Store doesn't immediately reflect query data to its system views.

Identify the query in the query store system catalog views:

```sql
SELECT q.query_id, qt.query_sql_text
FROM sys.query_store_query_text qt 
INNER JOIN sys.query_store_query q ON 
    qt.query_text_id = q.query_text_id 
WHERE query_sql_text like N'%PostalCode =%'  
  AND query_sql_text not like N'%query_store%';
GO
```

In the following samples, the previous query example in the `SalesLT` database was identified as `query_id` 39.

Once identified, apply the hint to enforce a maximum memory grant size in percent of configured memory limit to the `query_id`:
  
```sql
EXEC sys.sp_query_store_set_hints @query_id= 39, @query_hints = N'OPTION(MAX_GRANT_PERCENT=10)';
```  

 You can also apply query hints with the following syntax, for example the option to force the [legacy cardinality estimator](../performance/cardinality-estimation-sql-server.md):

```sql
EXEC sys.sp_query_store_set_hints @query_id= 39, @query_hints = N'OPTION(USE HINT(''FORCE_LEGACY_CARDINALITY_ESTIMATION''))';
```  

 You can apply multiple query hints with a comma-separated list:

```sql
EXEC sys.sp_query_store_set_hints @query_id= 39, @query_hints = N'OPTION(RECOMPILE, MAXDOP 1, USE HINT(''QUERY_OPTIMIZER_COMPATIBILITY_LEVEL_110''))';
```

 Review the Query Store hint in place for `query_id` 39:

```sql
SELECT query_hint_id, query_id, query_hint_text, last_query_hint_failure_reason, last_query_hint_failure_reason_desc, query_hint_failure_count, source, source_desc 
FROM sys.query_store_query_hints 
WHERE query_id = 39;
```

Finally, remove the hint from `query_id` 39, using [sp_query_store_clear_hints](../system-stored-procedures/sys-sp-query-store-clear-hints-transact-sql.md).  

```sql
EXEC sys.sp_query_store_clear_hints @query_id = 39;
```

## See also

- [sys.query_store_query_hints (Transact-SQL)](../system-catalog-views/sys-query-store-query-hints-transact-sql.md)   
- [sys.sp_query_store_set_hints (Transact-SQL)](../system-stored-procedures/sys-sp-query-store-set-hints-transact-sql.md)   
- [sys.sp_query_store_clear_hints (Transact-SQL)](../system-stored-procedures/sys-sp-query-store-clear-hints-transact-sql.md)   
- [Save an Execution Plan in XML Format](save-an-execution-plan-in-xml-format.md)
- [Display and Save Execution Plans](display-and-save-execution-plans.md)
- [Hints (Transact-SQL) - Query](../../t-sql/queries/hints-transact-sql-query.md)  

## Next steps

- [Best practices with Query Store](best-practice-with-the-query-store.md)
- [Best practices with Query Store hints](query-store-hints-best-practices.md)
- [Monitor performance by using Query Store](../../relational-databases/performance/monitoring-performance-by-using-the-query-store.md)
- [Configure the max degree of parallelism (MAXDOP) in Azure SQL Database](/azure/azure-sql/database/configure-max-degree-of-parallelism)
