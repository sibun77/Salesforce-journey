# SOQL Basics – SELECT Queries & Basic Querying

## 1. Introduction

### What is SOQL?
Salesforce Object Query Language (SOQL) is the native query language used to search your organization's Salesforce data for specific information. Conceptually similar to the `SELECT` statement in widely used SQL (Structured Query Language), SOQL is optimized specifically for the Salesforce multi-tenant platform.

### Why SOQL Exists
Salesforce created SOQL because the underlying architecture of Salesforce is not a standard relational database. It is a multi-tenant, metadata-driven architecture. Standard SQL statements (`INSERT`, `UPDATE`, `DELETE`) are handled via Data Manipulation Language (DML) in Apex to ensure business logic, triggers, and validation rules run correctly. SOQL exists purely for **reading** data.

### Difference Between SOQL and SQL
While SQL is designed for standard relational database management systems (RDBMS) where developers define schemas with raw DDL (Data Definition Language), SOQL is tightly coupled with Salesforce's Object metadata. 
* SOQL has no `SELECT *` (to prevent blind querying of hundreds of custom fields, safeguarding platform performance).
* SOQL lacks `JOIN` statements (relationships are traversed via dot-notation or subqueries).
* SOQL handles multi-tenant security automatically under the hood.

**Real-world Example:**
Instead of joining `Vehicle` and `Dealer` tables with a SQL `JOIN ON`, SOQL uses `Dealer__r.Name` to seamlessly pull data from the parent object.

---

## 2. Salesforce Database Architecture

### Multi-Tenant Database
Salesforce operates on a multi-tenant architecture, meaning a single database instance serves multiple customers (tenants). To prevent one tenant's queries from monopolizing resources, Salesforce enforces strict governor limits and optimizes data retrieval dynamically.

### Objects as Tables, Records as Rows, Fields as Columns
* **Objects (Tables):** Represent entities (e.g., `Account`, `Vehicle__c`).
* **Records (Rows):** Instances of objects (e.g., a specific Dealer or Vehicle).
* **Fields (Columns):** Attributes of an object (e.g., `VIN_Number__c`).

### Metadata-Driven Architecture
When you create a `Vehicle__c` object, Salesforce does not create a physical table named `Vehicle__c`. Instead, it stores the definition in metadata and stores the data in large, generic underlying database tables (e.g., `CustomEntityData`). SOQL abstracts this complexity, allowing developers to query `Vehicle__c` as if it were a distinct physical table.

**Architecture Diagram (Conceptual):**
```text
[ Developer SOQL Query ] --> [ SOQL Parser ] --> [ Metadata Cache (Resolves Vehicle__c) ] --> [ Query Optimizer ] --> [ Physical Database (Oracle) ]
```

---

## 3. What is SOQL?

### Definition & Purpose
SOQL is an object-oriented query language tailored for the Salesforce platform. Its primary purpose is to retrieve records securely and efficiently from the Salesforce database for use in Apex code, Lightning Web Components (LWC), Visualforce, or via REST/SOAP APIs.

### Syntax & Metadata Awareness
Because SOQL is metadata-aware, it implicitly understands relationships. It checks field-level security and sharing rules (depending on execution context) before returning data.

**Why it only queries Salesforce objects:**
SOQL runs against the Salesforce application layer, not directly against the underlying database. It relies on the Salesforce metadata engine to translate object names (`Vehicle__c`) into the underlying physical partitions.

---

## 4. SOQL vs SQL

| Feature | SQL (Traditional RDBMS) | SOQL (Salesforce) |
| :--- | :--- | :--- |
| **Syntax** | `SELECT * FROM Table` | `SELECT Id, Name FROM Object` (No `*` wildcard) |
| **Data Modification** | `INSERT`, `UPDATE`, `DELETE` | Read-only. Modification done via Apex DML. |
| **JOIN Support** | `INNER JOIN`, `OUTER JOIN` | Handled via Relationship Queries (Parent-to-Child, Child-to-Parent). |
| **Relationships** | Requires foreign key mapping | Pre-defined in Salesforce via Lookup/Master-Detail metadata. |
| **Security** | DBA controls | OLS, FLS, Sharing Rules, `USER_MODE` natively integrated. |
| **Performance** | Database tuning/indexing | Governed by Salesforce Query Optimizer; custom indexing via Support. |
| **Governor Limits**| Infrastructure dependent | Strict limits (e.g., 100 queries/transaction, 50,000 rows). |

---

## 5. SELECT Statement

The `SELECT` statement in SOQL defines what data you want to retrieve.

