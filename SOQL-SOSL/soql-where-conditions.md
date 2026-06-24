# SOQL Where Conditions

## 1. Introduction

### What the WHERE Clause Is
The `WHERE` clause in Salesforce Object Query Language (SOQL) is an optional but fundamental component used to filter the records returned by a query. It acts as a gatekeeper, evaluating each record against a specified condition or set of conditions. Only records that evaluate to `true` are returned.

### Why Filtering Records is Necessary
Querying an entire database table is rarely useful and often impossible due to Salesforce multi-tenant architecture and governor limits. Filtering ensures that:
* **Relevance:** Users and system processes receive only the data they need.
* **Performance:** Less data retrieved means faster execution times and lower memory consumption.
* **Resource Management:** It prevents hitting the strict 50,000 record retrieval limit per synchronous transaction.

### Importance of Efficient Querying
An inefficient query can scan millions of records, degrading performance for the entire org. Efficient filtering leverages indexed fields and selective query design to find the precise needle in the database haystack quickly.

### Real-World Example
In an Automotive CRM, retrieving all `Warranty_Claim__c` records is inefficient. Fetching only claims where `Status__c = 'Pending Approval'` ensures the Service Manager's dashboard loads instantly and stays within governor limits.

---

## 2. Where WHERE Fits in SOQL

### SOQL Query Structure
A standard SOQL query follows a strict syntax order. The `WHERE` clause always follows the `FROM` clause and precedes any grouping or ordering commands.

**Syntax Order:**
`SELECT` → `FROM` → `WHERE` → `WITH` → `GROUP BY` → `HAVING` → `ORDER BY` → `LIMIT` → `OFFSET`

### Execution Order Diagram

```text
+---------------------------------------------------+
|               SOQL Execution Pipeline             |
+---------------------------------------------------+
| 1. FROM     | Identifies the target Object        |
| 2. WHERE    | Filters the raw dataset             |
| 3. GROUP BY | Aggregates the filtered data        |
| 4. HAVING   | Filters the aggregated results      |
| 5. SELECT   | Projects the requested fields       |
| 6. ORDER BY | Sorts the final dataset             |
| 7. LIMIT    | Restricts the number of rows        |
| 8. OFFSET   | Skips the specified number of rows  |
+---------------------------------------------------+
```

---

## 3. Query Execution Process

Understanding how Salesforce processes a `WHERE` clause internally is crucial for enterprise architecture.

### Execution Steps
1.  **SOQL Parsing:** The compiler checks the query for syntax errors.
2.  **Metadata Validation:** Validates that the object and fields exist.
3.  **Security Evaluation:** Applies Object-Level Security (OLS), Field-Level Security (FLS), and Sharing Rules based on the execution context.
4.  **WHERE Clause Evaluation:** The Query Optimizer kicks in.
5.  **Query Optimizer:** Evaluates standard and custom indexes. If the `WHERE` clause is selective, it uses the index to find records. If not, it performs a full table scan.
6.  **Database Execution:** The underlying Oracle/PostgreSQL database executes the optimized SQL equivalent.
7.  **Result Generation:** Data is serialized and returned to the Apex context or API caller.

### Architecture Diagram

```text
[Apex/API Request] -> (SOQL Parser) -> (Metadata Validator) -> (Security/Sharing Engine)
                                                                       |
                                                                       v
[Database Execution] <- (Index Usage/Table Scan) <- (Salesforce Query Optimizer)
         |
         v
[Result Serializer] -> [Returned Records List]
```

---

## 4. Basic WHERE Clause

### Syntax
```sql
SELECT FieldList FROM ObjectName WHERE ConditionExpression
```

### Example
```sql
SELECT Id, Name, VIN__c 
FROM Vehicle__c 
WHERE Status__c = 'Active'
```

### Line-by-Line Breakdown
* `SELECT Id, Name, VIN__c`: The fields to retrieve.
* `FROM Vehicle__c`: The custom object being queried.
* `WHERE Status__c = 'Active'`: The filter. `Status__c` is the field, `=` is the operator, and `'Active'` is the literal string value being matched.

---

## 5. Comparison Operators

Comparison operators define how the field value should relate to the provided value.

