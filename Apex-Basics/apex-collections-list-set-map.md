# Apex Collections List Set Map

## 1. Introduction

### What Collections Are
In Apex, a collection is a specialized type of variable that can store multiple records or items of the same data type. Instead of creating individual variables for every piece of data, collections allow developers to group data together, enabling efficient storage, retrieval, and manipulation.

### Why Collections are Essential in Apex
Apex operates in a multi-tenant environment governed by strict execution limits (Governor Limits). Collections are the foundational building blocks that allow developers to handle multiple records simultaneously, preventing the code from hitting SOQL and DML limits.

### Relationship Between Collections and Bulkification
Bulkification is the practice of ensuring your code can handle more than one record at a time. Collections are the enablers of bulkification. Instead of executing a SOQL query or DML statement inside a loop for a single record, you gather records in a **List**, **Set**, or **Map**, and perform the operation on the entire collection at once.

### Why Enterprise Applications Rely Heavily on Collections
Enterprise applications deal with Large Data Volumes (LDV) and complex relational architectures. Collections allow for:
* In-memory data joins (reducing SOQL queries).
* De-duplication of processing criteria (using Sets).
* Fast record lookups (using Maps).

**Automotive CRM Example:** Processing 500 `Warranty_Claim__c` records received from SAP simultaneously requires collections to group the claims by `Dealer__c`, query existing data, and insert the records efficiently.

---

## 2. Collection Architecture in Apex

### Core Collection Types
* **List:** An ordered, index-based collection that allows duplicates.
* **Set:** An unordered collection of unique primitives or sObjects.
* **Map:** A collection of key-value pairs where each unique key maps to a single value.

### Internal Concepts & Memory Allocation
Apex collections are dynamically sized. They allocate memory on the **Heap** as elements are added. 
* **Lists** behave like dynamic arrays.
* **Sets** and **Maps** are implemented using hash tables, relying on the `hashCode()` and `equals()` methods for element uniqueness and retrieval.

### Generic Typing & Collection Hierarchy
Apex collections are **strongly typed** using Generics. You must declare the type of data a collection will hold at compile time (e.g., `List<String>`). This ensures type safety and prevents runtime casting errors.

---

## 3. Why Collections Matter in Salesforce

### Trigger Bulk Processing
`Trigger.new` and `Trigger.old` are inherently `List<sObject>`. Triggers must use collections to process all incoming records without hitting limits.

### SOQL Query Results
SOQL queries always return a `List<sObject>` (even if limit 1 is used, it can be assigned to a list).

### DML Operations
Salesforce DML operations (Insert, Update, Delete) are optimized to accept `List<sObject>`, executing a single database transaction for up to 10,000 records.

### Governor Limits & Large Data Volumes (LDV)
By using Sets to gather unique IDs and Maps to perform O(1) lookups in memory, developers avoid querying the database inside loops, which is the #1 cause of `System.LimitException: Too many SOQL queries: 101`.

---

## 4. Generic Collections

### Deep Dive into Generics
Generics allow you to define a collection that works with a specific data type `<T>`. This enforces **Compile-time validation** and **Type safety**.

### Examples
```java
List<Account> accountList = new List<Account>();
Set<Id> dealerIds = new Set<Id>();
Map<Id, Contact> contactMap = new Map<Id, Contact>();
```

### Line-by-Line Explanation
1. `List<Account> accountList = new List<Account>();`
   * Instantiates a new empty List in memory that can *only* hold `Account` sObjects. If you try to add a `Contact`, the compiler will throw an error.
2. `Set<Id> dealerIds = new Set<Id>();`
   * Creates a Set designed strictly for 18-character Salesforce `Id` types, ensuring no duplicate dealer IDs are stored.
3. `Map<Id, Contact> contactMap = new Map<Id, Contact>();`
   * Initializes a Map where the Key is an `Id` and the Value is a `Contact` sObject.

---

## 5. List Collections

### What is a List?
A List is an ordered collection of elements distinguished by their indices. 

### Characteristics
* **Ordered Collection:** Elements maintain the order in which they were inserted.
* **Duplicate Support:** Lists allow duplicate values and null values.
* **Index-Based Access:** Elements are accessed using a zero-based index (e.g., `myList[0]`).

