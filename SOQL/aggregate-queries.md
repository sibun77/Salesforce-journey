# Aggregate Queries

## 1. Introduction

### What Aggregate Queries are
In Salesforce, SOQL (Salesforce Object Query Language) Aggregate Queries are specialized database operations designed to group, summarize, and calculate data from multiple records, rather than retrieving individual record details. They utilize aggregate functions like `SUM()`, `MAX()`, and `COUNT()` to roll up data mathematically.

### Why Salesforce provides Aggregate Queries
Salesforce provides these queries to offload mathematical and grouping operations to the database layer. Instead of querying thousands of records into Apex and iterating through them to calculate a total (which consumes CPU time and Heap space), the database performs the heavy lifting and returns a single summarized result.

### Business value of summarized data
Summarized data drives business intelligence. Executives do not need to see 10,000 individual warranty claims; they need to see the *Total Warranty Cost per Region*. Aggregate queries transform raw operational data into actionable strategic metrics, enabling dashboards, KPI tracking, and enterprise reporting.

### Difference between record-level queries and aggregate queries
* **Record-Level Query:** `SELECT Id, Amount__c FROM Warranty_Claim__c` returns a list of individual `Warranty_Claim__c` sObjects.
* **Aggregate Query:** `SELECT SUM(Amount__c) FROM Warranty_Claim__c` returns an `AggregateResult` object containing a single calculated number.

### Real-world Examples
In an Automotive CRM:
* **Record-Level:** Show me the details of John's warranty claim for his transmission.
* **Aggregate:** Show me the total cost of all transmission warranty claims submitted by the Dallas Dealership in Q3.

---

## 2. Salesforce Query Architecture

### Standard Queries
Standard SOQL queries target the operational database to retrieve raw sObject records. They are optimized for retrieving specific fields for UI display or transaction processing.

### Aggregate Queries
Aggregate queries utilize the database's grouping and analytical engine. They process rows at the database level, collapse them based on defined criteria, and project mathematical summaries into `AggregateResult` arrays.

### Reporting Engine
Salesforce Reports are a metadata-driven UI layer built on top of complex, dynamic aggregate queries. When a user groups a report by "Dealer" and adds a subtotal, the Reporting Engine constructs an underlying SOQL aggregate query to fetch the exact numbers.

### Dashboard Engine
Dashboards cache and visualize the output of the Reporting Engine's aggregate queries. They rely heavily on optimized aggregations to render charts quickly without hitting transaction limits.

**How it fits together:**
The database engine processes raw records -> SOQL Aggregate Queries summarize them -> Apex/Reporting Engine consumes the summary -> Dashboards visualize the intelligence.

---

## 3. What are Aggregate Queries?

### Definition
An aggregate query is a SOQL statement that uses one or more aggregate functions, often combined with a `GROUP BY` clause, to return calculated insights rather than standard sObjects.

### Purpose
To compute mathematical summaries (counts, averages, totals) across large data sets efficiently without breaching Apex governor limits.

### Internal Execution & Query Lifecycle
1.  **Parsing:** The SOQL engine identifies the aggregate functions.
2.  **Filtering (WHERE):** The database filters the raw records based on the `WHERE` clause.
3.  **Grouping:** The remaining records are segmented into buckets based on the `GROUP BY` fields.
4.  **Aggregation:** The mathematical functions (e.g., `SUM`) are applied to each bucket.
5.  **Post-Filtering (HAVING):** The aggregated buckets are filtered.
6.  **Return:** The engine constructs and returns an array of `AggregateResult` objects.

### Why they return summarized data
Standard queries map directly to sObject schemas (e.g., an Account record). An aggregate result, like `SUM(Amount__c)`, does not exist as a physical field on a record. Therefore, Salesforce wraps these dynamic, abstract results in a generic key-value map object called `AggregateResult`.

---

## 4. AggregateResult

### What AggregateResult is
`AggregateResult` is a read-only sObject used exclusively to hold the dynamically generated values of an aggregate query.

