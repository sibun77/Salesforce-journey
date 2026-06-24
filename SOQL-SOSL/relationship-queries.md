# Relationship Queries

## 1. Introduction

### What Relationship Queries are
In Salesforce, a Relationship Query is a SOQL (Salesforce Object Query Language) statement that retrieves data from a primary object and its related objects in a single database trip. Instead of writing multiple separate queries and stitching the results together in memory, relationship queries leverage the platform's predefined data model to seamlessly traverse the links between records.

### Why Salesforce introduced Relationship Queries
Salesforce operates on a multi-tenant, metadata-driven architecture. Executing multiple disjointed queries consumes excessive database resources, network bandwidth, and governor limits. Relationship queries were introduced to:
* **Reduce Database Roundtrips:** Fetching a parent and its children at once reduces overhead.
* **Enforce Security at the Database Level:** The platform can evaluate sharing rules, Object-Level Security (OLS), and Field-Level Security (FLS) across the entire relationship chain efficiently.
* **Simplify Development:** Developers can access related data natively through dot notation (`.`) or nested queries without writing complex mapping logic in Apex.

### Querying One Object vs. Related Objects
* **Single Object:** `SELECT Id, Name FROM Vehicle__c` fetches only vehicle data.
* **Related Objects:** `SELECT Id, Name, Dealer__r.Name FROM Vehicle__c` fetches the vehicle *and* traverses the relationship to fetch the associated Dealer's name in the same transaction.

### Real-World Business Examples (Automotive CRM)
* **Parent-to-Child:** A Dealer wants to view an Account and all associated `Vehicle__c` records purchased by that customer.
* **Child-to-Parent:** A Warranty Claims Agent is looking at a `Claim_Line__c` and needs to see the parent `Warranty_Claim__c` status, the associated `Vehicle__c` VIN, and the `Dealer__c` name.

---

## 2. Salesforce Relationship Architecture

### Objects and Records
* **Objects:** Database tables defined by metadata (Standard or Custom).
* **Records:** Rows within those tables.

### Types of Relationships
* **Lookup Relationships:** A loosely coupled relationship. Deleting the parent does not delete the child. Security and ownership are maintained independently.
* **Master-Detail Relationships:** A tightly coupled relationship. Deleting the parent automatically deletes the child (Cascade Delete). The child inherits security and ownership from the master record. Roll-up summary fields can be created on the parent.
* **Junction Objects:** A custom object with two Master-Detail relationships, used to model a Many-to-Many relationship (e.g., `Vehicle_Feature__c` connecting `Vehicle__c` and `Feature__c`).
* **External Lookup:** Links a child standard, custom, or external object to a parent external object, mapped via External IDs.

### Internal Storage
Salesforce does not physically store objects as simple tables. Data is stored in massive, shared database tables (like `MTD_DATA` and `MTD_CUSTOM_ENTITY`) with a polymorphic design. Relationships are maintained via indexed Foreign Keys (Salesforce 15/18-character IDs). When a relationship is defined, Salesforce automatically generates the necessary metadata maps to resolve these IDs efficiently during a query.

### Architecture Diagram

```text
+----------------+       1:M        +--------------------+
|   Dealer__c    | <---------------+|     Vehicle__c     |
| (Parent/Lookup)|                  | (Child/Master)     |
+----------------+                  +---------+----------+
                                              |
                                              | 1:M
                                              v
                                    +--------------------+
                                    | Warranty_Claim__c  |
                                    | (Child/Detail)     |
                                    +--------------------+
```

---

## 3. What are Relationship Queries?

### Definition
Relationship Queries are SOQL statements that span multiple objects linked by lookup or master-detail relationships, fetching combined datasets.

### Purpose and Benefits
* **Governor Limit Conservation:** Helps stay under the strict 100 SOQL queries per synchronous transaction limit.
* **Heap Size Management:** Returns targeted data sets rather than fetching entire objects into Apex memory to perform programmatic filtering.
* **Readability:** Expresses complex business logic elegantly.

### Internal Metadata Usage
When a query is parsed, the SOQL engine consults the Org's Metadata Cache. It verifies that the requested relationship name exists, validates the user's access rights across the chain, and compiles the SOQL into optimized, underlying relational database SQL (Oracle/PostgreSQL) tailored to the multi-tenant schema.