### Comparison
| Feature | List | Set | Map |
| :--- | :--- | :--- | :--- |
| **Ordered** | Yes | No | No |
| **Duplicates** | Yes | No | Keys: No, Values: Yes |
| **Access By** | Index (Integer) | Value | Key |

---

## 6. Creating Lists

### Examples
```java
List<String> statuses = new List<String>();
statuses.add('Pending');

List<Vehicle__c> vehicles = [SELECT Id, Name FROM Vehicle__c LIMIT 100];
```

### Line-by-Line Explanation
1. `List<String> statuses = new List<String>();`
   * Allocates heap memory for a List of Strings.
2. `statuses.add('Pending');`
   * Appends the string 'Pending' to the 0th index of the List.
3. `List<Vehicle__c> vehicles = [SELECT Id, Name FROM Vehicle__c LIMIT 100];`
   * Executes a SOQL query fetching up to 100 Vehicle records and directly assigns the resulting array of records to the `vehicles` List.

---

## 7. List Methods

| Method | Description | Example |
| :--- | :--- | :--- |
| `add(T)` | Adds an element to the end. | `myList.add('Data');` |
| `addAll(List/Set)` | Adds all elements from another collection. | `myList.addAll(otherList);` |
| `remove(Integer)` | Removes element at the specified index. | `myList.remove(0);` |
| `clear()` | Removes all elements, freeing heap. | `myList.clear();` |
| `contains(T)` | Checks if the list has the element. | `myList.contains('Data');` |
| `size()` | Returns the number of elements. | `Integer s = myList.size();` |
| `get(Integer)` | Retrieves element at the index. | `String val = myList.get(0);` |
| `sort()` | Sorts primitive types or sObjects. | `myList.sort();` |
| `clone()` | Creates a shallow copy of the list. | `List<String> newList = myList.clone();` |

---

## 8. List Iteration Techniques

### Examples
```java
// 1. Traditional For Loop
for (Integer i = 0; i < vehicles.size(); i++) {
    System.debug(vehicles[i].Name);
}

// 2. Enhanced For Loop (Recommended for readability)
for (Vehicle__c v : vehicles) {
    System.debug(v.Name);
}

// 3. Iterator Pattern (Useful for complex removals)
Iterator<Vehicle__c> iter = vehicles.iterator();
while (iter.hasNext()) {
    System.debug(iter.next().Name);
}
```

---

## 9. List Performance Considerations

* **Index lookup complexity:** O(1). Accessing `myList[50]` is instantaneous.
* **Search complexity:** O(N). `contains()` requires scanning the list. For large datasets, this is slow. Use Sets for `.contains()` checks.
* **Memory usage:** Lists holding large sObjects consume significant Heap memory.
* **Optimization:** Clear lists (`myList.clear()`) when no longer needed to free heap space before large operations.

---

## 10. Set Collections

### What is a Set?
A Set is an unordered collection of elements that do not contain duplicates. 

### Characteristics
* **Unique Values:** Attempting to add a duplicate value does nothing; the Set simply ignores it.
* **Unordered Nature:** Sets do not guarantee the order of elements. You cannot access a Set by index.
* **Duplicate Prevention:** Excellent for gathering IDs from a list to perform a SOQL query.

---

## 11. Creating Sets

### Examples
```java
Set<Id> dealerIds = new Set<Id>();

Set<String> regions = new Set<String>{
    'APAC',
    'EMEA'
};
```

### Line-by-Line Explanation
1. `Set<Id> dealerIds = new Set<Id>();`
   * Initializes an empty Set specifically for Salesforce IDs.
2. `Set<String> regions = new Set<String>{...};`
   * Initializes a Set and instantly populates it with two string values using collection literal syntax.

---

## 12. Set Methods

| Method | Description | Example |
| :--- | :--- | :--- |
| `add(T)` | Adds an element if it doesn't already exist. | `mySet.add(recordId);` |
| `addAll(List/Set)` | Adds all elements from a List or Set. | `mySet.addAll(idList);` |
| `remove(T)` | Removes the specified element. | `mySet.remove(recordId);` |
| `contains(T)` | Returns true if the element exists. | `mySet.contains(recordId);` |
| `clear()` | Empties the Set. | `mySet.clear();` |
| `size()` | Returns the number of unique elements. | `mySet.size();` |

---