### Internal data structure
Internally, it acts like a `Map<String, Object>`. It stores data as key-value pairs where the key is the field alias (or implicit name) and the value is the calculated result.

### Accessing values and Type Casting
Because the values are stored as generic `Object` types, developers must explicitly cast them to the correct Apex data type (e.g., `Decimal`, `Integer`, `Id`).

### Aliases
Aliases allow you to name the output of an aggregate function. If no alias is provided, Salesforce assigns implicit aliases (`expr0`, `expr1`, etc.).

### Apex Example
```apex
// Query calculating total warranty claims for a dealer
AggregateResult[] results = [
    SELECT Dealer__c, SUM(Claim_Amount__c) TotalAmount
    FROM Warranty_Claim__c
    GROUP BY Dealer__c
];

for (AggregateResult ar : results) {
    // Extracting the grouped field (Id type)
    Id dealerId = (Id) ar.get('Dealer__c');
    
    // Extracting the aliased aggregate field (Decimal type)
    Decimal totalClaim = (Decimal) ar.get('TotalAmount');
    
    System.debug('Dealer: ' + dealerId + ' | Total: ' + totalClaim);
}
```

**Line-by-Line Explanation:**
1. Execute SOQL: Groups records by `Dealer__c` and sums the amount, aliasing it as `TotalAmount`. Stores in an array of `AggregateResult`.
2. Loop: Iterate through the array.
3. Cast to Id: Retrieve the grouping key using the field API name and cast it to `Id`.
4. Cast to Decimal: Retrieve the sum using the custom alias `TotalAmount` and cast to `Decimal`.
5. Debug: Output the summarized data.

---

## 5. Aggregate Functions Overview

| Function | Purpose | Return Type | Supported Fields | Common Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **COUNT()** | Counts rows returned | `Integer` | None (Table-level) | Total number of Warranty Claims. |
| **COUNT(Id)** | Counts non-null records | `AggregateResult` | Id | Total claims submitted. |
| **COUNT_DISTINCT()** | Counts unique non-null values | `AggregateResult` | Any | Number of unique Dealers who submitted claims. |
| **SUM()** | Adds numerical values | `AggregateResult` | Number, Currency, Percent | Total revenue from Spare Orders. |
| **AVG()** | Calculates mathematical mean | `AggregateResult` | Number, Currency, Percent | Average repair cost per Vehicle. |
| **MIN()** | Finds lowest/earliest value | `AggregateResult` | Number, Currency, Date, String | Earliest Warranty Claim date. |
| **MAX()** | Finds highest/latest value | `AggregateResult` | Number, Currency, Date, String | Highest invoice amount this month. |

---

## 6. COUNT()

### Syntax & Behavior
The count function calculates the number of rows that match the query criteria.

* `COUNT()`: Returns an `Integer`. It counts all rows, including those with null values. It is the *only* aggregate function that does not return an `AggregateResult` when used alone.
* `COUNT(Id)`: Returns an `AggregateResult`. Counts the number of non-null Id fields (which is effectively all records).
* `COUNT(fieldName)`: Returns an `AggregateResult`. Counts the number of records where `fieldName` is *not null*.

### Examples
```apex
// Returns an Integer. Total vehicles in the system.
Integer totalVehicles = [SELECT COUNT() FROM Vehicle__c];

// Returns AggregateResult. Counts claims with a populated Service_Center__c.
AggregateResult[] claimsWithCenters = [
    SELECT COUNT(Service_Center__c) TotalAssigned 
    FROM Warranty_Claim__c
];
```

---

## 7. COUNT_DISTINCT()

### Unique Record Counting
`COUNT_DISTINCT(fieldName)` returns the number of distinct, non-null values for a specific field. 

### Supported Fields & Limitations
Supported on mostly all field types. However, it requires a table scan of the specific field and can be performance-heavy on Large Data Volumes (LDV).