### Why Salesforce Doesn't Support SQL JOIN Syntax
Traditional SQL `JOIN` clauses allow arbitrary joining of tables based on *any* criteria (e.g., `WHERE table1.name = table2.name`).
Salesforce forbids this to guarantee performance and security in a shared environment. By forcing developers to use predefined Relationship Queries (which act like implicit `LEFT OUTER JOIN` or `INNER JOIN` operations), Salesforce ensures that:
1.  Queries use indexed Foreign Keys.
2.  The Query Optimizer can accurately predict cost and execution plans.
3.  Sharing and security models are implicitly applied without complex nested `WHERE` clauses.

---

## 4. SOQL Relationship Fundamentals

### Relationship Fields vs Relationship Names
* **Relationship Field:** The actual field on the object storing the 18-character ID (e.g., `Dealer__c`).
* **Relationship Name:** The alias used in SOQL to traverse the relationship (e.g., `Dealer__r` or `Contacts`).

### API Names and Child Relationship Names
When you create a lookup field on a Child object pointing to a Parent:
* **Field Name:** `Vehicle__c` (on the Warranty Claim object).
* **Child Relationship Name:** `Warranty_Claims__r` (Plural, defined during field creation. Used by the Parent to query children).

### Describe Metadata
You can inspect these names programmatically in Apex:
```apex
// Describe the Account object to find Child Relationships
Schema.DescribeSObjectResult describeResult = Account.sObjectType.getDescribe();
for (Schema.ChildRelationship cr : describeResult.getChildRelationships()) {
    System.debug('Child Object: ' + cr.getChildSObject() + ' | Relationship Name: ' + cr.getRelationshipName());
}
```

---

## 5. Child-to-Parent Queries

### Concept
A child-to-parent query retrieves data from a base object and "reaches up" to its parent (or grandparent) to grab associated data. This is functionally an implicit `LEFT OUTER JOIN` (if the lookup is optional) or `INNER JOIN` (if the lookup/master-detail is required).

### Dot Notation and Traversing
You traverse from child to parent using a dot (`.`). For standard relationships, use the relationship name (usually the object name without `Id`). For custom relationships, replace `__c` with `__r`.

### Traversal Limits
You can traverse up to **5 levels** from child to parent.

### Examples and Explanations

```sql
SELECT Name, Account.Name
FROM Contact
```
* `SELECT Name`: Gets the Contact's name.
* `Account.Name`: Reaches up to the parent Account and retrieves its Name. (Level 1 traversal).
* `FROM Contact`: Defines the base object.

```sql
SELECT Subject, Account.Name
FROM Case
```
* `SELECT Subject`: Gets the Case Subject.
* `Account.Name`: Traverses the standard `AccountId` lookup field using the relationship name `Account` to get the Name.
* `FROM Case`: Base object is Case.

```sql
SELECT Name, Owner.Name
FROM Account
```
* `SELECT Name`: Gets the Account name.
* `Owner.Name`: Traverses the standard polymorphic `OwnerId` field to get the User or Queue name.
* `FROM Account`: Base object.

---

## 6. Parent-to-Child Queries

### Concept
A parent-to-child query retrieves a base parent record and all of its related child records. This is functionally an implicit `LEFT OUTER JOIN` coupled with an aggregate grouping, returning a collection of children for every parent row.

### Nested SELECT and Subqueries
This is achieved using a nested `SELECT` statement enclosed in parentheses within the main `SELECT` clause.

### Child Relationship Name
You *must* use the plural **Child Relationship Name** (e.g., `Contacts`, `Warranty_Claims__r`), not the object API name.

### Traversal Limits
You can traverse only **1 level down** in a single SOQL query. (You cannot do a subquery within a subquery).

### Example and Explanation

```sql
SELECT Name,
(
    SELECT LastName, Email
    FROM Contacts
)
FROM Account
```
* `SELECT Name,`: Retrieves the parent Account's Name.
* `(`: Opens the subquery for the child relationship.
* `SELECT LastName, Email`: Fields to retrieve from the child records.
* `FROM Contacts`: The standard Child Relationship Name linking Contact to Account.
* `)`: Closes the subquery. The result is returned as a `List<Contact>` attached to the Account record.
* `FROM Account`: Base object.

---

## 7. Relationship Names

### Standard vs Custom
* **Standard:** Typically the object name (e.g., `Account` going up, `Contacts` going down).
* **Custom:** Created by users. Always ends in `__r` when used as a relationship name in SOQL.