## 13. Set Performance Considerations

* **Fast lookups:** Checking `set.contains()` is an O(1) operation because of internal hashing. This makes Sets vastly superior to Lists for search operations.
* **Heap efficiency:** Sets only store unique values, saving memory compared to a List that might contain redundant data.
* **Best Use Case:** Gathering `Id` values from `Trigger.new` to feed into the `IN` clause of a SOQL query.

---

## 14. Map Collections

### What is a Map?
A Map is a collection of key-value pairs. Each key maps to a single value.

### Characteristics
* **Key-Value Pairs:** Keys must be unique; values can be duplicated.
* **Fast Retrieval:** Hash-based lookup allows O(1) retrieval of a value if the key is known.

---

## 15. Creating Maps

### Examples
```java
Map<Id, Dealer__c> dealerMap = new Map<Id, Dealer__c>(
    [SELECT Id, Name FROM Dealer__c WHERE Region__c = 'APAC']
);
```

### Line-by-Line Explanation
1. `Map<Id, Dealer__c> dealerMap = ...`
   * Declares a map with `Id` as the key and `Dealer__c` sObject as the value.
2. `new Map<Id, Dealer__c>([SELECT...])`
   * A powerful shortcut: passing a SOQL query into a Map constructor automatically creates a Map where the `Id` of the queried record is the Key, and the record itself is the Value.

---

## 16. Map Methods

| Method | Description | Example |
| :--- | :--- | :--- |
| `put(Key, Value)` | Adds or updates a key-value pair. | `myMap.put(id, record);` |
| `get(Key)` | Retrieves the value for the key. | `myMap.get(id);` |
| `containsKey(Key)` | Checks if the key exists. | `myMap.containsKey(id);` |
| `keySet()` | Returns a Set of all keys. | `Set<Id> keys = myMap.keySet();` |
| `values()` | Returns a List of all values. | `List<sObject> vals = myMap.values();` |
| `remove(Key)` | Removes the mapping for a key. | `myMap.remove(id);` |

---

## 17. KeySet() and Values()

### Real-World Use Cases
```java
// Extracting keys to use in a SOQL query
Set<Id> dealerIds = dealerMap.keySet();
List<Warranty_Claim__c> claims = [SELECT Id FROM Warranty_Claim__c WHERE Dealer__c IN :dealerIds];

// Extracting values for DML
List<Dealer__c> dealersToUpdate = dealerMap.values();
update dealersToUpdate;
```
* `keySet()` is constantly used to extract parent IDs to query child records.
* `values()` is used when you have modified the objects within the map and are ready to push them to the database via DML.

---

## 18. Nested Collections

### What Are They?
Collections placed inside other collections. Essential for grouping related records in memory without writing nested SOQL.

### Examples & Business Use Cases
* **`Map<Id, List<Warranty_Claim__c>>`**: Groups a list of Warranty Claims by their parent Dealer ID. Allows you to iterate over Dealers, and immediately access all related claims.
* **`List<Map<String, Object>>`**: Often used for parsing dynamic JSON arrays during REST API integrations (e.g., SAP payloads).
* **`Map<String, Set<Id>>`**: Grouping a unique set of Vehicle IDs by their Model Name (String).

---

## 19. Collections and SOQL

* **Query results as Lists:** Standard SOQL `[SELECT Id FROM Account]` returns a List.
* **Map constructors:** `new Map<Id, sObject>([SOQL])` immediately indexes records by ID.
* **Relationship queries:** Subqueries return a List. E.g., `List<Contact> cons = acc.Contacts;`
* **Aggregate results:** `List<AggregateResult> results = [SELECT COUNT(Id), Dealer__c FROM... GROUP BY Dealer__c];`

---

## 20. Collections and DML

Salesforce DML is bulkified natively to accept Lists.

```java
List<Vehicle__c> newVehicles = new List<Vehicle__c>{...};
insert newVehicles; // Bulk Insert

List<Warranty_Claim__c> claimsToUpdate = new List<Warranty_Claim__c>{...};
update claimsToUpdate; // Bulk Update

List<Dealer__c> deadDealers = [SELECT Id FROM Dealer__c WHERE Status__c = 'Closed'];
delete deadDealers; // Bulk Delete
```
*Always perform DML on collections, never on individual records inside a loop.*

