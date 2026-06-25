# Apex Syntax & Variables – Primitive Types and Collections

## 1. Introduction

### What is Apex Syntax?
Apex is a strongly typed, object-oriented programming language executed on the Salesforce platform. Its syntax dictates how statements, variables, and logic are structured. Apex is designed specifically to query, manipulate, and manage Salesforce data safely within a multi-tenant environment.

### Why Understanding Variables and Collections is Important
Variables store data temporarily in memory, while collections group multiple variables. In Salesforce, Governor Limits restrict database operations (DML and SOQL). Collections are the foundation of **bulkification**—processing thousands of records in a single transaction without hitting these limits.

### Comparison with Java
Apex syntax is intentionally modeled after Java to flatten the learning curve.
* **Similarities:** Strongly typed, OOP concepts (classes, inheritance, polymorphism), bracket-based syntax.
* **Differences:** Apex is tightly integrated with the database (inline SOQL/DML), case-insensitive, executes on the cloud, and operates under strict Governor Limits. Apex does not support multiple threads or standard file I/O operations.

### Real-World Business Use Cases
In an Automotive CRM:
* **Variables** hold a single calculated `Warranty_Claim_Amount`.
* **Lists** hold 200 incoming `Vehicle__c` records from an SAP integration.
* **Maps** link a `Dealer_ID` (Key) to a list of associated `Work_Order__c` records (Value).

---

## 2. Apex Language Fundamentals

### Case Sensitivity
Unlike Java, Apex is **case-insensitive**. `Integer age = 5;` and `INTEGER AGE = 5;` are treated identically. However, best practice dictates using camelCase for variables and PascalCase for classes to maintain readability.

### Statements and Code Blocks
* **Statements:** Executable instructions ending with a semicolon `;`.
* **Code Blocks:** A group of statements enclosed in curly braces `{}`.

### Comments
Comments document code behavior and are ignored by the compiler.
* **Single-line comments:** Start with `//`
* **Multi-line comments:** Enclosed between `/*` and `*/`
* **Documentation comments:** Start with `/**` and end with `*/` (often used for ApexDoc).

```apex
/**
 * @description Calculates warranty claim totals.
 * @return void
 */
public void calculateClaims() {
    // This is a single-line comment
    Decimal totalClaim = 0.0; /* This is a
                                 multi-line comment */
}
```

---

## 3. Variables in Apex

### What are Variables?
Variables are named memory locations used to store data that can change during program execution.

### Memory Allocation
Memory is allocated on the heap when an object or primitive is instantiated. Salesforce enforces strict Heap Size limits (6MB synchronous, 12MB asynchronous).

### Naming Conventions
* Start with a letter.
* Use camelCase (e.g., `dealerName`, `totalInvoiceAmount`).
* Cannot contain spaces or special characters (except underscores, but cannot end with or have consecutive underscores).
* Cannot be a reserved keyword (e.g., `class`, `public`).

### Initialization and Default Values
If a variable is declared but not initialized, its default value is `null`.

### Examples
```apex
String name = 'Shibabrata';
Integer age = 25;
```
**Line-by-line Explanation:**
* `String name = 'Shibabrata';`: Declares a variable named `name` of type `String` and allocates memory to store the literal text value `'Shibabrata'`.
* `Integer age = 25;`: Declares a variable named `age` of type `Integer` and assigns it the numeric value `25`.

---

## 4. Variable Scope

Scope defines where a variable can be accessed within the code.

| Scope Type | Definition | Memory Lifecycle |
| :--- | :--- | :--- |
| **Local Variables** | Declared inside a method or loop. | Destroyed when the method/block ends. |
| **Instance Variables** | Declared in a class, outside methods. Require object instantiation. | Exists as long as the object instance exists. |
| **Static Variables** | Declared with `static`. Shared across all instances of the class within a transaction. | Exists for the duration of the Apex transaction. |
| **Global Variables** | Accessible outside the namespace, usually in `global` classes. | Exists per instance or transaction depending on if it's static. |

### Example
```apex
public class VehicleProcessor {
    public static Integer totalProcessed = 0; // Static scope
    public String dealerCode;                 // Instance scope

    public void processVehicle() {
        Integer counter = 1;                  // Local scope
        totalProcessed += counter;
    }
}
```

---

## 5. Primitive Data Types