### Examples
```apex
// How many unique Service Centers filed claims this month?
AggregateResult[] uniqueCenters = [
    SELECT COUNT_DISTINCT(Service_Center__c) UniqueCenters
    FROM Warranty_Claim__c
    WHERE CreatedDate = THIS_MONTH
];
```

---

## 8. SUM()

### Numeric & Currency Fields
`SUM()` aggregates numerical fields. When used with Currency fields, it automatically respects the organization's corporate currency if multi-currency is enabled, though it returns the raw numerical value in Apex.

### Roll-up Calculations
It acts as a dynamic alternative to Roll-Up Summary Fields, especially useful when relationships are Lookups instead of Master-Detail.

### Examples
```apex
// Calculate the total value of all approved warranty claims
AggregateResult[] totalApproved = [
    SELECT SUM(Approved_Amount__c) TotalValue
    FROM Warranty_Claim__c
    WHERE Status__c = 'Approved'
];
```

---

## 9. AVG()

### Average Calculations & Decimal Handling
Calculates the mean. It automatically ignores null values in its calculation (it does not treat nulls as zero). The result may have high decimal precision, so scaling/rounding in Apex is recommended.

### Business Reporting
Used for KPIs like CSAT scores, average time to resolution, and average invoice amounts.

### Examples
```apex
// Find the average labor cost across all work orders
AggregateResult[] avgLabor = [
    SELECT AVG(Labor_Cost__c) AverageLabor
    FROM Work_Order__c
];
Decimal avg = (Decimal)avgLabor[0].get('AverageLabor');
avg = avg.setScale(2); // Format for currency
```

---

## 10. MIN()

### Earliest & Smallest Values
Returns the lowest value. For numbers, it is the mathematical minimum. For Dates/DateTimes, it is the earliest chronological time. For strings, it evaluates alphabetically.

### Examples
```apex
// Find the date of the very first warranty claim for a specific vehicle
AggregateResult[] firstClaim = [
    SELECT MIN(CreatedDate) FirstClaimDate
    FROM Warranty_Claim__c
    WHERE Vehicle__c = 'a01...Id'
];
```

---

## 11. MAX()

### Latest & Highest Values
Returns the highest numeric value, latest date, or highest alphabetical string.

### Business Scenarios
Identifying the most recent interaction, the largest deal size, or the latest service appointment.

### Examples
```apex
// Find the highest priced spare part in inventory
AggregateResult[] highestPart = [
    SELECT MAX(Unit_Price__c) MaxPrice
    FROM Spare_Part__c
];
```

---

## 12. GROUP BY

### Purpose
`GROUP BY` divides query results into separate subsets (buckets) based on the values of one or more specified fields, allowing aggregate functions to be calculated per bucket.

### Internal Execution & Result Generation
The database engine sorts the dataset by the grouped fields, calculates the aggregates for each unique combination, and returns one `AggregateResult` row per group.

### Examples
```apex
// Total warranty cost grouped by Dealer
AggregateResult[] dealerCosts = [
    SELECT Dealer__c, SUM(Claim_Amount__c) TotalCost
    FROM Warranty_Claim__c
    GROUP BY Dealer__c
];
```

---

## 13. GROUP BY ROLLUP

### Hierarchical Summaries
`ROLLUP` extends `GROUP BY` by adding subtotal rows and a grand total row to the results. It is highly useful for hierarchical data reporting.

### Examples
```apex
// Rollup by Region, then Dealer
AggregateResult[] rollupResults = [
    SELECT Region__c, Dealer__c, SUM(Claim_Amount__c) Total
    FROM Warranty_Claim__c
    GROUP BY ROLLUP(Region__c, Dealer__c)
];
```
*Output generation concept:*
* Row 1: North, Dealer A, $5000 (Detailed)
* Row 2: North, Dealer B, $3000 (Detailed)
* Row 3: North, `null`, $8000 (Subtotal for North)
* Row 4: `null`, `null`, $8000 (Grand Total)

---

## 14. GROUP BY CUBE

### Multi-dimensional Aggregation
While `ROLLUP` calculates subtotals hierarchically, `CUBE` calculates subtotals for *every possible combination* of the grouped fields. 