* **`SELECT`**: Specifies the fields to return (e.g., `SELECT Id, Name`).
* **`FROM`**: Specifies the object to query (e.g., `FROM Vehicle__c`).
* **`WHERE`**: Filters the records based on specific criteria (e.g., `WHERE Mileage__c < 10000`).
* **`LIMIT`**: Restricts the maximum number of records returned.
* **`ORDER BY`**: Sorts the result set by one or more fields.
* **`OFFSET`**: Skips a specified number of rows before returning the result (used for pagination).

**Syntax Diagram:**
```sql
SELECT fieldList
FROM objectType
[WHERE conditionExpression]
[ORDER BY fieldName [ASC|DESC] [NULLS FIRST|LAST]]
[LIMIT number_of_rows]
[OFFSET number_of_rows_to_skip]
```

---

## 6. Basic Query Syntax

```sql
SELECT Id, Name, VIN_Number__c 
FROM Vehicle__c
```

**Step-by-step Execution Breakdown:**
1. **`SELECT`**: The engine prepares to project three fields: `Id`, `Name`, and `VIN_Number__c`.
2. **`FROM Vehicle__c`**: The engine accesses the custom object metadata for `Vehicle__c`.
3. **Execution**: The database retrieves all vehicle records, extracts the requested fields, and returns a list of `Vehicle__c` sObjects.

---

## 7. Query Execution Process

When Apex executes SOQL, it follows a rigorous path:

1. **Apex Execution Context**: An inline SOQL query `[SELECT Id FROM Account]` is triggered.
2. **Query Parser**: Validates syntax against object metadata (throws compilation errors if fields don't exist).
3. **Security Checks**: Verifies if the context requires sharing rules (`with sharing`) or field-level security (`WITH USER_MODE`).
4. **Query Optimizer**: Analyzes the `WHERE` clause. Determines if an index can be used (e.g., querying by `Id` or an External ID). Checks the *Selectivity* of the query.
5. **Database Engine**: Translates SOQL to optimized physical SQL, fetches data from the multi-tenant tables.
6. **Result Generation**: Packages the returned rows into an Apex `List<sObject>` and returns it to the transaction.

---

## 8. Querying Standard Objects

Standard objects are built-in Salesforce tables. Common use cases:
* **Account**: Storing Dealerships or B2B Customers.
* **Contact**: Storing Dealer Employees.
* **Opportunity**: Storing Sales Deals.
* **Case**: Storing Warranty Claims or Customer Service Tickets.

**Example:**
```sql
SELECT Id, Name, Type, BillingState FROM Account WHERE Type = 'Dealership'
```

---

## 9. Querying Custom Objects

Custom objects are user-defined. They end in `__c`. Custom fields on standard or custom objects also end in `__c`.

**Example:**
```sql
SELECT Id, Claim_Amount__c, Status__c, Vehicle__c 
FROM Warranty_Claim__c 
WHERE Status__c = 'Pending Approval'
```

---

## 10. Selecting Fields

You must explicitly list every field you want to query.
* **Single Field:** `SELECT Id FROM Vehicle__c`
* **Multiple Fields:** `SELECT Id, Name, VIN_Number__c FROM Vehicle__c`
* **Standard Fields on Custom Objects:** `Id`, `Name`, `CreatedDate`, `OwnerId`.
* **Custom Fields:** Must include the `__c` suffix.

**Best Practice:** Only select fields you actually need to conserve heap size and improve query performance.

---

## 11. Using Aliases

Aliases in SOQL are primarily used in **Aggregate Queries** to rename the output columns. They cannot be used arbitrarily to rename fields in standard `SELECT` statements the way they can in SQL.

**Example (Valid in SOQL):**
```sql
SELECT Vehicle__r.Model__c, SUM(Claim_Amount__c) TotalClaims
FROM Warranty_Claim__c
GROUP BY Vehicle__r.Model__c
```
*`TotalClaims` is the alias for `SUM(Claim_Amount__c)` and can be accessed in Apex via `(Decimal) aggregateResult.get('TotalClaims')`.*

---

## 12. Querying Records

* **Query All Records:** `SELECT Id, Name FROM Vehicle__c` (Dangerous without `LIMIT` or `WHERE`).
* **Query Specific Fields:** Tailoring the `SELECT` list.
* **Query Large Datasets:** Use `Batch Apex` or `SOQL For Loops` to avoid heap limit exceptions when retrieving over 50,000 records.

---

## 13. Relationship Query Introduction

Salesforce handles joins via relationships.
* **Child-to-Parent (Look up):** Uses dot-notation (`__r`).
  ```sql
  SELECT Id, Name, Vehicle__r.VIN_Number__c FROM Warranty_Claim__c
  ```
* **Parent-to-Child (Subquery):** Uses a nested SELECT.
  ```sql
  SELECT Id, Name, (SELECT Id, Status__c FROM Warranty_Claims__r) FROM Vehicle__c
  ```

---

## 14. Aggregate Query Introduction

Used for summarizing data. Returns an `AggregateResult` object instead of the standard sObject.
* `COUNT(Id)`: Number of records.
* `SUM(Amount__c)`: Total value.
* `AVG(Mileage__c)`: Average value.
* `MIN(CreatedDate)`, `MAX(CreatedDate)`: Smallest/Largest values.

---

## 15. SOQL Governor Limits

Salesforce enforces limits to protect the multi-tenant architecture.

| Limit Type | Synchronous Apex Limit | Asynchronous Apex (Batch/Future) |
| :--- | :--- | :--- |
| **Total SOQL Queries** | 100 | 200 |
| **Total Rows Retrieved** | 50,000 | 50,000 |
| **Total Aggregate Queries** | 300 | 300 |
| **Query Timeout** | 120 seconds | 120 seconds |
| **Heap Size Limit** | 6 MB | 12 MB |

*If you query 50,001 records, Salesforce throws a `System.LimitException: Too many query rows: 50001`.*

---

## 16. SOQL Security

By default, Apex runs in **System Mode**, bypassing user permissions. To enforce security:

* **`WITH SECURITY_ENFORCED` (Legacy approach):** Checks Object and Field Level Security.
* **`WITH USER_MODE` (Modern approach):** Enforces OLS, FLS, and Sharing Rules dynamically. Respects polymorphism and polymorphic fields better.
* **`WITH SYSTEM_MODE`:** Explicitly bypasses security.
* **Class-level Sharing:** `public with sharing class` enforces record-level sharing rules.

**Example:**
```apex
List<Vehicle__c> vehicles = [SELECT Id, Name FROM Vehicle__c WITH USER_MODE];
```

---

## 17. Query Optimization

The Salesforce **Query Optimizer** builds query execution plans based on indices and statistics.
* **Selective Queries:** A query is selective if it uses an indexed field in the `WHERE` clause (e.g., `Id`, `Name`, `External ID` fields, or custom indexed fields) and filters out a large percentage of records.
* **Cost:** The Optimizer assigns a cost to different execution plans. The threshold for a standard index to be used is generally that the filter must return less than 10% of the total records (up to 333k records).
* **Query Plan Tool:** Available in the Developer Console to visualize query cost.

---

## 18. Common Query Mistakes

1. **Simulating `SELECT *`:** Querying all fields wastes heap space and slows down the transaction.
2. **SOQL Inside Loops:** The cardinal sin of Salesforce. Will quickly hit the 100 SOQL query limit.
   * *Solution:* Collect IDs in a `Set<Id>` and query once outside the loop.
3. **Unselective Filters on LDV:** Querying `WHERE Status__c = 'Closed'` without an index on a table of 5 million records causes full table scans and timeouts.
4. **Hardcoded IDs:** Querying `WHERE RecordTypeId = '012x0000000ABCD'` breaks between environments. Use `Schema.SObjectType` describes instead.

---

## 19. Best Practices

* **Query Only Required Fields:** Save CPU time and Heap size.
* **Bulk-Safe Querying:** Always design queries assuming they will handle 200 records simultaneously.
* **Use Bind Variables:** Protect against SOQL injection and make queries cleaner (e.g., `WHERE Dealer__c IN :dealerIds`).
* **Map Processing:** Put SOQL results directly into a Map for easy retrieval: 
  `Map<Id, Vehicle__c> vMap = new Map<Id, Vehicle__c>([SELECT Id FROM Vehicle__c]);`
* **Architect-Level Guidance:** On objects > 1M records, design queries with indexed filters. Work with DBAs to request Custom Indexes on frequently filtered fields.

---

## 20. Real Project Scenarios: Automotive CRM

**Scenario: A dealer needs to view all rejected warranty claims for specific vehicles.**

```sql
SELECT Id, Name, Claim_Amount__c, Failure_Reason__c, Vehicle__r.VIN_Number__c, Vehicle__r.Model__c
FROM Warranty_Claim__c
WHERE Status__c = 'Rejected' 
  AND Vehicle__r.Dealer__c = :currentDealerId
WITH USER_MODE
LIMIT 500
```
* **Why it's written this way:** * Uses Parent-to-Child dot notation (`Vehicle__r.VIN_Number__c`) to avoid separate queries.
  * Filters dynamically using `:currentDealerId`.
  * Protects data with `WITH USER_MODE`.
  * Includes a `LIMIT` to protect performance.

---

## 21. Apex + SOQL Examples

**Production-Quality Apex Method:**

```apex
public inherited sharing class WarrantyClaimService {
    
    /**
     * Retrieves high-value warranty claims for a specific dealer.
     * @param dealerId The Account Id of the Dealership.
     * @return List of Warranty Claims.
     */
    public static List<Warranty_Claim__c> getHighValueClaims(Id dealerId) {
        // Validation to prevent bad queries
        if (String.isBlank(dealerId)) {
            return new List<Warranty_Claim__c>();
        }

        // Dynamic Binding variable
        Decimal threshold = 5000.00;

        // Safe, bulkified, user-mode query
        List<Warranty_Claim__c> highValueClaims = [
            SELECT Id, Name, Claim_Amount__c, Status__c,
                   Vehicle__r.VIN_Number__c, Vehicle__r.Name
            FROM Warranty_Claim__c
            WHERE Vehicle__r.Dealer__c = :dealerId
              AND Claim_Amount__c >= :threshold
            WITH USER_MODE
            ORDER BY Claim_Amount__c DESC
        ];
        
        return highValueClaims;
    }
}
```
* **Line-by-line:**
  * `inherited sharing`: Ensures the class respects the caller's sharing context.
  * Input validation protects against querying null IDs.
  * The SOQL query uses dot-notation to filter on the Parent (`Vehicle__r.Dealer__c`).
  * `WITH USER_MODE` respects user permissions.

---

## 22. Performance Considerations

* **Large Data Volumes (LDV):** Once a table exceeds 1-2 million rows, unindexed queries will timeout. 
* **Selectivity Thresholds:** An index is only used if the filter returns < 10% of records (up to 333,333). If a query is not selective, Salesforce performs a full table scan.
* **Skinny Tables:** For extreme LDV, architects can request Salesforce Support to create "Skinny Tables" which combine standard and custom fields into a single database table without joins, drastically speeding up read times.
* **Heap Usage:** Do not load large query results into memory. Use the **SOQL For Loop** to chunk heap usage:
  ```apex
  for (Vehicle__c v : [SELECT Id, Name FROM Vehicle__c]) {
      // Processes records in chunks of 200, managing heap efficiently.
  }
  ```

---

## 23. Interview Questions & Answers

**Beginner:**
* *Q: What is the difference between SOQL and DML?*
  * A: SOQL is for retrieving (reading) data. DML is for modifying (inserting, updating, deleting) data.

**Intermediate:**
* *Q: How do you bypass governor limits for retrieving more than 50,000 rows?*
  * A: In synchronous Apex, you cannot. You must use Batch Apex, which allows querying up to 50 million records via the `Database.QueryLocator`.

**Advanced:**
* *Q: Explain the difference between `WITH SECURITY_ENFORCED` and `WITH USER_MODE`.*
  * A: `WITH SECURITY_ENFORCED` checks object and field-level security but was introduced before the more robust user-mode. `WITH USER_MODE` (introduced later) ensures polymorphic fields are handled securely, tracks execution deeply in the engine, and evaluates sharing rules directly in the query context.

**Architect-Level:**
* *Q: You are querying an object with 10 million records. Your query times out despite filtering on an indexed field. Why?*
  * A: The query is likely hitting the selectivity threshold. If the filter condition matches more than 10% of the total records (or > 333,333 records), the Query Optimizer discards the index and attempts a full table scan, resulting in a timeout. You must add additional restrictive filters, archive data, or use a custom index on a more selective field.

---

## 24. Revision Summary

* **SOQL:** Metadata-aware query language for Salesforce. Read-only.
* **Syntax:** `SELECT [fields] FROM [object] WHERE [conditions]`. No `SELECT *`.
* **Execution:** Parsed, checked for security, optimized, then executed against the multi-tenant DB.
* **Limits:** 100 queries/transaction, 50k rows/transaction.
* **Security:** Use `WITH USER_MODE` and `with sharing` classes.
* **Optimization:** Ensure `WHERE` clauses are selective using indexed fields.
* **Best Practice:** Never put SOQL inside a `for` loop. Always query only needed fields. Use SOQL for-loops for large lists to manage heap limit.