| Type | Purpose | Storage/Precision | Default | Example | Common Use Cases |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Integer** | Whole numbers. | 32-bit | `null` | `100` | Loop counters, list sizes. |
| **Long** | Large whole numbers. | 64-bit | `null` | `2147483648L` | Large record counts, file sizes. |
| **Double** | Numbers with decimals. | 64-bit (Floating point) | `null` | `3.14159` | Scientific calculations, geolocation. |
| **Decimal** | Exact decimal numbers. | Arbitrary precision | `null` | `100.50` | Currency, financial calculations. |
| **Boolean** | Truth values. | True/False | `null` | `true` | Flags, conditional logic. |
| **String** | Text sequences. | Heap limit | `null` | `'Auto CRM'` | Names, descriptions. |
| **Date** | Date only (no time). | YYYY-MM-DD | `null` | `Date.newInstance(2026, 6, 25)` | Birthdates, delivery dates. |
| **Datetime** | Date and time. | GMT internally | `null` | `Datetime.now()` | Audit trails, SLA timers. |
| **Time** | Time only (no date). | HH:MM:SS.MS | `null` | `Time.newInstance(9, 21, 0, 0)` | Business hours logic. |
| **ID** | Salesforce Record ID. | 18-character | `null` | `'001xx000003D...'` | Foreign keys, record referencing. |
| **Blob** | Binary data. | Heap limit | `null` | `Blob.valueOf('data')` | Attachments, crypto hashes. |

---

## 6. Integer vs Long vs Decimal vs Double

| Feature | Integer | Long | Double | Decimal |
| :--- | :--- | :--- | :--- | :--- |
| **Size** | 32-bit | 64-bit | 64-bit | Arbitrary (varies) |
| **Decimals?**| No | No | Yes | Yes |
| **Precision**| Exact | Exact | Approximate (Lossy) | Exact (Lossless) |
| **Best For** | Counters | Big IDs | Fast math, Physics | Money, Currency |

### Why Decimal is Preferred for Money
`Double` uses binary floating-point representation, which cannot accurately represent some fractional numbers (e.g., 0.1), leading to rounding errors. `Decimal` handles exact arithmetic, making it mandatory for currency.

### Automotive CRM Examples
* **Warranty Claim Amount:** `Decimal claimAmount = 1500.75;`
* **Invoice Amount:** `Decimal invoiceTotal = 45000.00;`
* **Spare Order Price:** `Decimal partPrice = 125.50;`
* **Odometer Reading:** `Integer mileage = 45000;`

---

## 7. String Data Type

Strings are enclosed in single quotes. They are immutable (modifying a string creates a new string in memory).

### Key Methods
* `contains(substring)`: Returns true if the string contains the substring.
* `startsWith(prefix)`: Checks if it starts with the prefix.
* `endsWith(suffix)`: Checks if it ends with the suffix.
* `substring(startIndex, endIndex)`: Extracts a portion of the string.
* `replace(target, replacement)`: Replaces occurrences of a substring.
* `split(regex)`: Splits into a `List<String>`.
* `trim()`: Removes leading and trailing whitespace.
* `toUpperCase()` / `toLowerCase()`: Case conversion.
* `equals(string)`: Case-sensitive comparison.
* `equalsIgnoreCase(string)`: Case-insensitive comparison.

### Example
```apex
String vin = '  1HGCM82633A004   ';
String cleanVin = vin.trim().toUpperCase(); // Removes spaces, ensures uppercase
Boolean isValid = cleanVin.startsWith('1HG'); // true
```

---

## 8. Date, Datetime, and Time

| Feature | Date | Datetime | Time |
| :--- | :--- | :--- | :--- |
| **Stores** | Year, Month, Day | Date + Time | Hour, Min, Sec, Ms |
| **Timezone** | Context user's TZ | Stored in GMT | Context user's TZ |

### Time Zones and GMT Behavior
Salesforce stores all `Datetime` fields in **GMT** in the database. When displayed in the UI or printed via `System.debug()`, it is converted to the context user's Time Zone.

### Examples
```apex
Date deliveryDate = Date.newInstance(2026, 6, 25);
Datetime auditTime = Datetime.now(); // Captures current GMT time
Time shiftStart = Time.newInstance(8, 0, 0, 0); // 8:00 AM
```

---

## 9. ID Data Type

IDs are specialized strings representing 15-character or 18-character Salesforce record identifiers.

* **15-Digit ID:** Case-sensitive. Seen in the UI.
* **18-Digit ID:** Case-insensitive. The last 3 characters act as a checksum. Used in API and Apex.