### How to Find the Child Relationship Name
1.  **Object Manager:** Go to the Child Object -> Fields & Relationships -> Click the Lookup Field -> Look at "Child Relationship Name". Append `__r` if it's a custom field.
2.  **Schema Builder:** Hover over the relationship line connecting the objects.
3.  **Describe API:** Use Apex `getChildRelationships()` as shown in Section 4.

---

## 8. Standard Object Relationship Examples

* **Account → Contacts:** `SELECT Name, (SELECT FirstName FROM Contacts) FROM Account`
* **Account → Opportunities:** `SELECT Name, (SELECT Amount FROM Opportunities) FROM Account`
* **Account → Cases:** `SELECT Name, (SELECT Status FROM Cases) FROM Account`
* **Opportunity → OpportunityLineItems:** `SELECT Name, (SELECT ListPrice FROM OpportunityLineItems) FROM Opportunity`
* **User → Cases (Created By):** `SELECT Name, (SELECT CaseNumber FROM CasesCreated) FROM User`

*Note: For standard objects, the relationship names are hardcoded by Salesforce. Always refer to the Object Reference documentation if unsure.*

---

## 9. Custom Object Relationships

### `__c` Fields vs `__r` Relationship References
When creating a custom relationship field `Dealer__c` on `Vehicle__c`:
* **`Dealer__c`**: The field that stores the 18-character ID. Use this in `WHERE` clauses. (e.g., `WHERE Dealer__c = '001...'`)
* **`Dealer__r`**: The relationship navigation object. Use this to get fields. (e.g., `SELECT Dealer__r.Name`)

### Automotive Example
**Scenario:** Fetch a Vehicle, its Dealer, and all its Warranty Claims.

```sql
SELECT 
    Name, 
    VIN__c, 
    Dealer__r.Name, 
    Dealer__r.BillingCity,
    (
        SELECT Claim_Number__c, Status__c 
        FROM Warranty_Claims__r
    )
FROM Vehicle__c
WHERE Dealer__r.Region__c = 'North America'
```

---

## 10. Lookup vs Master-Detail Queries

While SOQL syntax is identical for both, their behaviors differ regarding database execution and security.

| Feature | Lookup Relationship | Master-Detail Relationship |
| :--- | :--- | :--- |
| **Query Behavior** | Implicit `LEFT OUTER JOIN`. Child may return null parent fields. | Implicit `INNER JOIN` (if looking from child). Parent always exists. |
| **Cascade Delete** | No. Deleting parent sets lookup to NULL (if configured). | Yes. Deleting parent deletes all child records. |
| **Ownership** | Child has its own `OwnerId`. | Child inherits `OwnerId` from Parent. |
| **Security (Sharing)** | Child has separate Org-Wide Defaults (OWD). | Child is controlled by Parent OWD. |
| **Roll-Up Summaries** | Cannot be natively aggregated in parent SOQL. | Parent can have Roll-Up fields, easily queried. |

---

## 11. Polymorphic Relationships

### Concept
A polymorphic relationship is one where the referenced object can be of multiple different SObject types.

### Examples
* **`OwnerId`:** Can point to a `User` or a `Group` (Queue).
* **`WhoId`:** On Task/Event, points to a `Contact` or a `Lead`.
* **`WhatId`:** On Task/Event, points to an `Account`, `Opportunity`, `Case`, or custom object.

### Querying Polymorphic Fields
You can use `TYPEOF` to selectively query fields based on the actual object type.

```sql
SELECT Id,
  TYPEOF What
    WHEN Account THEN Phone, Industry
    WHEN Opportunity THEN Amount, StageName
    ELSE Name
  END
FROM Task
```
* `TYPEOF What`: Evaluates the polymorphic `WhatId` relationship.
* `WHEN Account THEN`: If it's an Account, fetch Phone and Industry.
* `ELSE Name`: Fallback field.

---

## 12. Junction Objects

### Many-to-Many Relationships
Salesforce models M:M using a Junction Object containing two Master-Detail relationships.
**Automotive Example:** `Vehicle__c` (M) --- `Vehicle_Feature__c` (Junction) --- `Feature__c` (M)

### Querying a Junction Object
To find all Features for a specific Vehicle, query the Junction object from the child-up:

```sql
SELECT Id, Feature__r.Name, Feature__r.Price__c
FROM Vehicle_Feature__c
WHERE Vehicle__c = 'a0X...'
```

To find a Vehicle and nested features (Parent-to-Child-to-Parent):
```sql
SELECT Name,
    (SELECT Feature__r.Name FROM Vehicle_Features__r)
FROM Vehicle__c
```