### Difference from ROLLUP
If you group by `(Region, Dealer)`, `ROLLUP` gives subtotals for Region, and a Grand Total. `CUBE` gives subtotals for Region, subtotals for Dealer (regardless of region), and a Grand Total. Highly intensive, used for matrix analytics.

### Examples
```apex
AggregateResult[] cubeResults = [
    SELECT Lead_Source__c, Rating__c, COUNT(Id)
    FROM Lead
    GROUP BY CUBE(Lead_Source__c, Rating__c)
];
```

---

## 15. HAVING Clause

### Purpose and Execution Order
`HAVING` is to aggregate functions what `WHERE` is to standard fields. It filters the results *after* the grouping and aggregation have occurred. 

### Execution Order:
1. `WHERE` (Filters raw records)
2. `GROUP BY` (Buckets records)
3. `Aggregate Calculation` (Math happens)
4. `HAVING` (Filters the calculated buckets)

### Examples
```apex
// Find dealers who have submitted more than $50,000 in claims
AggregateResult[] highVolumeDealers = [
    SELECT Dealer__c, SUM(Claim_Amount__c) TotalAmount
    FROM Warranty_Claim__c
    GROUP BY Dealer__c
    HAVING SUM(Claim_Amount__c) > 50000
];
```

---

## 16. WHERE vs HAVING

| Feature | `WHERE` Clause | `HAVING` Clause |
| :--- | :--- | :--- |
| **Execution Order** | Executes *before* grouping. | Executes *after* grouping. |
| **Filtering Stage** | Filters individual records. | Filters aggregated groups/buckets. |
| **Aggregate Support**| Cannot contain aggregate functions. | Evaluates aggregate functions. |
| **Performance** | Highly efficient. Reduces dataset size early. | Less efficient. Computes math before filtering. |
| **Use Case** | "Only calculate claims from 2026." | "Only show dealers whose total claims > $50k." |

---

## 17. Aggregate Query Execution Process

Internally, Salesforce processes complex SOQL queries in a strict pipeline:

1.  **Query Parsing:** Syntax validation and field mapping.
2.  **Metadata Resolution:** Verifying object/field existence.
3.  **Security Evaluation:** Applying Sharing Rules, object/field level security (if enforced).
4.  **WHERE Filtering:** Using indexes to quickly discard irrelevant raw records.
5.  **Record Selection:** Fetching the raw data subset into database memory.
6.  **Group Formation:** Sorting and hashing the data by `GROUP BY` columns.
7.  **Aggregate Calculation:** Running functions (`SUM`, `AVG`) on the buckets.
8.  **HAVING Filtering:** Discarding buckets that do not meet post-aggregation criteria.
9.  **Result Construction:** Formatting outputs into `AggregateResult` format for Apex.

---

## 18. Aggregate Queries in Apex

### Production-Quality Example
```apex
public class WarrantyAnalyticsService {
    
    public static Map<Id, Decimal> getHighCostDealers() {
        Map<Id, Decimal> dealerMetrics = new Map<Id, Decimal>();
        
        // Query to find active dealers with high average repair costs
        List<AggregateResult> results = [
            SELECT Dealer__c, 
                   COUNT(Id) TotalClaims, 
                   SUM(Claim_Amount__c) TotalCost, 
                   AVG(Labor_Cost__c) AvgLabor
            FROM Warranty_Claim__c
            WHERE Status__c = 'Approved' 
            WITH USER_MODE
            GROUP BY Dealer__c
            HAVING AVG(Labor_Cost__c) > 500
        ];
        
        for (AggregateResult ar : results) {
            // Casting grouping Id
            Id dealerId = (Id) ar.get('Dealer__c');
            
            // Casting aliased decimal
            Decimal totalCost = (Decimal) ar.get('TotalCost');
            
            dealerMetrics.put(dealerId, totalCost);
        }
        
        return dealerMetrics;
    }
}
```