Apex automatically handles conversions between 15 and 18-digit IDs. Using the `Id` type instead of `String` validates the ID format at runtime.

### Example
```apex
Id dealerId = '0015000000WvHXY'; // Validates formatting
```

---

## 10. Type Conversion

### Implicit Conversion
Salesforce automatically converts lower precision types to higher precision types (e.g., `Integer` to `Double`).

### Explicit Conversion (Casting)
Requires manual casting methods, typically using `valueOf()`.

### Examples
```apex
Integer num = Integer.valueOf('100');
String value = String.valueOf(100);
```
**Line-by-line Explanation:**
* `Integer num = Integer.valueOf('100');`: Takes the string literal `'100'`, parses it, and assigns the numerical integer value `100` to `num`.
* `String value = String.valueOf(100);`: Takes the numerical integer `100`, converts it to its text representation, and assigns `'100'` to the string variable `value`.

---

## 11. Constants (final Keyword)

The `final` keyword defines a constant. The variable can only be assigned a value once.
* **Immutable values:** Prevents accidental modification.
* **Best practices:** Name constants in `UPPER_SNAKE_CASE`.

### Example
```apex
public static final Decimal TAX_RATE = 0.08; // Cannot be changed later
```

---

## 12. Introduction to Collections

**Collections** are variables that can hold multiple records or elements.
* **Why they matter:** They allow batch processing of data.
* **Bulkification:** Processing records in collections prevents hitting DML and SOQL Governor Limits (e.g., max 100 SOQL queries, max 150 DML statements per transaction).

**Mental Model Diagram:**
```text
[ Record 1 ] \
[ Record 2 ] -- > [ Collection (List/Set/Map) ] -- > [ Single DML Operation ]
[ Record 3 ] /
```

---

## 13. Lists

A **List** is an ordered collection of elements that can contain duplicates. Access is done via a zero-based index.

### Concepts
* **Creating:** `List<Type> listName = new List<Type>();`
* **Adding/Removing:** Dynamic sizing.
* **Sorting:** Elements can be sorted using `.sort()`.
* **Iteration:** Commonly traversed using `for(Type var : listName)`.

### Example
```apex
List<String> carModels = new List<String>{'Civic', 'Accord'}; // Creation & Init
carModels.add('CR-V'); // Adding
String firstCar = carModels[0]; // Index Access (Civic)
```

---

## 14. List Methods

* `add(element)`: Appends an element to the end.
* `addAll(list)`: Appends all elements from another list or set.
* `remove(index)`: Removes the element at the specified index.
* `clear()`: Removes all elements.
* `contains(element)`: Returns true if the element exists.
* `size()`: Returns the number of elements.
* `sort()`: Sorts elements in ascending order.

### Example
```apex
List<Integer> mileageList = new List<Integer>{50000, 12000, 30000};
mileageList.sort(); // Orders to: 12000, 30000, 50000
Integer count = mileageList.size(); // returns 3
```

---

## 15. Sets

A **Set** is an unordered collection of elements that do not contain any duplicates.

### Deep Dive
* **Unique Values:** Automatically filters out duplicates upon insertion.
* **Performance:** `contains()` method in a Set is significantly faster (O(1)) than in a List (O(n)).
* **Common Use Cases:** Extracting parent IDs from trigger records to run a bulk SOQL query.

### Example
```apex
Set<Id> dealerIds = new Set<Id>();
dealerIds.add('0015000000WvHXY');
dealerIds.add('0015000000WvHXY'); // Ignored, already exists
```

---

## 16. Set Methods

* `add(element)`: Adds the element if it doesn't already exist.
* `remove(element)`: Removes the specific element.
* `contains(element)`: Extremely fast check for element existence.
* `size()`: Returns element count.
* `clear()`: Empties the set.

### Example
```apex
Set<String> validStatuses = new Set<String>{'Approved', 'Pending'};
Boolean canProcess = validStatuses.contains('Approved'); // true
```

---

## 17. Maps

A **Map** is a collection of key-value pairs where each unique key maps to a single value.

### Deep Dive
* **Keys:** Must be unique. Often use `Id` or `String`.
* **Values:** Can be primitive, objects, or other collections. Can contain duplicates.
* **Bulk Processing:** Essential for mapping a queried parent record to its ID so child records can access parent data in memory without inner SOQL queries.
* **Fast Lookup:** Retrieving a value by its key is incredibly fast.