| Operator | Name | Description | Example |
| :--- | :--- | :--- | :--- |
| `=` | Equals | Exact match. | `WHERE Status__c = 'Approved'` |
| `!=` | Not Equals | Excludes exact match. | `WHERE Mileage__c != 0` |
| `<>` | Not Equals | Alternative syntax to `!=`. | `WHERE Mileage__c <> 0` |
| `<` | Less Than | Strictly less than. | `WHERE Price__c < 50000` |
| `<=` | Less or Equal | Less than or equal to. | `WHERE Age_Years__c <= 5` |
| `>` | Greater Than | Strictly greater than. | `WHERE Repair_Cost__c > 1000` |
| `>=` | Greater/Equal | Greater than or equal to. | `WHERE Claim_Amount__c >= 500` |

### Business Example
```sql
SELECT Id, Repair_Cost__c FROM Work_Order__c WHERE Repair_Cost__c >= 5000
```
*Retrieves all high-value automotive work orders.*

---

## 6. Logical Operators

Logical operators combine multiple filter conditions.

| Operator | Description | Precedence |
| :--- | :--- | :--- |
| `AND` | True if ALL conditions are true. | High |
| `OR` | True if ANY condition is true. | Medium |
| `NOT` | Reverses the boolean outcome. | Highest |

### Examples
```sql
-- AND Example
SELECT Id FROM Warranty_Claim__c WHERE Status__c = 'Pending' AND Amount__c > 1000

-- OR Example
SELECT Id FROM Dealer__c WHERE Region__c = 'North' OR Region__c = 'South'

-- NOT Example
SELECT Id FROM Vehicle__c WHERE NOT Status__c = 'Sold'
```

---

## 7. Parentheses in Conditions

### Evaluation Order
Parentheses `()` dictate the execution order of complex logical statements, overriding standard operator precedence. Innermost parentheses are evaluated first.

### Complex Filtering Example
```sql
SELECT Id, Name 
FROM Dealership__c 
WHERE (Region__c = 'EMEA' OR Region__c = 'APAC') 
  AND (Annual_Revenue__c > 1000000 AND Is_Active__c = TRUE)
```
*Without parentheses, the `AND` operators would evaluate before the `OR`, resulting in entirely different, likely incorrect, business logic.*

---

## 8. NULL Handling

### Behavior in Salesforce
* `NULL` represents the absence of a value.
* In SOQL, `NULL` is treated as a distinct value.
* **Important:** Empty strings `''` and `NULL` are often treated similarly in Salesforce database logic, but it is best practice to check for `NULL`.

### Examples
```sql
-- Find records with missing VINs
SELECT Id FROM Vehicle__c WHERE VIN__c = NULL

-- Find records that HAVE a VIN
SELECT Id FROM Vehicle__c WHERE VIN__c != NULL
```

---

## 9. String Filtering

### The LIKE Operator
The `LIKE` operator is used for partial string matching. It requires wildcards.

* `%` (Percent): Matches zero or more characters.
* `_` (Underscore): Matches exactly one character.
* **Case Sensitivity:** SOQL string comparisons (including `LIKE`) are **case-insensitive** by default.

### Examples
```sql
-- Starts with 'Ford'
SELECT Id FROM Vehicle__c WHERE Make__c LIKE 'Ford%'

-- Ends with 'Motors'
SELECT Id FROM Dealer__c WHERE Name LIKE '%Motors'

-- Contains 'Engine' (Warning: Non-selective!)
SELECT Id FROM Work_Order__c WHERE Description__c LIKE '%Engine%'

-- Exact character match: 'T_yota' matches 'Toyota'
SELECT Id FROM Vehicle__c WHERE Make__c LIKE 'T_yota'
```

---

## 10. IN Operator

The `IN` and `NOT IN` operators check if a field value matches any value within a specified list or Apex collection.

### Syntax
```sql
SELECT Id FROM Dealer__c WHERE Region__c IN ('NA', 'EMEA', 'APAC')
```

### Set Binding in Apex
Using `IN` with an Apex `Set<Id>` or `List<String>` is the standard pattern for bulkifying code.

```apex
Set<Id> targetAccountIds = new Set<Id>{'001...','001...'};

// Efficient bulkified query
List<Vehicle__c> vehicles = [
    SELECT Id, Name 
    FROM Vehicle__c 
    WHERE Dealer_Account__c IN :targetAccountIds
];
```

---

## 11. INCLUDES and EXCLUDES

These operators are specifically designed for **Multi-Select Picklist** fields. Standard operators like `=` or `IN` do not work correctly on multi-select picklists.

### Logic
* `INCLUDES`: True if the multi-select picklist contains *any* of the specified values.
* `EXCLUDES`: True if the multi-select picklist contains *none* of the specified values.