**Line-by-Line Explanation:**
* `Map<Id, Decimal> dealerMetrics`: Initialize a map to return structured data.
* `SELECT Dealer__c, COUNT(Id)...`: Select grouping field and apply functions with explicit aliases.
* `FROM Warranty_Claim__c`: Target object.
* `WHERE Status__c = 'Approved'`: Pre-filter records to only approved claims (optimization).
* `WITH USER_MODE`: Enforces object and FLS security for the executing user (Security Best Practice).
* `GROUP BY Dealer__c`: Segment data by dealer.
* `HAVING AVG(Labor_Cost__c) > 500`: Post-filter to only keep dealers averaging > $500 in labor.
* `for (AggregateResult ar : results)`: Iterate over the dynamic results.
* `(Id) ar.get(...)` & `(Decimal) ar.get(...)`: Typecast the generic Objects to strong Apex types.
* `dealerMetrics.put(...)`: Populate the map for the caller.

---

## 19. Aggregate Queries in LWC

### Apex Controller & Serialization
LWC cannot directly consume `AggregateResult` objects natively in datatables without formatting. The best practice is to map the `AggregateResult` into a custom Apex Wrapper Class and return a `List<Wrapper>` via `@AuraEnabled(cacheable=true)`.

### Displaying Data
```apex
// Wrapper Class
public class DealerStat {
    @AuraEnabled public String dealerName {get;set;}
    @AuraEnabled public Decimal totalAmount {get;set;}
}
// Return List<DealerStat> to LWC
```
In the LWC JavaScript, this data easily binds to a `lightning-datatable` or can be fed into charting libraries (like Chart.js) to render dynamic Executive Dashboards showing claim volumes by dealer.

---

## 20. Aggregate Queries and Reports

| Feature | SOQL Aggregate Queries | Salesforce Reports / CRM Analytics |
| :--- | :--- | :--- |
| **Execution** | Programmatic (Apex/API) | Declarative / UI Driven |
| **Use Case** | Backend logic, triggers, custom LWC dashboards. | End-user analytics, standard dashboards. |
| **Complexity**| Highly customizable via Apex, but code-heavy. | Point-and-click, natively handles UI rendering. |
| **Limits** | Strict Governor Limits (50k rows processed). | Handles millions of rows (especially CRM Analytics). |

---

## 21. Aggregate Queries and Security

By default, Apex runs in System Context, meaning it ignores Object-Level Security (OLS) and Field-Level Security (FLS). When calculating aggregations, this can accidentally expose sums of restricted fields (e.g., Executive Revenue) to lower-level users.

### Enforcement
* **WITH USER_MODE:** The modern, preferred way to enforce OLS, FLS, and Sharing Rules directly in the SOQL statement.
* **WITH SECURITY_ENFORCED:** Legacy equivalent, though `USER_MODE` provides better exception handling.
* **Sharing Rules:** Controlled by the `with sharing` keyword on the Apex class.

```apex
AggregateResult[] secureResults = [
    SELECT SUM(Confidential_Revenue__c) TotalRev 
    FROM Dealership_Financials__c 
    WITH USER_MODE
];
```

---

## 22. Governor Limits

Aggregate queries are bound by Salesforce's strict multi-tenant governor limits. 

| Limit Type | Limit Boundary | Description |
| :--- | :--- | :--- |
| **Processed Rows** | 50,000 | The total number of *raw* records evaluated by the query (pre-grouping) counts toward the 50K total query row limit. |
| **Returned Groups** | 2,000 | If returning `AggregateResult` without a `LIMIT` clause or `WHERE` clause limiting groups, max returned rows might hit standard pagination or memory limits. |
| **CPU Time** | 10,000 ms (Sync) | Complex grouping over Large Data Volumes consumes significant CPU time. |
| **Heap Size** | 6 MB (Sync) | While `AggregateResult` saves heap compared to standard records, massive `GROUP BY CUBE` queries can still breach heap. |

*Note: If `SELECT COUNT()` processes 100,000 records, it will throw a `LimitException: Too many query rows: 50001`.*