### Example
```apex
Map<Id, Decimal> claimAmountsByDealer = new Map<Id, Decimal>();
```

---

## 18. Map Methods

* `put(key, value)`: Adds a new pair or overwrites the value if the key exists.
* `get(key)`: Retrieves the value for the given key.
* `containsKey(key)`: Checks if the key exists.
* `keySet()`: Returns a `Set` of all keys.
* `values()`: Returns a `List` of all values.
* `remove(key)`: Removes the key-value pair.
* `size()`: Returns the number of pairs.

### Example
```apex
Map<String, String> partNames = new Map<String, String>();
partNames.put('P-001', 'Brake Pad');
String part = partNames.get('P-001'); // returns 'Brake Pad'
```

---

## 19. Generic Collections

Collections are generic; you define the type of data they hold using angle brackets `<Type>`.

### Examples
```apex
List<Account> accountsToInsert = new List<Account>();
Set<Id> vehicleIds = new Set<Id>();
Map<Id, Contact> contactsById = new Map<Id, Contact>();
```
**Line-by-line Explanation:**
* `List<Account> accountsToInsert = new List<Account>();`: Initializes an empty List that can strictly hold only `Account` sObject records.
* `Set<Id> vehicleIds = new Set<Id>();`: Initializes an empty Set that holds unique Salesforce `Id` primitives.
* `Map<Id, Contact> contactsById = new Map<Id, Contact>();`: Initializes a Map where the lookup Key is an `Id` and the returned Value is a `Contact` sObject.

---

## 20. Nested Collections

Collections containing other collections. Used for complex data grouping.

### Examples
```apex
Map<Id, List<Contact>> contactsByAccountId = new Map<Id, List<Contact>>();
```
* **Use Case:** Grouping multiple child records under a single parent ID without querying the database multiple times. E.g., Mapping a `Dealer_Id` to a List of their `Mechanic` Contacts.

```apex
List<Map<String, Object>> rawJsonData = new List<Map<String, Object>>();
```
* **Use Case:** Parsing generic JSON arrays from REST API integrations.

---

## 21. Collections and Governor Limits

Collections are the absolute key to surviving Salesforce limits.
* **Heap Size:** Storing massive collections (e.g., List of 50,000 Accounts) will crash the transaction (Heap limit 6MB). Clear collections using `.clear()` if no longer needed.
* **CPU Time:** Iterating over large lists inside nested loops eats CPU time (10,000ms limit). Use Maps to replace inner loops.
* **Bulkification:** Accumulating records into a List and executing `insert listName;` uses 1 DML statement, whereas putting `insert var;` inside a loop will quickly hit the 150 DML limit.

---

## 22. Collections in Triggers

Triggers process records in batches of up to 200. You MUST use collections to handle them.

### Production-Quality Example (Automotive)
```apex
trigger WarrantyClaimTrigger on Warranty_Claim__c (before insert) {
    // 1. Gather all parent Vehicle IDs using a Set to ensure uniqueness
    Set<Id> vehicleIds = new Set<Id>();
    for (Warranty_Claim__c claim : Trigger.new) {
        if (claim.Vehicle__c != null) {
            vehicleIds.add(claim.Vehicle__c);
        }
    }

    // 2. Query parent data into a Map for fast lookup (No SOQL in loop!)
    Map<Id, Vehicle__c> vehicleMap = new Map<Id, Vehicle__c>(
        [SELECT Id, Warranty_Status__c FROM Vehicle__c WHERE Id IN :vehicleIds]
    );

    // 3. Process records using the Map
    for (Warranty_Claim__c claim : Trigger.new) {
        Vehicle__c parentVehicle = vehicleMap.get(claim.Vehicle__c);
        if (parentVehicle != null && parentVehicle.Warranty_Status__c == 'Expired') {
            claim.addError('Cannot file a claim on an expired warranty.');
        }
    }
}
```

---

## 23. Collections in LWC and Apex Controllers

When passing data to Lightning Web Components (LWC), Apex collections are serialized into JSON.

* **Serialization:** Apex Lists become JavaScript Arrays. Apex Maps become JavaScript Objects.
* **Wrapper Classes:** Often used to package multiple variables and collections into a single object for easy LWC consumption.

### Example
```apex
public class DealerDataWrapper {
    @AuraEnabled public String dealerName;
    @AuraEnabled public List<String> certifiedBrands;
}
```