### Example
```sql
-- Assuming Installed_Features__c is a Multi-Select Picklist
SELECT Id, VIN__c 
FROM Vehicle__c 
WHERE Installed_Features__c INCLUDES ('Sunroof', 'Navigation')
```
*Finds vehicles that have a Sunroof, Navigation, or both.*

---

## 12. Boolean Filtering

Filtering checkbox fields uses the literal boolean values `TRUE` or `FALSE`. Do not use quotes.

### Example
```sql
-- Correct
SELECT Id FROM Warranty_Claim__c WHERE Is_Approved__c = TRUE

-- Incorrect (Will throw syntax error)
SELECT Id FROM Warranty_Claim__c WHERE Is_Approved__c = 'TRUE'
```

---

## 13. Date Filtering

Salesforce provides powerful Date Literals to simplify dynamic date querying without requiring Apex calculation.

### Salesforce Date Literals Table

| Literal | Description |
| :--- | :--- |
| `YESTERDAY` | Starts 12:00:00 AM yesterday, continues 24 hours. |
| `TODAY` | Starts 12:00:00 AM today, continues 24 hours. |
| `TOMORROW` | Starts 12:00:00 AM tomorrow, continues 24 hours. |
| `LAST_WEEK` | Starts 12:00:00 AM on the first day of the previous week. |
| `THIS_MONTH` | Starts 12:00:00 AM on the first day of the current month. |
| `LAST_N_DAYS:n` | The previous N days, not including today. |
| `NEXT_N_DAYS:n` | The next N days, including today. |
| `THIS_YEAR` | The current calendar year. |

### Use Case
```sql
-- Find warranty claims submitted in the last 30 days
SELECT Id FROM Warranty_Claim__c WHERE CreatedDate = LAST_N_DAYS:30
```

---

## 14. DateTime Filtering

When filtering on `DateTime` fields (like `CreatedDate`), Salesforce stores the value in GMT (UTC). 

### Time Zone Behavior
SOQL queries natively handle the conversion between the querying user's timezone and the GMT database value.

### Formatting
If passing a literal string instead of a Date Literal, use ISO 8601 format: `YYYY-MM-DDThh:mm:ssZ`.

```sql
SELECT Id FROM Work_Order__c 
WHERE CreatedDate > 2026-06-01T00:00:00Z
```

---

## 15. Filtering Picklists

### Standard Picklists
Treated essentially as strings. 
```sql
SELECT Id FROM Vehicle__c WHERE Exterior_Color__c = 'Midnight Blue'
```

### Restricted Picklists / Global Value Sets
If a picklist is restricted, querying for a value that is not in the active value set will not throw an error; it simply returns zero records. The database does not validate the literal against the metadata during query execution.

---

## 16. Formula Field Filtering

### Performance Warning
Filtering on Formula Fields is universally discouraged in large data volumes.

* **Why?** Formula fields are calculated dynamically at runtime. Because they don't hold static data in the database, they cannot be natively indexed (unless specially requested via Salesforce Support, and only if deterministic).
* **Result:** A `WHERE` clause on a formula field almost always results in a **Full Table Scan**.

### Best Practice
Instead of:
```sql
-- BAD: Is_High_Value__c is a formula (Amount > 10000)
SELECT Id FROM Claim__c WHERE Is_High_Value__c = TRUE
```
Do this:
```sql
-- GOOD: Query the underlying physical fields directly
SELECT Id FROM Claim__c WHERE Amount__c > 10000
```

---

## 17. Relationship Field Filtering

SOQL allows filtering based on fields of related objects.

### Child-to-Parent Filtering (Dot Notation)
You can traverse up to 5 levels up in SOQL.
```sql
-- Find vehicles associated with a VIP Dealer
SELECT Id, VIN__c 
FROM Vehicle__c 
WHERE Dealer__r.Is_VIP__c = TRUE
```

### Cross-Object Considerations
Filtering on cross-object relationships can be performance-heavy. If the parent object is massive, filtering by parent fields (`Dealer__r.Region__c`) forces the database to join and evaluate both tables.

---

## 18. Bind Variables

Bind variables allow you to inject Apex variables directly into your SOQL string using the colon `:` syntax.

### Types of Bind Variables
* **Primitives:** String, Integer, Id, Date.
* **Collections:** List, Set (Used with `IN`).