---

## 23. Query Optimization

### Strategies for High Performance
1.  **Selective Filters:** Always use a `WHERE` clause on Indexed Fields (e.g., `CreatedDate = THIS_YEAR`, External IDs, Lookup fields) to reduce the dataset *before* grouping.
2.  **Avoid Grouping on Formula Fields:** Grouping by non-deterministic formula fields forces full table scans.
3.  **Large Data Volumes (LDV):** For tables > millions of rows, avoid synchronous SOQL aggregations. Use Asynchronous Apex (Batch), CRM Analytics, or populate summary data via trigger-based Rollups into a separate metrics custom object.
4.  **Query Plan Tool:** Use the Developer Console Query Plan tool to ensure the cost of the aggregate query is below 1.0 (indicating index usage).

---

## 24. Common Mistakes

| Mistake | Consequence | Solution |
| :--- | :--- | :--- |
| **Forgetting Aliases** | Relying on `expr0` makes code unreadable and prone to breaking if query order changes. | Always use explicit aliases: `SUM(Amount) TotalAmt` |
| **Null pointer exceptions** | `SUM()` returns `null` if no records are found, causing NPEs during casting. | Check for null before casting: `if(ar.get('Total') != null)` |
| **WHERE vs HAVING mix-ups** | Filtering aggregate results using `WHERE` causes syntax errors. | Use `HAVING` for post-aggregation filters. |
| **High Cardinality Grouping** | Grouping by fields like `Description` or `Name` crashes performance. | Only group by Picklists, Lookups, or Booleans. |

---

## 25. Real Project Scenarios: Automotive CRM

**1. Count Warranty Claims per Dealer**
```apex
SELECT Dealer__c, COUNT(Id) ClaimCount FROM Warranty_Claim__c GROUP BY Dealer__c
```

**2. Total Warranty Amount per Month**
```apex
SELECT CALENDAR_MONTH(CreatedDate) Month, SUM(Claim_Amount__c) Total FROM Warranty_Claim__c GROUP BY CALENDAR_MONTH(CreatedDate)
```

**3. Average Repair Cost by Vehicle Model**
```apex
SELECT Vehicle__r.Model__c, AVG(Total_Repair_Cost__c) AvgCost FROM Work_Order__c GROUP BY Vehicle__r.Model__c
```

**4. Maximum Claim Amount in SAP Integration Queue**
```apex
SELECT MAX(Amount__c) MaxPending FROM Warranty_Claim__c WHERE SAP_Sync_Status__c = 'Pending'
```

**5. Minimum Invoice Value for Spare Orders**
```apex
SELECT MIN(Invoice_Total__c) MinInvoice FROM Spare_Order__c WHERE Order_Date__c = LAST_N_DAYS:30
```

**6. Dealer-wise Claim Statistics (Multiple aggregates)**
```apex
SELECT Dealer__c, COUNT(Id) TotalClaims, SUM(Claim_Amount__c) TotalCost, MAX(Claim_Amount__c) LargestClaim FROM Warranty_Claim__c GROUP BY Dealer__c
```

---

## 26. Enterprise Analytics Design

For Senior Architects, relying purely on SOQL Aggregate queries at runtime is a risk in Large Enterprise scale. 
* **Executive Dashboards:** Do not build LWC dashboards that query live transactional data (e.g., Millions of Work Orders). Instead, design a Nightly Batch that runs the aggregate queries and saves the `AggregateResult` data into a custom `Daily_Metric__c` object. The dashboard queries the metrics object instantly.
* **Business KPIs:** Utilize CRM Analytics (Tableau CRM) for cross-object, multi-million-row slicing. Keep SOQL Aggregates reserved for transactional validation (e.g., "Prevent closure if total parts cost > $5000").

---

## 27. Performance Considerations