---

## 24. Real Project Scenarios (Automotive CRM)

1.  **`Map<Id, Warranty_Claim__c>`:** Fast access to old trigger map (`Trigger.oldMap`) to check if a claim status changed.
2.  **`Set<Id> dealerIds`:** Collecting all unique Dealer IDs from an imported CSV of 500 spare part orders to query dealer discount levels.
3.  **`List<Invoice__c> invoicesToInsert`:** Accumulating new invoices generated at the end of the month to perform a single `insert` operation.
4.  **`Map<Id, List<Claim_Line__c>>`:** Organizing incoming Claim Lines by their Parent Claim ID to run calculation logic without querying the database again.

---

## 25. Performance Best Practices (Architect-Level)

* **O(1) Lookups:** Always use Maps for lookups instead of iterating through a List with an `if` statement.
* **De-duplication:** Use Sets instead of checking if a List `.contains()` an item (Set contains is optimized, List contains scans the whole array).
* **Avoid Nested Loops:** Loop over an outer list, build a map, then use the map. Nested loops cause exponential CPU time growth (O(n^2)).
* **Heap Optimization:** Use SOQL for loops (`for (Account acc : [SELECT Id FROM Account])`) for massive datasets; it batches memory chunking.

---

## 26. Common Mistakes and Solutions

| Mistake | Impact | Solution |
| :--- | :--- | :--- |
| **Null Collections** | `NullPointerException` on `.add()` | Always initialize: `List<Type> x = new List<Type>();` |
| **Using Lists for Unique IDs** | Duplicates cause SOQL errors | Use `Set<Id>` instead of `List<Id>`. |
| **SOQL inside a Loop** | Hits 100 SOQL limit fast | Extract IDs to a Set, query outside to a Map. |
| **Using Double for Money** | Rounding errors | ALWAYS use `Decimal` for currency. |
| **Hardcoding IDs** | Breaks in Production | Query dynamically or use Custom Labels/Metadata. |

---

## 27. Debugging Collections

* **System.debug():** Standard way to print to logs. `System.debug('Map Size: ' + myMap.size());`
* **Developer Console:** Check the "Debug Only" filter to see outputs.
* **Heap Limits:** If hitting limits, check the "Heap Allocations" in the Execution Log.
* **VS Code Debugging:** Use the Apex Replay Debugger to step through loops and inspect collection states variable by variable.

---

## 28. Interview Questions & Answers

### Beginner
**Q: What is the difference between a List and a Set?**
A: A List is ordered and allows duplicates. A Set is unordered and requires elements to be unique.

### Intermediate
**Q: Why do we use Maps in trigger contexts?**
A: To relate queried parent records to the child records currently in the trigger context. It provides O(1) fast lookup, avoiding the anti-pattern of putting SOQL queries inside `for` loops.

### Advanced
**Q: Explain how implicit type conversion works between Integer, Double, and Decimal.**
A: Apex implicitly casts lower precision types to higher ones (Integer to Double/Decimal). However, you cannot implicitly cast a Double to a Decimal or a Decimal to an Integer without explicit `.valueOf()` or `intValue()` casting to prevent accidental data loss.

### Architect-Level
**Q: You have a List of 100,000 Account records causing a Heap Size limit error. How do you resolve it?**
A: Process them asynchronously (Batch Apex), use a SOQL for-loop to chunk memory processing, remove unused variables by setting them to `null`, and clear collections `.clear()` as soon as they are no longer required to free up heap space.

---

## 29. Revision Summary

* **Variables:** Hold data in memory. Strongly typed. Default to `null`.
* **Primitive Types:** Integer (counters), Decimal (money), Boolean, String, Id, Datetime.
* **Type Conversion:** Implicit (upcasting), Explicit (manual casting using methods).
* **Lists:** Ordered, index-based, allows duplicates.
* **Sets:** Unordered, unique values, high-performance lookups.
* **Maps:** Key-Value pairs, crucial for bulkification and avoiding nested loops.
* **Collection Methods:** `add()`, `remove()`, `size()`, `clear()`, `get()`, `put()`.
* **Generic Collections:** Use `< >` to specify data types statically.
* **Nested Collections:** `Map<Id, List<sObject>>` handles parent-to-many-children logic.
* **Best Practices:** Initialize variables, strictly avoid SOQL/DML in loops, use Decimal for financial data (Auto CRM invoices, claims), and leverage Sets for ID uniqueness.