### Apex Example
```apex
public List<SAP_Invoice__c> getInvoices(Id accountId, Date minDate) {
    // Both :accountId and :minDate are bind variables
    return [
        SELECT Id, Invoice_Number__c, Total_Amount__c 
        FROM SAP_Invoice__c 
        WHERE Dealer_Account__c = :accountId 
          AND Invoice_Date__c >= :minDate
    ];
}
```

---

## 19. Dynamic SOQL Filtering

Dynamic SOQL involves constructing the SOQL string at runtime and executing it using `Database.query()`.

### The Risk: SOQL Injection
If you concatenate user input directly into a dynamic string, malicious users can modify the query logic.

### Safe Dynamic Pattern (Using `escapeSingleQuotes`)
```apex
public static List<Vehicle__c> searchVehicles(String userProvidedVIN) {
    // Sanitize input to prevent SOQL Injection
    String sanitizedVIN = String.escapeSingleQuotes(userProvidedVIN);
    
    String query = 'SELECT Id, Make__c, Model__c FROM Vehicle__c ';
    query += 'WHERE VIN__c = \'' + sanitizedVIN + '\'';
    
    return Database.query(query);
}
```

### Safe Dynamic Pattern (Using Bind Variables)
*Apex allows bind variables in dynamic SOQL if the variable is in scope!*
```apex
public static List<Vehicle__c> searchVehiclesSafe(String userProvidedVIN) {
    // No concatenation needed, variable is resolved at runtime
    String query = 'SELECT Id FROM Vehicle__c WHERE VIN__c = :userProvidedVIN';
    return Database.query(query);
}
```

---

## 20. Query Optimization

### Selective Queries
A query is **selective** if it uses an index to filter records, returning a small subset of total records. The standard threshold is 10% of the first million records, and 5% thereafter.

### Indexed Fields
Salesforce automatically indexes:
* `Id`, `Name`, `OwnerId`, `CreatedDate`, `SystemModstamp`
* Lookup and Master-Detail fields
* Fields marked as **External ID** or **Unique**

### Query Plan Tool
Architects use the Developer Console Query Plan Tool to verify if a `WHERE` clause will use a `TableScan` or an `Index`. If the cost is > 1.0, the query is unselective and will fail in Large Data Volume (LDV) environments.

---

## 21. Governor Limits

Understanding how filtering impacts limits is crucial.

| Limit Type | Limit | Mitigation via WHERE Clause |
| :--- | :--- | :--- |
| **Max SOQL Queries** | 100 per sync transaction | Bulkify using `WHERE Field IN :collection` to reduce query count. |
| **Max Query Rows** | 50,000 total | Use strict `WHERE` conditions and `LIMIT` clauses. |
| **CPU Time** | 10,000 ms | Efficient `WHERE` clauses push processing to the DB, saving Apex CPU. |
| **Heap Size** | 6 MB | Filtering out unneeded rows prevents list memory bloat. |

---

## 22. Performance Best Practices