---

## 21. Collections in Triggers

### Trigger Bulkification Pattern
Production-quality triggers use Sets to gather IDs, Maps to query related data, and Lists to update.

```java
public class WarrantyClaimTriggerHandler {
    public static void handleBeforeInsert(List<Warranty_Claim__c> newClaims) {
        // 1. Set to hold Parent IDs
        Set<Id> vehicleIds = new Set<Id>();
        for (Warranty_Claim__c claim : newClaims) {
            if (claim.Vehicle__c != null) vehicleIds.add(claim.Vehicle__c);
        }

        // 2. Map for O(1) Lookups
        Map<Id, Vehicle__c> vehicleMap = new Map<Id, Vehicle__c>(
            [SELECT Id, Warranty_Status__c FROM Vehicle__c WHERE Id IN :vehicleIds]
        );

        // 3. Process records
        for (Warranty_Claim__c claim : newClaims) {
            if (vehicleMap.containsKey(claim.Vehicle__c)) {
                Vehicle__c parent = vehicleMap.get(claim.Vehicle__c);
                if (parent.Warranty_Status__c == 'Expired') {
                    claim.addError('Cannot claim against expired warranty.');
                }
            }
        }
    }
}
```

---

## 22. Collections in Batch Apex

### Large Data Processing
Batch Apex processes millions of records by chunking them into smaller lists (default 200). 
* `Database.QueryLocator` fetches up to 50M records.
* The `execute(Database.BatchableContext BC, List<sObject> scope)` method provides a List chunk.
* **Memory Management:** Keep collections scoped to the `execute` method to ensure garbage collection clears heap memory between chunks.

---

## 23. Collections in Queueable Apex

### Asynchronous Processing
Queueable Apex allows passing complex data types (including Lists, Sets, Maps) to the constructor to maintain state asynchronously.
* **Serialization:** Collections passed to Queueable must be serializable.
* **State Management:**
```java
public class SAPIntegrationQueueable implements Queueable, Database.AllowsCallouts {
    private List<Id> claimIds;
    public SAPIntegrationQueueable(List<Id> claims) {
        this.claimIds = claims;
    }
    public void execute(QueueableContext context) {
        // Process claimIds List via REST API to SAP
    }
}
```

---

## 24. Collections in LWC

### Apex Controllers and LWC
When passing collections from Apex to Lightning Web Components, Apex Lists and Maps are serialized into JSON.
* **Apex List:** Becomes a JavaScript Array (`[]`).
* **Apex Map:** Becomes a JavaScript Object (`{}`).
* **Wrapper Classes:** Enterprise patterns often wrap Collections inside an Apex Object to send strongly-typed, structured JSON to LWC components.

---

## 25. Collection Memory Management

### Heap vs Stack
* References to the collection exist on the **Stack**.
* The actual data (sObjects, strings) is stored on the **Heap**.
* **Garbage Collection:** Setting a large list to `null` (e.g., `myLargeList = null;`) removes the stack reference, allowing Salesforce's garbage collector to reclaim the Heap memory.
* **Large Collection Handling:** Always clear collections (`clear()`) when processing large volumes sequentially to avoid `LimitException: Apex heap size too large`.

---

## 26. Governor Limits and Collections

| Limit | Description | Impact of Collections |
| :--- | :--- | :--- |
| **Heap Size** | 6 MB Sync / 12 MB Async | Large Lists/Maps of complex sObjects can breach this. Clear collections after use. |
| **CPU Time** | 10,000 ms Sync | Nested List loops cause exponential CPU growth (O(N^2)). Maps reduce this to O(N). |
| **SOQL Queries** | 100 Sync | Sets allow grouping IDs to use `IN` clauses, dropping query count to 1. |
| **DML Statements** | 150 Sync | Lists allow batching DML inserts/updates, utilizing 1 statement for 10,000 records. |

---

## 27. Performance Optimization

### Architect-Level Recommendations
1.  **Map > Nested Loops:** Never loop over a List inside another List. Build a Map of the inner List's records and do an O(1) `.get()` inside the outer loop.
2.  **Set Duplicate Removal:** Always use Sets for IDs before passing them to SOQL to prevent the database from doing unnecessary duplicate evaluations.
3.  **SOQL For-Loops:** For extreme heap optimization, use SOQL for-loops to process records in chunks natively: `for(Vehicle__c v : [SELECT Id FROM Vehicle__c]) {...}`.

