# Salesforce Query Optimization – Indexes & Selectivity

## 1. Introduction

### What is Query Optimization?
Query Optimization in Salesforce is the process of structuring SOQL queries and configuring database schema (via indexes and archiving) so that the underlying database engine can retrieve data with maximum efficiency and minimum resource consumption. 

### Why Performance Matters in Salesforce
Salesforce operates in a **multi-tenant architecture**. To prevent a single customer (tenant) from monopolizing database resources, Salesforce enforces strict governor limits (e.g., 50,000 rows returned, 100 synchronous SOQL queries, CPU timeouts). 

**Impact of Slow Queries:**
* **Apex Governor Limit Exceptions:** `System.LimitException: Too many SOQL queries` or `Apex CPU time limit exceeded`.
* **UI Degradation:** Slow Lightning page loads and timeout errors for end-users.
* **Integration Failures:** REST/SOAP API timeouts when querying Large Data Volumes (LDV).

**Business Importance of Optimized SOQL:**
In an enterprise **Automotive CRM**, poorly optimized queries can delay critical operations. If a service advisor searches for a customer's `Vehicle__c` history and the query takes 15 seconds, the customer waits, and dealership throughput drops. Optimized SOQL ensures real-time operational efficiency.

---

## 2. Salesforce Database Architecture

Salesforce uses a metadata-driven, multi-tenant database architecture heavily abstracted from the physical relational database (Oracle/PostgreSQL) beneath it.

### Multi-Tenant Database & Shared Resources
Data for all Salesforce customers resides in the same physical tables (e.g., `MTK` and `MT_Data`). Salesforce uses an `OrgId` column to partition data. Because resources are shared, the database engine cannot simply run raw SQL.

### Metadata-Driven Architecture
When you create a custom object (`Vehicle__c`), you are not creating a new database table. You are adding metadata records. The **Query Execution Engine** must join your tenant data with your metadata to construct the actual SQL executed against the underlying database.

### Architecture Diagram: Database Abstraction
```text
[ Developer's SOQL Query ] -> SELECT Id, Name FROM Vehicle__c WHERE VIN__c = '123'
             |
             v
[ SOQL Parser & Optimizer] -> Validates syntax, security, and selects indexes
             |
             v
[ Physical SQL Generator ] -> Translates SOQL to optimized underlying SQL
             |                (SELECT val0, val1 FROM MT_Data WHERE OrgId = '00D...' AND val2 = '123')
             v
[ Physical RDBMS Engine  ] -> Executes query, retrieves generic strings
             |
             v
[ Data Type Converter    ] -> Casts strings back to Salesforce Types (Dates, Numbers)
             |
             v
[ Result Set (SObjects)  ] -> Returned to Apex/API
```

---

## 3. What is the Salesforce Query Optimizer?

### Definition
The **Salesforce Query Optimizer** is a proprietary database engine component that sits between the SOQL parser and the physical database. Its job is to determine the most efficient way to execute a given SOQL query.