1.  **Filter on Indexed Fields:** Always prioritize custom indexed or standard indexed fields in the `WHERE` clause.
2.  **Avoid Leading Wildcards:** `LIKE '%Term'` disables index usage. The database must scan every record. If partial matching is required, use `LIKE 'Term%'`.
3.  **Avoid Negative Operators:** Operators like `!=` and `NOT IN` generally force full table scans. Use `=` and `IN` whenever possible.
4.  **Order of Conditions:** Place the most restrictive condition first in your `AND` statements (though the query optimizer is usually smart enough to handle this, it's good practice).

---

## 23. Security Considerations

A `WHERE` clause does **not** bypass security automatically, but explicit security enforcements are best practice.

### WITH SECURITY_ENFORCED
Throws an exception if the user lacks Object or Field Level permissions to the fields referenced in the `SELECT` or `WHERE` clauses.

```apex
List<Vehicle__c> vList = [
    SELECT Id, VIN__c 
    FROM Vehicle__c 
    WHERE Status__c = 'Active' 
    WITH SECURITY_ENFORCED
];
```

### WITH USER_MODE
Executes the query respecting OLS, FLS, and Sharing Rules natively.

```apex
List<Warranty_Claim__c> claims = [
    SELECT Id FROM Warranty_Claim__c WHERE Amount__c > 1000 WITH USER_MODE
];
```

---

## 24. Common Mistakes

| Mistake | Impact | Solution |
| :--- | :--- | :--- |
| **Hardcoding Ids** | Fails across environments (Sandbox to Prod). | Use logic to retrieve IDs or use Custom Labels/Metadata. |
| **Leading Wildcards** | Causes Full Table Scans. | Use SOSL for full text search, or exact matching. |
| **Query in FOR Loop** | Hits 100 SOQL limit instantly. | Accumulate Ids in a `Set`, query once outside the loop using `IN`. |
| **Ignoring NULLs** | Incorrect calculation of aggregates or bad logic. | Explicitly handle `WHERE Field != NULL`. |

---

## 25. Real Project Scenarios (Automotive CRM)

### 1. Warranty Claims Optimization
**Scenario:** Service agents need to view high-value, pending warranty claims.
```sql
SELECT Id, Claim_Number__c, Amount__c, Dealer__r.Name
FROM Warranty_Claim__c
WHERE Status__c = 'Pending Approval' 
  AND Amount__c > 10000 
  AND Dealer__c != NULL
```
*Why:* We filter on `Status__c` (ideally indexed if it's a heavily used picklist) and exclude orphaned claims (`!= NULL`).

### 2. SAP Integration - Syncing Spare Orders
**Scenario:** A nightly batch job needs to query orders that haven't been synced to SAP.
```sql
SELECT Id, Order_Total__c 
FROM Spare_Order__c 
WHERE SAP_Sync_Status__c = 'Pending' 
  AND CreatedDate = LAST_N_DAYS:3
```
*Why:* Bounding the query with `LAST_N_DAYS` ensures that even if millions of orders exist, we only scan a highly selective, indexed 3-day window.

---

## 26. Enterprise Query Design

For Large Data Volumes (LDV - e.g., 20+ Million `SAP_Invoice__c` records):

* **Custom Indexes:** Request Salesforce Support to create two-column custom indexes if standard indexing isn't enough.
* **Skinny Tables:** For extreme performance, architects can request Skinny Tables, which pre-join highly queried fields without the overhead of standard Salesforce system fields.
* **Deterministic Formulas:** Only use formula fields in `WHERE` clauses if they are marked deterministic and you have explicitly asked Support to index them.
* **Async Processing:** If a query cannot be made selective, push the process to a Batch Apex or Bulk API job where limits are relaxed (up to 50M rows).

---

## 27. Interview Questions & Answers

### Beginner
**Q: Can you use aliases in a WHERE clause?**
**A:** No. Unlike standard SQL, SOQL does not support field aliases in the `WHERE` clause. You must use the actual API name.

### Intermediate
**Q: What is the difference between INCLUDES and IN?**
**A:** `IN` is used for exact matches against a collection of standard values. `INCLUDES` is exclusively used to check if a Multi-Select Picklist contains specific values.

### Advanced
**Q: How do you fix a `System.QueryException: Non-selective query against large object type`?**
**A:** Ensure at least one filter in the `WHERE` clause utilizes an indexed field (like `Id`, `CreatedDate`, or an External ID). Furthermore, ensure the filter reduces the result set to below the selectivity threshold (typically < 10% of records). Avoid `LIKE '%term'` and `!=` operators.

### Architect-Level
**Q: Describe the performance implications of cross-object WHERE filters (e.g., `WHERE Parent__r.Custom_Field__c = 'X'`) on a table with 15 million rows.**
**A:** Cross-object filters force the database backend to execute a `JOIN`. If `Parent__r.Custom_Field__c` is not indexed, this forces a full table scan on the parent, which then maps back to the child, leading to a timeout or limit exception. To optimize, either index the parent field or denormalize the data by copying the parent value to an indexed field on the child object using a before-save flow.

---

## 28. Revision Summary

* **WHERE:** Filters data, reduces rows, saves limits. Always executes before `GROUP BY` and `ORDER BY`.
* **Operators:** `=, !=, <, >, <=, >=, AND, OR, NOT`. Use parenthesis `()` to group logic.
* **LIKE:** `%` matches multiple chars, `_` matches one. Avoid leading `%` (non-selective).
* **IN/NOT IN:** Perfect for bulkified Apex querying with Sets/Lists.
* **INCLUDES/EXCLUDES:** Mandatory for Multi-Select Picklists.
* **Date Literals:** Use `TODAY`, `LAST_N_DAYS:n`, etc., for dynamic date windows without Apex math.
* **Bind Variables:** `:variable` securely passes Apex context into queries.
* **Security:** `WITH SECURITY_ENFORCED` checks FLS/OLS.
* **Optimization Rule #1:** Always strive for selective queries using Indexed Fields (Id, External IDs, Lookups) to survive Large Data Volumes.