---

## 13. Multi-Level Relationship Queries

### Multiple Parent Traversal
You can chain dot notation to reach high up the hierarchy.
**Automotive Example:** `Warranty_Claim__c` -> `Vehicle__c` -> `Dealer__c` (Account) -> `Owner` (User).

```sql
SELECT 
    Name, 
    Vehicle__r.Dealer__r.Owner.Name
FROM Warranty_Claim__c
```
* `Name`: Claim Name
* `Vehicle__r.`: Up to Vehicle (Level 1)
* `Dealer__r.`: Up to Dealer Account (Level 2)
* `Owner.`: Up to Account Owner (Level 3)
* `Name`: The User's name.

---

## 14. Relationship Query Limits

| Limit Type | Maximum Allowed | Description |
| :--- | :--- | :--- |
| **Child-to-Parent Traversal** | 5 levels | E.g., `Contact.Account.Owner.Profile.Name` |
| **Parent-to-Child Subqueries** | 1 level down | Cannot do `SELECT (SELECT (SELECT...)))` |
| **Subqueries per Query** | 20 | Maximum of 20 child relationships in one query. |
| **Cross-Object References** | 55 per query | Total distinct relationships traversed. |
| **Returned Records (Sync)** | 50,000 | Standard transaction limit for all rows retrieved. |

---

## 15. Relationship Query Execution

### Internal Lifecycle
1.  **Parser & Metadata Resolution:** Checks syntax. Validates objects, fields, and `__r` names against metadata.
2.  **Security Evaluation:** Injects sharing rule constraints (e.g., appending implicit `AND OwnerId IN (...)`) based on user context.
3.  **Relationship Mapping:** Translates SOQL dot notation into underlying Oracle/Postgres `JOIN` paths using system-maintained foreign keys.
4.  **Query Optimizer:** Generates a query plan. Decides whether to use indexes (e.g., standard lookup indexes) or perform full table scans based on selectivity.
5.  **Database Execution:** Executes the mapped SQL.
6.  **Result Construction:** Packages the flat relational data back into nested SObject hierarchies for Apex/API consumption.

---

## 16. Query Optimization

### Relationship Selectivity
A query is selective if its filters target a small percentage of total records. The Optimizer prefers indexed fields (Lookups, External IDs, Master-Detail).
* **Bad:** `WHERE Vehicle__r.Color__c = 'Red'` (Cross-object formula or non-indexed traversal).
* **Good:** `WHERE Vehicle__c = 'a0X...'` (Directly querying the indexed ID field).

### Large Data Volumes (LDV)
In LDV scenarios (millions of records), parent-to-child queries can cause timeouts if a parent has thousands of children (Data Skew). Avoid querying child relationships where `Child Count > 10,000`.

### Query Plan Tool
Use the Developer Console Query Plan tool to view the cost of relationship queries. A cost > 1 indicates a full table scan, meaning your relationship filters are not selective enough.

---

## 17. Relationship Queries and Security

### `WITH SECURITY_ENFORCED` vs `WITH USER_MODE`
Apex runs in System Mode by default, bypassing FLS and OLS. To respect user permissions during relationship queries:

* **Legacy Approach:** `WITH SECURITY_ENFORCED`
    * Throws an exception if the user lacks read access to *any* field in the query, including relationship traversals.