---

## 28. Common Mistakes

* **Using Lists instead of Sets for Lookups:** Using `.contains()` on a List of 10,000 items inside a loop will crash the CPU limit. *Solution: Use Sets.*
* **Null Collections:** Attempting to `add()` to a List that hasn't been instantiated (`List<String> x; x.add('a');`) throws a NullPointerException. *Solution: Always `new List<String>()`.*
* **Querying inside Loops:** *Solution: Gather criteria in Sets, Query into Lists/Maps, Process.*

---

## 29. Real Project Scenarios

### Automotive CRM Examples
* **`Map<Id, Warranty_Claim__c>`:** Used to cache Warranty Claims fetched from the database before applying updates from a third-party SAP Integration.
* **`Set<Id> dealerIds;`:** Gathered from Work Orders to find all associated Dealers in one SOQL query.
* **`List<Invoice__c>`:** Used to hold hundreds of generated invoices in memory before committing a bulk `insert` DML.
* **`Map<Id, List<Claim_Line__c>>`:** A parent-to-child map mapping one `Warranty_Claim__c` ID to its multiple `Claim_Line__c` records, allowing processing logic to easily calculate total claim amounts in memory.

---

## 30. Enterprise Best Practices

* **Bulk-safe Coding:** Code should work exactly the same for 1 record as it does for 200 records.
* **Collection Reuse:** Do not instantiate new collections inside loops. Declare them before the loop.
* **Lazy Loading:** Only populate Maps with data if and when it is needed.
* **Memory Optimization:** Avoid `SELECT *` (or querying unnecessary large text fields) into collections to save Heap Size.

---

## 31. Debugging Collections

* **`System.debug()`:** `System.debug(myMap.keySet());` to quickly inspect data.
* **Developer Console:** Use the Execution Log to view output. Be aware that huge collections truncate in debug logs.
* **Heap Inspection:** The debug log details Heap usage per line, allowing you to see exactly which List is blowing up memory limits.
* **VS Code Replay Debugger:** Best enterprise tool. Download logs and step through code to inspect Map states and List indices dynamically.

---

## 32. Interview Questions & Answers

### Beginner
**Q: What is the difference between a List and a Set?**
A: A List is ordered and allows duplicates. A Set is unordered and requires elements to be unique.

### Intermediate
**Q: How do you avoid SOQL 101 limits using collections?**
A: By iterating over records, adding search criteria (like IDs) to a Set, and running a single SOQL query outside the loop using the `IN :mySet` clause. 

### Advanced
**Q: Explain how to process a List of 50,000 records without hitting CPU or Heap limits.**
A: Pass the List to Batch Apex. This chunks the List into manageable sizes (e.g., 200). Within the `execute` method, clear lists/maps manually or allow them to go out of scope to ensure the Garbage Collector reclaims heap memory between chunks. Avoid nested loops by using Maps.

### Architect-Level
**Q: Why might `List.contains()` cause a CPU timeout compared to `Set.contains()`?**
A: `List.contains()` operates at O(N) complexity, scanning every item sequentially. Inside a loop of M items, this results in O(N*M) complexity. `Set.contains()` uses hashing, operating at O(1) complexity, dropping the overall operational cost to O(M).

---

## 33. Revision Summary

* **Lists:** Ordered, index-based, allows duplicates. Best for returning SOQL, processing DML.
* **Sets:** Unordered, unique elements. Best for de-duplicating IDs and O(1) `.contains()` checks.
* **Maps:** Key-Value pairs. Hash-based. Best for fast O(1) memory joins and lookups.
* **Generic Collections:** Require `<T>` to enforce type safety at compile time.
* **Nested Collections:** `Map<Id, List<sObject>>` pattern allows grouping children under parents in memory.
* **Bulkification:** Triggers process chunks of 200. Collections are mandatory for bulk-safe triggers.
* **Governor Limits:** Unoptimized collections cause Heap Limit (6MB) and CPU Limit (10k ms) exceptions.
* **Performance:** Replace List searches with Set searches. Replace nested loops with Map lookups.
* **Best Practices:** Initialize outside loops, clear heap manually for large sets, query only needed fields.