* **Heap Usage:** `AggregateResult` arrays are lightweight compared to standard sObject arrays, but maintaining maps of grouped data in Apex can bloat the Heap. Process in chunks if necessary.
* **Grouping Efficiency:** Grouping by Lookups (e.g., `Dealer__c`) is highly efficient because Lookups are natively indexed.
* **Aggregate Query Cost:** If a table has 5 million `Warranty_Claim__c` records, `SELECT COUNT()` with no selective filters will fail. Use the REST API `query/` endpoint for record counts or highly selective `WHERE` clauses.

---

## 28. Best Practices

1.  **Always use Meaningful Aliases:** `SUM(Amount) totalRevenue` not `SUM(Amount)`.
2.  **Filter First, Group Second:** Shrink the dataset with `WHERE` using indexed fields before applying `GROUP BY`.
3.  **Avoid Unnecessary Math:** Don't calculate `AVG()` if the business only needs `SUM()`. It wastes CPU.
4.  **Bulk-Safe Apex:** When querying aggregates inside trigger handlers (e.g., checking if a dealer exceeded their monthly claim budget), ensure the `WHERE` clause leverages the `Trigger.new` keys via `IN` binding to remain bulkified.
5.  **Use `WITH USER_MODE`:** Always enforce security.

---

## 29. Debugging & Troubleshooting

* **Incorrect AggregateResult access:** `System.SObjectException: Invalid field for AggregateResult`. *Solution:* Ensure you are requesting the exact Alias defined in the query.
* **Null aggregate values:** *Solution:* Use `Decimal result = ar.get('total') != null ? (Decimal)ar.get('total') : 0.00;`
* **Developer Console:** Use the Query Editor tab. Check the "Use Tooling API" box if standard queries fail to format aggregates properly.
* **Debug Logs:** When inspecting arrays in debug logs, `AggregateResult` often prints as `{expr0=500, expr1=10}`. Cross-reference the implicit expressions with the query order.

---

## 30. Interview Questions & Answers

### Beginner
**Q: Can you retrieve individual record fields in an aggregate query?**
A: No. If you use a `GROUP BY` clause, you can only `SELECT` the grouped fields or aggregate functions. Attempting to select a standard field without grouping it will throw a compile error.

### Intermediate
**Q: What is the difference between `COUNT()` and `COUNT(Id)`?**
A: `COUNT()` returns an `Integer` representing the total rows. `COUNT(Id)` returns an `AggregateResult` array, counting non-null Ids.

### Advanced
**Q: How do aggregate queries affect the 50,000 SOQL row limit?**
A: The limit applies to the number of raw records the query engine *processes* to calculate the result, not just the number of `AggregateResult` rows returned. Summing 50,001 rows will throw a Limit Exception.

### Architect-Level
**Q: A client has 10 million Order records. They want a real-time LWC dashboard showing the total sum of Orders grouped by Account. How do you design this?**
A: Real-time SOQL aggregation will hit limits instantly. Solution: Build an async process (Batch Apex/Dataflow) to pre-aggregate the data nightly into a custom `Account_Summary__c` object, or leverage CRM Analytics. Alternatively, for strict real-time, maintain the sum via Trigger logic (Rollup) on the Account upon Order creation, and query the Account fields directly in LWC.

---

## 31. Revision Summary

* **Aggregate Queries:** Summarize data at the DB layer to save Apex CPU and Heap.
* **AggregateResult:** Generic key-value object holding results. Requires explicit type casting.
* **COUNT / COUNT_DISTINCT:** Row counts vs Unique value counts.
* **SUM / AVG / MIN / MAX:** Core mathematical functions for metrics.
* **GROUP BY:** Segments data into buckets based on field values.
* **GROUP BY ROLLUP/CUBE:** Adds hierarchical and matrix subtotals.
* **WHERE vs HAVING:** `WHERE` filters raw data; `HAVING` filters aggregated buckets.
* **Security:** Always use `WITH USER_MODE`.
* **Limits:** Pre-aggregated processed rows count towards the 50K query row limit.
* **Best Practice:** Pre-filter with selective `WHERE` clauses on indexed fields to optimize performance before grouping.