* **Modern Approach (Spring '23+):** `WITH USER_MODE`
    * Replaces `WITH SECURITY_ENFORCED`. Respects OLS, FLS, and Sharing Rules.

```apex
// Apex Example using USER_MODE
List<Vehicle__c> vehicles = [
    SELECT Name, Dealer__r.Name 
    FROM Vehicle__c 
    WITH USER_MODE
];
```

---

## 18. Relationship Queries in Apex

### Single Record / Child-to-Parent
```apex
// Fetching a Claim and its parent Vehicle VIN
Warranty_Claim__c claim = [
    SELECT Id, Name, Vehicle__r.VIN__c 
    FROM Warranty_Claim__c 
    WHERE Id = :claimId 
    WITH USER_MODE 
    LIMIT 1
];
System.debug('VIN is: ' + claim.Vehicle__r.VIN__c);
```

### Multiple Records / Parent-to-Child (Bulk-Safe)
```apex
// Fetching Vehicles and their Claims
List<Vehicle__c> vehicles = [
    SELECT Id, Name, 
        (SELECT Id, Status__c FROM Warranty_Claims__r WHERE Status__c = 'Open')
    FROM Vehicle__c
    WHERE Dealer__c IN :dealerIds
    WITH USER_MODE
];

// Iterating safely
for (Vehicle__c veh : vehicles) {
    System.debug('Vehicle: ' + veh.Name);
    // The nested query returns a List<Warranty_Claim__c>
    for (Warranty_Claim__c claim : veh.Warranty_Claims__r) {
        System.debug('Open Claim: ' + claim.Id);
    }
}
```
* `WHERE Dealer__c IN :dealerIds`: Standard bulkification pattern, filtering by a Set of IDs.
* `veh.Warranty_Claims__r`: Accessing the list of children in memory.

---

## 19. Relationship Queries in LWC

### Using `@wire` with Apex
```javascript
import { LightningElement, wire } from 'lwc';
import getVehicles from '@salesforce/apex/VehicleController.getVehicles';

export default class VehicleList extends LightningElement {
    @wire(getVehicles)
    vehicles;

    // In LWC HTML template, traverse relationships using dot notation:
    // {vehicle.Dealer__r.Name}
}
```

### UI API Limitations
The standard UI API (`getRecord`) has limitations on deep relationship traversals and cannot execute nested parent-to-child SOQL directly. For complex relationships, an Imperative Apex call returning structured wrapper classes or serialized JSON is best practice.

---

## 20. Relationship Queries in REST API

### Standard REST Query
Encode the SOQL query in the URL:
`GET /services/data/v58.0/query?q=SELECT+Name,+(SELECT+LastName+FROM+Contacts)+FROM+Account`
Returns a JSON structure where the child relationship is represented as a nested `records` array.

### GraphQL API
Salesforce's modern approach for mobile and UI development, allowing deeply nested relationship querying in a single API request without strictly writing SOQL:
```graphql
query {
  uiapi {
    query {
      Vehicle__c {
        edges {
          node {
            Name { value }
            Dealer__r {
              Name { value }
            }
          }
        }
      }
    }
  }
}
```

---

## 21. Common Mistakes

### 1. Confusing `__c` with `__r`
* **Error:** `SELECT Dealer__c.Name FROM Vehicle__c`
* **Fix:** `SELECT Dealer__r.Name FROM Vehicle__c`

### 2. SOQL Inside Loops
* **Error:** Querying parent records, looping through them, and querying children inside the loop.
* **Fix:** Use a single Parent-to-Child query with a subquery, or query children using `WHERE ParentId IN :parentIds`.

### 3. Null Pointer Exceptions on Traversals
* **Error:** `String city = claim.Vehicle__r.Dealer__r.BillingCity;` (Throws NPE if Vehicle__c or Dealer__c is null).
* **Fix:** Always use safe navigation `?.` in Apex:
    `String city = claim.Vehicle__r?.Dealer__r?.BillingCity;`

---

## 22. Real Project Scenarios (Automotive CRM)

### Scenario 1: Invoice Generation
**Goal:** Generate a final invoice containing the Customer, the Work Order, and all Spare Parts used.
**Query:**
```sql
SELECT 
    Invoice_Number__c, 
    Customer__r.Name,
    Customer__r.MailingAddress,
    Work_Order__r.Total_Labor_Hours__c,
    (
        SELECT Spare_Part__r.Part_Number__c, Quantity__c, Line_Total__c 
        FROM Spare_Orders__r
    )
FROM Invoice__c
WHERE Id = :invoiceId
```
**Why:** Grabs 3 levels of parent data (Invoice, Customer, Work Order) and 1 level of child data (Spare Orders, traversing up to Part Number) in a single database hit, perfectly structuring the data for a PDF generation engine.

---

## 23. Performance Considerations

* **Data Skew:** If one Dealer owns 50,000 Vehicles, running `SELECT Name, (SELECT Id FROM Vehicles__r) FROM Account` will throw a `System.QueryException: Aggregate query has too many rows for direct assignment`.
* **Heap Usage:** Nesting massive child queries consumes Apex Heap space quickly (Limit: 6MB sync). If a parent has many children, do a direct Child query instead: `SELECT Id FROM Vehicle__c WHERE Dealer__c = :dealerId`.
* **Indexes:** Remember that cross-object fields (Formula fields referencing parent data) cannot usually be indexed. Avoid `WHERE Vehicle__r.Formula_Field__c = 'X'`.

---

## 24. Best Practices

1.  **Selectivity is King:** Always filter on the primary object using indexed fields.
2.  **Avoid SELECT *:** Salesforce doesn't support `SELECT *`, but avoid querying fields you don't need, especially large text areas within subqueries.
3.  **Meaningful Naming:** If you have multiple lookups to the same object (e.g., `Selling_Dealer__c` and `Servicing_Dealer__c`), name the child relationships clearly (`Vehicles_Sold__r` and `Vehicles_Serviced__r`).
4.  **Use Maps for Data Stitching:** If a relationship query is too complex or hits limits, query parents into a `Map<Id, Parent__c>`, query children independently, and stitch them together in memory using Apex loops.

---

## 25. Debugging & Troubleshooting Relationship Queries

### Diagnosing Query Issues
When relationship queries fail or return unexpected results, use these techniques to isolate the issue:

1.  **"Didn't understand relationship" Error:**
    * *Cause:* Incorrect relationship name.
    * *Fix:* Open the Developer Console -> File -> Open Resource -> search for the object. Look at the schema definition to verify the exact spelling and suffix (`__r` vs `__c`).

2.  **Null Values in Child-to-Parent Fields:**
    * *Cause:* The lookup field is empty on the record, or Field-Level Security (FLS) is hiding the parent field from the running user.
    * *Fix:* Run the query in Developer Console using "Query Editor". If the field is populated there but null in Apex (when using `WITH USER_MODE`), check Profile/Permission Set FLS for that parent field.

3.  **"Query is too complex" Error:**
    * *Cause:* Combining too many parent/child traversals with complex `OR` filters.
    * *Fix:* Break the query down. Use the **Query Plan Tool** in Developer Console (Enable via Help -> Preferences -> Enable Query Plan) to evaluate the query cost. If cost > 1, add selective indexed filters.

4.  **Workbench & VS Code:**
    * Use the **SOQL Builder** in VS Code to auto-complete relationship names. This relies on your local downloaded metadata and prevents typo-related errors.
    * Use **Workbench** (REST Explorer or SOQL Query tab) to execute queries as specific users to test sharing rule visibility.

---

## 26. Interview Questions & Answers

### Beginner
**Q: How do you traverse from a Custom Object to a Standard Object in SOQL?**
A: Use dot notation and replace the `__c` of the lookup field with `__r`. Example: `Account__r.Name`.

**Q: Can you perform a subquery inside a subquery?**
A: No, Salesforce restricts Parent-to-Child subqueries to one level deep.

### Intermediate
**Q: Explain the difference between `__c` and `__r`.**
A: `__c` represents the custom field storing the ID (used for assignment and filtering). `__r` represents the relationship reference used to traverse to the related object's fields in SOQL.

**Q: What is a Polymorphic Query?**
A: A query on a relationship field that can reference multiple object types, like `OwnerId` or `WhatId`, often requiring the `TYPEOF` clause to pull specific fields based on the record type.

### Advanced / Architect
**Q: You have a batch class querying Accounts and a subquery fetching thousands of related Contacts. It occasionally fails with heap limits. How do you redesign it?**
A: Replace the parent-to-child query with a child-to-parent query. Query the Contacts directly, filtering by the Account IDs in scope, or use an Inner Join approach. If the parent-to-child query must remain, use a `FOR` loop query (`for(Account a : [SELECT...])`) to allow Salesforce to chunk the memory automatically.

**Q: How does the Salesforce Optimizer handle a cross-object `WHERE` clause (e.g., `WHERE Contact.Account.Industry = 'Tech'`)?**
A: The query optimizer struggles to use indexes on cross-object fields. It generally results in a full table scan of the child object. To optimize, create an indexed formula field or denormalize the data by using a trigger to stamp the parent value directly on the child object, then query that indexed child field.

---

## 27. Revision Summary

* **Child-to-Parent:** Uses dot notation (`.`), max 5 levels up. Acts like an implicit Left Outer / Inner Join.
* **Parent-to-Child:** Uses nested `(SELECT...)`, max 1 level down. Requires Plural Child Relationship Name.
* **Naming:** Custom lookups use `__c` for ID, `__r` for traversal.
* **Lookup vs Master-Detail:** M-D ensures child security is controlled by parent; Lookup operates independently.
* **Security:** Always use `WITH USER_MODE` in Apex to enforce user permissions.
* **Performance:** Avoid cross-object `WHERE` filters on large datasets. Be mindful of Data Skew causing massive subquery returns. Use Safe Navigation `?.` in Apex to prevent Null Pointer Exceptions.