### Internal Working & Decision-Making Process
Because Salesforce uses a multi-tenant structure, standard RDBMS optimizers (like Oracle's Cost-Based Optimizer) don't work effectively out-of-the-box. The Salesforce Query Optimizer:
1.  **Evaluates Filters:** Looks at the `WHERE` clause conditions.
2.  **Checks Indexes:** Determines if any filtered fields have standard or custom indexes.
3.  **Calculates Selectivity:** Uses pre-calculated database statistics to estimate how many records each filter will return.
4.  **Assigns Cost:** Calculates a "cost" for different execution plans (e.g., Table Scan vs. Index Scan).
5.  **Chooses Plan:** Selects the execution plan with the lowest cost.

### Execution Plan Selection Diagram
```text
               [ Filter: Status__c = 'Open' AND CreatedDate = TODAY ]
                                      |
                      +---------------+---------------+
                      |                               |
              [ Check Status__c ]             [ Check CreatedDate ]
              Indexed? No                     Indexed? Yes (Standard)
                      |                               |
                      v                               v
             Cost: Table Scan                 Cost: Index Seek
                      |                               |
                      +--------------+----------------+
                                     |
                       [ Optimizer Chooses: Index Seek ]
                       [ Leading Operation: CreatedDate ]
```

---

## 4. Query Execution Lifecycle

When `List<Warranty_Claim__c> claims = [SELECT Id FROM Warranty_Claim__c];` runs, the following lifecycle occurs:

1.  **SOQL Parsing:** Validates the syntax of the SOQL query.
2.  **Metadata Resolution:** Resolves `Warranty_Claim__c` to the underlying universal data table and specific Org partitions.
3.  **Security Evaluation:** Injects implicit filters based on Organization-Wide Defaults (OWD), Sharing Rules, and FLS (if `WITH SECURITY_ENFORCED` is used).
4.  **Query Optimizer:** Evaluates available indexes and calculates query costs.
5.  **Index Selection:** Selects the most selective index to act as the "Leading Operation".
6.  **Database Scan:** Executes the physical query using the chosen index (or table scan).
7.  **Record Retrieval:** Fetches the raw data.
8.  **Result Construction:** Maps physical generic columns back to SObject fields and returns the List.

---

## 5. What are Indexes?

### Definition
An index is a background database data structure that improves the speed of data retrieval operations on a database table at the cost of slower writes and increased storage space.

### Internal Data Structure (B-Tree Overview)
Salesforce standard and custom indexes are conceptually based on **B-Trees** (Balanced Trees). 
* Instead of scanning millions of records row-by-row (Table Scan), the database traverses a tree-like structure.
* Data is sorted in the tree, allowing logarithmic time complexity `O(log n)` for lookups instead of linear `O(n)`.

### Why Indexes Improve Performance
If an Automotive Org has 10 million `Vehicle__c` records, searching for `VIN__c = 'XYZ789'` without an index requires scanning 10 million rows. With an index on `VIN__c`, the database traverses the B-Tree and finds the specific row pointer in just a few read operations.

---

## 6. Types of Salesforce Indexes

| Index Type | Description |
| :--- | :--- |
| **Standard Indexes** | Automatically created by Salesforce on specific system/standard fields. |
| **Custom Indexes** | Explicitly created indexes. Achieved by marking a field "External ID" or "Unique", or via Salesforce Support. |
| **External ID Indexes** | Fields marked as External ID are automatically indexed. Used for upserts and integrations (e.g., `SAP_Invoice_Id__c`). |
| **Unique Indexes** | Fields marked as Unique are automatically indexed to enforce constraint checks rapidly. |
| **Primary Key (Id)** | The 15/18 character Salesforce `Id` field is automatically indexed and is the fastest lookup mechanism. |
| **Foreign Key (Lookup/MD)** | All Lookup and Master-Detail relationship fields are automatically indexed to facilitate parent-child query joins. |
| **Compound Indexes** | Salesforce natively maintains compound indexes (like `Name` + `RecordTypeId`), but developers can't explicitly create them without Support. |

---

## 7. Fields That Are Automatically Indexed

| Field | Type | Why it is Indexed? |
| :--- | :--- | :--- |
| **Id** | Primary Key | Fundamental identifier for the record. |
| **Name** | Standard | Used in global search, list views, and UI lookups. |
| **RecordTypeId** | System | Extensively used to filter object types. |
| **OwnerId** | System | Crucial for sharing and visibility calculations. |
| **CreatedDate / LastModifiedDate** | System Audit | Heavily used for delta extractions and reporting. |
| **SystemModstamp** | System | Used by internal syncing and external integrations. |
| **Lookup / Master-Detail** | Foreign Key | Necessary for efficient relationship queries and cascaded deletes. |
| **External IDs / Unique** | Custom | Required to enforce uniqueness and optimize UPSERT operations. |

---

## 8. Custom Indexes

If a custom field (e.g., `Warranty_Status__c`) is frequently used in `WHERE` clauses but isn't an External ID or Unique, it is **not indexed by default**.

### How to Request Custom Indexes
1.  **Self-Service:** Mark the field as an **External ID** or **Unique** if business logic permits.
2.  **Salesforce Support:** If the field cannot be unique/external (e.g., a picklist), you must log a case with Salesforce Support requesting a custom index.

### Best Use Cases for Custom Indexes
* Fields used heavily in SOQL `WHERE` clauses, List Views, or Reports.
* Fields with **High Cardinality** (many distinct values).

### Limitations
* **Cannot be indexed:** Multi-select picklists, Text Area (Long/Rich), un-deterministic Formula fields, and Encrypted (classic) fields.
* **Over-indexing:** Too many indexes slow down DML operations (`INSERT`, `UPDATE`, `DELETE`) because every index must be updated.

---

## 9. What is Query Selectivity?

### Definition
Query selectivity refers to how well a query filter narrows down the total number of records in an object. A query is **selective** if it returns a small percentage of total records using an indexed field.

### Selective vs Non-Selective Queries
* **Selective Query:** Uses an indexed field in the `WHERE` clause and falls *below* the selectivity threshold (e.g., retrieves 5,000 out of 10,000,000 records). 
* **Non-Selective Query:** Does not use an indexed field, OR uses an indexed field but requests too large a percentage of the total data (e.g., retrieves 4,000,000 out of 10,000,000 records).

Salesforce will force a **Full Table Scan** for non-selective queries. If the object has more than 100,000 to 200,000 records, a non-selective query may time out.

---

## 10. Query Selectivity Thresholds

For the Query Optimizer to actually *use* an index, the filter must meet these thresholds:

### Standard Indexed Field Thresholds
* Must target **less than 30%** of the first 1 million records.
* Must target **less than 15%** of records after the first 1 million.
* *Maximum limit:* The filter cannot target more than **1 million** records total.

### Custom Indexed Field Thresholds
* Must target **less than 10%** of the first 1 million records.
* Must target **less than 5%** of records after the first 1 million.
* *Maximum limit:* The filter cannot target more than **333,333** records total.

*(Note: If a query exceeds these limits, the Optimizer abandons the index and performs a Table Scan).*

---

## 11. Selective Queries

### Characteristics
* Uses `Id`, Name, Lookup, or Custom Indexed fields.
* Filters out the vast majority of records.
* Executes incredibly fast, avoiding CPU and Governor limits.

**Example (Highly Selective):**
```sql
-- VIN__c is marked as an External ID (Indexed)
-- This targets exactly 1 record out of millions.
SELECT Id, Model__c FROM Vehicle__c WHERE VIN__c = '1G1RC6E45FU123456'
```

---

## 12. Non-Selective Queries

### Characteristics
* Filters on non-indexed fields.
* Filters on indexed fields but uses negative operators (`!=`, `NOT IN`) or leading wildcards (`LIKE '%abc'`).
* Retrieves more than the threshold percentage of total table data.

**Example (Non-Selective):**
```sql
-- Status__c is NOT indexed.
-- Vehicle object has 5 Million records.
-- Result: FULL TABLE SCAN. System.QueryException!
SELECT Id FROM Vehicle__c WHERE Status__c = 'Active'
```

---

## 13. Query Cost

Query cost is a numerical value calculated by the Salesforce Optimizer. 

* **Cost < 1:** The query is selective. The index will be used.
* **Cost >= 1:** The query is non-selective. A full table scan will occur.

If an execution plan has a cost of `0.2`, it means the filter targets 20% of the selectivity threshold. If the cost is `1.5`, it exceeds the threshold by 50% and fails the selectivity check.

---

## 14. Query Plan Tool

### What it is
The Query Plan Tool is a feature in the Salesforce Developer Console that exposes the Optimizer's internal execution plans.

### How to use it
1.  Open Developer Console.
2.  Go to **Help > Preferences** -> Check **Enable Query Plan**.
3.  Go to the **Query Editor** tab.
4.  Enter your SOQL and click the **Query Plan** button.

### Fields in the Query Plan

| Field | Meaning |
| :--- | :--- |
| **Leading Operation** | The strategy used (e.g., `Index`, `TableScan`, `Sharing`). |
| **Cost** | The relative cost (Must be < 1 for an Index to be used). |
| **Cardinality** | The estimated number of records the Leading Operation will return. |
| **sObject Cardinality** | Total records in the object. |
| **Notes** | Explains why an index couldn't be used (e.g., `Not indexed`, `Too many rows`). |

---

## 15. Cardinality

### Definition
Cardinality refers to the uniqueness of data values contained in a column.

* **High Cardinality:** A field with mostly unique values. (e.g., `VIN__c`, `Email`, `Phone`). These make **excellent** candidates for indexes.
* **Low Cardinality:** A field with few distinct values. (e.g., `Gender__c`, `IsActive__c`, `Status__c` with 3 picklist values). Indexes on low cardinality fields are rarely useful because querying 'Active' might return 80% of the table, exceeding threshold limits instantly.

---

## 16. Histograms (Concept)

How does the Optimizer know how many records `Status = 'Closed'` will return before running the query? 

Salesforce maintains background **Histograms** (data statistics). A histogram tracks data distribution for indexed fields. Periodically, the database takes a snapshot of the table to know that "70% of Work Orders are Closed, 30% are Open." The Optimizer uses these histograms to instantly calculate the `Cost` without running the query.

---

## 17. Large Data Volumes (LDV)

### Definition
LDV usually refers to objects with millions of records (typically > 5 million). In Automotive CRMs, objects like `Warranty_Claim_Line_Item__c` or `Telematics_Data__c` easily hit LDV territory.

### Challenges
With LDV, standard queries that worked in a sandbox will fail in Production due to timeout errors. Sharing calculations become extremely slow (Sharing table bloat).

### LDV Query Design
1.  **Always filter by indexed fields.**
2.  **Filter by standard dates** (`CreatedDate` / `SystemModstamp`) to partition data access.
3.  **Archiving Strategy:** Move old records to Big Objects, Data Lakes, or external databases.

---

## 18. Skinny Tables

### What are Skinny Tables?
A Skinny Table is a custom, under-the-hood table created by Salesforce Support to solve extreme LDV query performance issues.

Normally, Standard fields and Custom fields reside in two separate underlying database tables (e.g., `Account` and `Account_Custom_Data`). Querying both requires an invisible database join. A Skinny Table combines frequently used standard and custom fields into **one single physical table**, bypassing the join.

### Benefits & Limitations
* **Benefits:** Massively speeds up read operations, reporting, and list views. Excludes soft-deleted records (`isDeleted = false`), reducing table size.
* **Limitations:** Can only contain 100 fields. If you add a new field to your object, you must contact Support to add it to the Skinny Table. Does not span across multiple objects.

---

## 19. Formula Fields and Performance

### Why formula fields may not be indexed
Formula fields are calculated at runtime. Therefore, they do not exist as physical data in the database, meaning they **cannot be indexed by default**.

```sql
-- TERRIBLE PERFORMANCE IF LDV
SELECT Id FROM Invoice__c WHERE Days_Overdue__c > 30 
```

### Optimization Strategies
1.  **Deterministic Formulas:** If a formula is deterministic (doesn't use `TODAY()`, `NOW()`, or cross-object fields), you can contact Salesforce Support to create a custom index on it.
2.  **Workflow/Trigger Backup:** Instead of a formula, use a Flow or Apex Trigger to calculate the value and store it in a standard, indexable Number/Text field.

---

## 20. Relationship Query Performance

### Parent-to-Child (Subqueries)
```sql
SELECT Id, Name, (SELECT Id FROM Work_Orders__r) FROM Account
```
* **Risk:** Can cause `Query Timeout` if a parent has tens of thousands of child records (Data Skew). Use indexing on the Parent object.

### Child-to-Parent
```sql
SELECT Id, Account.Name FROM Work_Order__c
```
* **Optimization:** Highly efficient because lookup fields (`AccountId`) are automatically indexed. 

### Relationship Selectivity
If you filter on cross-object fields (`WHERE Account.Industry = 'Automotive'`), the Optimizer evaluates the selectivity of the child object *first*, then joins the parent. It is often more efficient to query the parent first, extract IDs, and then query the child using `IN :parentIds`.

---

## 21. SOQL vs SOSL Performance

| Feature | SOQL (Salesforce Object Query Language) | SOSL (Salesforce Object Search Language) |
| :--- | :--- | :--- |
| **Engine** | Database execution engine (RDBMS). | Search index engine (Apache Lucene based). |
| **Use Case** | Exact matches, specific objects, highly structured queries. | Text/String searches across multiple objects. |
| **Performance** | Extremely fast if indexed fields are used. Slow for text matching (`LIKE`). | Extremely fast for text and partial word matching. |
| **Leading Wildcards** | Very slow (`LIKE '%abc'`). | Supported natively and highly optimized. |
| **Limits** | 50,000 rows returned per transaction. | 2,000 rows returned per transaction. |

---

## 22. Governor Limits (SOQL Specific)

| Limit Description | Synchronous Limit | Asynchronous Limit |
| :--- | :--- | :--- |
| **Total number of SOQL queries** | 100 | 200 |
| **Total number of records retrieved by SOQL** | 50,000 | 50,000 |
| **Total number of SOSL queries** | 20 | 20 |
| **Maximum CPU Time** | 10,000 ms (10s) | 60,000 ms (60s) |
| **Query Timeout limits** | 120 seconds | 120 seconds |

---

## 23. Security and Query Optimization

### How Security Impacts Performance
When `WITH SECURITY_ENFORCED` or `WITH USER_MODE` is used, the Optimizer must inject OWD, Sharing Rule, and FLS checks into the SQL `WHERE` clause. 

If an object is set to **Private** OWD, the Optimizer might choose the `Sharing` index as the Leading Operation instead of your custom index, because it determines that restricting the user to "only records they own" is the fastest way to shrink the result set.

*(Architect Note: A query that is fast for an Admin might result in a CPU Timeout for a Standard User because the database has to calculate complex sharing hierarchy joins).*

---

## 24. Common Performance Problems

1.  **Leading Wildcards (`LIKE '%abc'`):** Bypasses the B-Tree index completely, forcing a table scan.
2.  **SOQL Inside Loops:** The #1 cause of `System.LimitException: Too many SOQL queries: 101`.
3.  **OR Conditions:** Prevents efficient index usage. Splitting into multiple queries and merging lists in Apex is sometimes faster.
4.  **Negative Operators (`!=`, `NOT IN`, `EXCLUDES`):** Indexes locate what *is* there, not what *isn't*. Forces table scans.
5.  **Functions in WHERE clauses (`CALENDAR_YEAR(CreatedDate)`):** Invalidates the index on `CreatedDate`. Use bounded ranges instead (`CreatedDate >= 2023-01-01T00:00:00Z AND CreatedDate < 2024-01-01T00:00:00Z`).

---

## 25. Performance Best Practices

1.  **Always filter on Indexed Fields:** Use `Id`, `CreatedDate`, or custom External IDs.
2.  **Avoid SELECT * equivalents:** Query only the fields you strictly need to reduce Heap Size.
3.  **Bulkify Apex:** Collect IDs in a `Set<Id>` and query once using `WHERE Id IN :setIds`.
4.  **Query Caching:** Use Platform Cache or static variables for static data (like RecordType IDs) instead of querying them repeatedly.
5.  **Handle Data Skew:** Avoid queries that retrieve parents with > 10,000 children.

---

## 26. Real Project Scenarios (Automotive CRM)

### Scenario 1: Optimizing Warranty Claim Searches
**Problem:** Service Advisors search for Claims by `Status__c` (Low Cardinality) and `Region__c`. Queries timeout.
**Solution:** `Status__c` and `Region__c` are non-selective. We created a composite text field `Region_Status__c` (e.g., `NA-Open`), marked it as an External ID, and populated it via Before-Save Flow. Querying the indexed `Region_Status__c` solved the timeout.

### Scenario 2: SAP Integration Lookups (Invoices)
**Problem:** UPSERTing 10,000 Invoices via REST API fails due to SOQL limits when looking up related `Dealer__c` records.
**Solution:** Indexed `SAP_Dealer_Code__c` as an External ID on the `Dealer__c` object. Instead of querying for IDs in Apex, mapped the relationship natively in the REST payload using the External ID.

### Scenario 3: Large Work Order Queries
**Problem:** A nightly batch job querying `Work_Order__c` where `Requires_Followup__c = true` times out (10 million records total).
**Solution:** Formula fields cannot be indexed easily. Replaced the formula with a boolean field updated by Apex. Requested Salesforce Support to index the boolean. Modified batch query to process data in chunks using `CreatedDate`.

---

## 27. Debugging Slow Queries

1.  **Query Plan Tool (Dev Console):** Use this first to check `Cost` and `Leading Operation`.
2.  **Debug Logs:** Set `Profiling` and `Database` logging levels to `FINEST` to review execution time in milliseconds.
3.  **Workbench:** Useful for running REST queries to test API response times without UI overhead.
4.  **VS Code SOQL Builder:** Excellent for running iterative queries locally against sandbox environments.

---

## 28. Common Mistakes & Solutions

| Mistake | Solution |
| :--- | :--- |
| **Filtering on Formula Fields** | Materialize the data into a standard indexed field via Flow/Triggers. |
| **Using `!=` (Not Equals)** | Change logic to use `IN ('Value1', 'Value2')` if the picklist is finite. |
| **Hardcoding IDs** | Unmaintainable across environments. Query by `DeveloperName` (which is indexed on metadata objects). |
| **SOQL in `FOR` loop** | Bulkify using Collections (`List`, `Map`, `Set`). |

---

## 29. Interview Questions & Answers

**Beginner: What is the difference between SOQL and SOSL?**
*Answer:* SOQL is for exact querying using SQL-like syntax on specific objects. SOSL is a full-text search engine across multiple objects simultaneously.

**Intermediate: How do you fix a query that times out due to Large Data Volumes?**
*Answer:* Ensure the `WHERE` clause filters on an indexed field. If it already does, ensure it meets the selectivity threshold (<30% of records). If using a formula, move logic to an actual indexed field. If wildcards or negative operators are used, remove them. 

**Advanced: What is the Query Plan Tool and what does 'Cost' indicate?**
*Answer:* It's a Dev Console tool to view the Optimizer's execution plan. Cost represents the relative performance of the query plan. A cost `< 1` means the query is selective and an index will be used. A cost `>= 1` indicates a full table scan.

**Architect: Explain how OWD and Sharing Rules can unexpectedly impact SOQL performance.**
*Answer:* The Optimizer treats sharing rules as a hidden `WHERE` clause filter. In a Private OWD model, the system must join the object table with its associated `Share` table. If the user owns millions of records, the `Share` table bloats (Data Skew), and the resulting database join can cause severe performance degradation and timeouts.

---

## 30. Revision Summary

* **Query Optimizer:** Internal engine that chooses the lowest-cost execution plan based on thresholds and histograms.
* **Indexes:** B-Tree structures enabling `O(log n)` lookups.
* **Standard Indexes:** `Id`, `Name`, `CreatedDate`, Lookups.
* **Custom Indexes:** `External ID`, `Unique`, or via Support request.
* **Selectivity:** Query must target <30% (Standard Index) or <10% (Custom Index) of the first 1M records to use an index.
* **Query Plan Tool:** Use to verify Index vs TableScan. Target Cost < 1.
* **Cardinality:** High cardinality (unique values) = good for indexes. Low cardinality (few values) = bad.
* **Skinny Tables:** Combines standard/custom fields, ignores soft-deleted rows, speeds up LDV, requires Support intervention.
* **Best Practices:** Avoid leading wildcards, negative operators, unindexed formulas in WHERE clauses, and always Bulkify.