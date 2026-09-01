# Backend Pagination – Server-Side Pagination

# 1. Introduction

When building enterprise Salesforce applications, you frequently need to display large lists of records—such as **Warranty Claims**, **Work Orders**, **Spare Parts**, **Invoices**, **Dealers**, **Customers**, **Vehicles**, **Shipments**, or **Service History**. 

**Pagination** is the process of dividing this large dataset into smaller, discrete pages rather than showing everything on a single screen.

**Backend Pagination** (or **Server-Side Pagination**) means that the database (Salesforce) only retrieves and sends the exact subset of records needed for the current page. The client (LWC) does not possess the entire dataset.

Server-side pagination is critical in Salesforce because loading every record into the browser is not scalable. Attempting to load 50,000 warranty claims into the LWC causes:
*   **Heap Size Errors** in Apex (limit is 6MB).
*   **CPU Time Out Errors** (limit is 10,000ms).
*   **Massive JSON Payloads** that freeze the browser.
*   **Terrible User Experience** due to long initial loading times.

Server-side pagination prevents this by ensuring the browser only ever handles exactly what the user is looking at (e.g., 50 records at a time).

---

# 2. What is Server-Side Pagination?

Server-side pagination offloads the data-slicing work to the Salesforce database. Instead of the UI requesting all records and hiding the ones not currently on the page, the LWC asks Apex for an exact "page" of records starting from a specific index.

### Architecture Flow Diagram

```text
User clicks "Next Page"
       ↓
LWC (Calculates requested page number and size)
       ↓
Apex (Receives pageNumber and pageSize parameters)
       ↓
SOQL (Executes query dynamically)
       ↓
LIMIT + OFFSET / Keyset (Filters data strictly at database level)
       ↓
Salesforce Database
       ↓
Only Required Records (e.g., exactly 25 records)
       ↓
Apex Response (Returns records + Total count)
       ↓
LWC (Updates UI state and records array)
       ↓
UI (Renders the data table)
```

1. **User interaction:** The user initiates an action (loads page, clicks Next, changes filter).
2. **LWC Request:** The component calculates what chunk of data it needs.
3. **Apex & SOQL:** The server executes a selective query using SOQL limits and offsets to fetch only that specific chunk.
4. **Response:** A small, lightweight payload is returned to the client.

---

# 3. Why Use Server-Side Pagination?

### Advantages
* **Handles large datasets:** Can navigate through thousands or millions of records securely.
* **Reduces browser memory usage:** The LWC only stores the current page's data in memory (e.g., 50 records instead of 10,000).
* **Reduces response payload:** Network requests are tiny, preventing browser freezing.
* **Better initial loading performance:** First Contentful Paint (FCP) is lightning fast.
* **Better scalability:** Absolutely mandatory for enterprise applications processing Large Data Volumes (LDV).
* **Works well with large Salesforce objects:** Ideal for transactional objects like Invoices and Work Orders.

### Disadvantages
* **Requires server request for page navigation:** Clicking "Next" requires a network round-trip, introducing slight latency.
* **More Apex complexity:** Requires managing state, `COUNT()` queries, and dynamic SOQL.
* **Query optimization is required:** Non-selective queries can still time out.
* **OFFSET has limitations:** Standard Salesforce `OFFSET` maxes out at 2,000 records.
* **Additional server calls:** Rapid clicking can cause duplicate requests if not handled properly.

---

# 4. Client-Side vs Server-Side Pagination

| Feature | Client-Side Pagination | Server-Side Pagination |
| :--- | :--- | :--- |
| **Data Retrieval** | Retrieves ALL records at once. | Retrieves ONLY the current page's records. |
| **Browser Memory** | Extremely High (stores all records). | Low (stores only current page). |
| **Server Requests** | One massive, slow request. | Small, fast requests per page navigation. |
| **Initial Load** | Very Slow (waiting for all data). | Very Fast (waiting for one page). |
| **Page Navigation**| Instant (no network call). | Requires network call (slight latency). |
| **Large Datasets** | Crashes browser / Hits Apex limits. | Highly scalable. |
| **SOQL** | Standard `SELECT Id FROM Object`. | Uses `LIMIT`, `OFFSET`, or Keyset logic. |
| **LIMIT / OFFSET** | Not used. | Heavily utilized. |
| **Performance** | Terrible as data grows. | Consistent regardless of total data size. |
| **Complexity** | Low. | High. |
| **Use Cases** | Setup objects, config data (< 1,000 rows). | Warranty Claims, Work Orders (> 1,000 rows). |

**Summary:** 
Client-side pagination divides **already-loaded** records. 
Server-side pagination retrieves **only the records required** for the current page from the database.

---

# 5. Basic Pagination Concepts

* **Total Records:** The total count of records matching the current filters (e.g., 1,250 claims).
* **Page Size:** How many records display per page (e.g., 50).
* **Total Pages:** The total number of available pages (Total Records / Page Size).
* **Current Page:** The page number the user is currently viewing (e.g., Page 2).
* **Limit:** The maximum number of records to return in the SOQL query (equals Page Size).
* **Offset:** How many records to skip before starting to return records in the SOQL query.
* **Start Record:** The index of the first record on the current page.
* **End Record:** The index of the last record on the current page.

**Example:**
Total Records = 1,250
Page Size = 50

* Page 1 → Records 1–50
* Page 2 → Records 51–100
* Page 3 → Records 101–150

When requesting Page 3, the server skips the first 100 records and returns the next 50.

---

# 6. Pagination Formula

To implement server-side pagination, you must calculate these values dynamically in Apex or JS:

**Offset:**
`(pageNumber - 1) * pageSize`
*Example (Page 3, Size 50):* `(3 - 1) * 50 = 100`. (Skip the first 100 records).

**Limit:**
`pageSize`
*Example:* `50`.

**Total Pages:**
`Math.ceil(totalRecords / pageSize)`
*Example:* `Math.ceil(1250 / 50) = 25`. (Always round up; 1201 records with size 50 is 25 pages).

**Start Record:**
`((pageNumber - 1) * pageSize) + 1`
*Example (Page 3):* `((3 - 1) * 50) + 1 = 101`.

**End Record:**
`Math.min(pageNumber * pageSize, totalRecords)`
*Example (Page 25):* `Math.min(25 * 50, 1250) = 1250`. (Prevents showing "1201-1250" if only 1210 records exist).

---

# 7. SOQL LIMIT

`LIMIT` specifies the maximum number of rows a SOQL query will return.

**Example:**
```sql
SELECT Id, Name, Status__c FROM Warranty_Claim__c LIMIT 10
```

*   **Purpose:** Restricts dataset size, improving query speed and mapping directly to your **Page Size**.
*   **Dynamic LIMIT:** In Apex, you bind it using a variable: `LIMIT :pageSize`.
*   **Performance:** Queries with a `LIMIT` execute faster because the database stops searching once the limit is met (assuming proper indexes).

---

# 8. SOQL OFFSET

`OFFSET` specifies the starting row index into the result set returned by your query.

**Example:**
```sql
SELECT Id, Name FROM Warranty_Claim__c ORDER BY Name ASC LIMIT 10 OFFSET 20
```

*   **Purpose:** Skips a specific number of rows before returning the result.
*   **How OFFSET works:** The database calculates the full result set (based on filters), skips the first N rows, and returns the rest (up to the LIMIT).
*   **Relationship with LIMIT:** They are used together. `LIMIT 10 OFFSET 20` means "Skip 20 records, then give me the next 10."
*   **Ordering requirements:** You **MUST** use an `ORDER BY` clause when using `OFFSET`. Without it, Salesforce does not guarantee row order, meaning records could randomly appear on multiple pages or be skipped entirely.

---

# 9. LIMIT + OFFSET Pagination

This is the standard architectural pattern for LWC server-side pagination.

**Flow:**
Page Number (e.g., 3) 
       ↓
Calculate OFFSET ( (3 - 1) * 10 = 20 )
       ↓
Apex Request (pageSize = 10, offset = 20)
       ↓
SOQL `LIMIT 10 OFFSET 20`
       ↓
Return Records (Records 21-30)
       ↓
LWC Renders Datatable

---

# 10. Basic Apex Pagination Method

```java
@AuraEnabled(cacheable=true)
public static List<Warranty_Claim__c> getClaims(Integer pageSize, Integer pageNumber) {
    // Calculate how many records to skip
    Integer offsetValue = (pageNumber - 1) * pageSize;

    // Execute query with bounded limits
    return [
        SELECT Id, Name, Status__c
        FROM Warranty_Claim__c
        WITH USER_MODE
        ORDER BY CreatedDate DESC
        LIMIT :pageSize
        OFFSET :offsetValue
    ];
}
```
**Explanation:**
1.  `offsetValue` translates the human-readable `pageNumber` into a database-readable row skip count.
2.  `WITH USER_MODE` ensures Object and Field-Level Security (FLS) are respected.
3.  `ORDER BY CreatedDate DESC` guarantees deterministic sorting. Without this, pagination is broken.
4.  `LIMIT` bounds the payload size.
5.  `OFFSET` shifts the starting point.

---

# 11. Returning Total Record Count

The LWC needs to know the total number of records to calculate `Total Pages` and to know when to disable the "Next" button.

```java
@AuraEnabled(cacheable=true)
public static Integer getTotalClaims() {
    return [
        SELECT COUNT() 
        FROM Warranty_Claim__c 
        WITH USER_MODE
    ];
}
```
**Explanation:** 
`COUNT()` is an aggregate function that returns an Integer representing the number of rows matching the query. It is much faster and consumes less heap space than querying the actual records. The LWC uses this to calculate total pages.

---

# 12. Returning Records and Count Together

Making two separate server calls from LWC (one for records, one for count) is inefficient and can lead to race conditions. Instead, use a **Wrapper Class** to return everything in a single payload.

```java
public class PaginationResult {
    @AuraEnabled public List<Warranty_Claim__c> records;
    @AuraEnabled public Integer totalRecords;
    @AuraEnabled public Integer pageNumber;
    @AuraEnabled public Integer pageSize;
    @AuraEnabled public Integer totalPages;
}
```

---

# 13. Wrapper Class for Pagination

*   **Why wrapper classes are useful:** They bundle diverse data types (Lists, Integers, Strings) into a single object that Apex can serialize into a JSON response.
*   **Avoiding multiple server calls:** One network trip fetches the chunk of data AND the metadata (total records) required to render the UI.
*   **LWC Access:** LWC handles the response as a single JavaScript Object (`result.records`, `result.totalRecords`).

---

# 14. Complete Basic Server-Side Pagination Example

**Scenario:** Warranty Claim List. 10 records per page.

**Apex (WarrantyClaimController.cls):**
```java
public with sharing class WarrantyClaimController {
    
    public class PaginationResult {
        @AuraEnabled public List<Warranty_Claim__c> records;
        @AuraEnabled public Integer totalRecords;
    }

    @AuraEnabled(cacheable=true)
    public static PaginationResult getPagedClaims(Integer pageNumber, Integer pageSize) {
        PaginationResult result = new PaginationResult();
        
        // 1. Get Total Count
        result.totalRecords = [SELECT COUNT() FROM Warranty_Claim__c WITH USER_MODE];
        
        // 2. Calculate Offset
        Integer offset = (pageNumber - 1) * pageSize;
        
        // 3. Get Records
        result.records = [
            SELECT Id, Name, Status__c, Claim_Amount__c 
            FROM Warranty_Claim__c 
            WITH USER_MODE 
            ORDER BY CreatedDate DESC 
            LIMIT :pageSize 
            OFFSET :offset
        ];
        
        return result;
    }
}
```

**JavaScript (warrantyClaimList.js):**
```javascript
import { LightningElement, track } from 'lwc';
import getPagedClaims from '@salesforce/apex/WarrantyClaimController.getPagedClaims';

export default class WarrantyClaimList extends LightningElement {
    @track records = [];
    currentPage = 1;
    pageSize = 10;
    totalRecords = 0;
    totalPages = 0;
    isLoading = false;
    error = null;

    connectedCallback() {
        this.loadRecords();
    }

    async loadRecords() {
        this.isLoading = true;
        this.error = null;
        try {
            const result = await getPagedClaims({ 
                pageNumber: this.currentPage, 
                pageSize: this.pageSize 
            });
            this.records = result.records;
            this.totalRecords = result.totalRecords;
            this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
        } catch (error) {
            this.error = 'Error loading records';
            this.records = [];
        } finally {
            this.isLoading = false;
        }
    }

    handlePrevious() {
        if (this.currentPage > 1) {
            this.currentPage--;
            this.loadRecords();
        }
    }

    handleNext() {
        if (this.currentPage < this.totalPages) {
            this.currentPage++;
            this.loadRecords();
        }
    }

    get isFirstPage() { return this.currentPage === 1; }
    get isLastPage() { return this.currentPage === this.totalPages || this.totalPages === 0; }
    get hasRecords() { return this.records.length > 0; }
}
```

**HTML (warrantyClaimList.html):**
```html
<template>
    <lightning-card title="Warranty Claims">
        
        <!-- Loading Spinner -->
        <template if:true={isLoading}>
            <lightning-spinner alternative-text="Loading" size="medium"></lightning-spinner>
        </template>

        <!-- Error State -->
        <template if:true={error}>
            <div class="slds-text-color_error slds-p-around_medium">{error}</div>
        </template>

        <!-- Data Table -->
        <template if:true={hasRecords}>
            <ul class="slds-m-around_medium">
                <template for:each={records} for:item="claim">
                    <li key={claim.Id}>{claim.Name} - {claim.Status__c}</li>
                </template>
            </ul>
        </template>

        <!-- Empty State -->
        <template if:false={hasRecords}>
            <template if:false={isLoading}>
                <div class="slds-p-around_medium">No warranty claims found.</div>
            </template>
        </template>

        <!-- Pagination Controls -->
        <div class="slds-p-around_medium slds-grid slds-grid_align-spread slds-grid_vertical-align-center">
            <lightning-button label="Previous" disabled={isFirstPage} onclick={handlePrevious}></lightning-button>
            <span>Page {currentPage} of {totalPages}</span>
            <lightning-button label="Next" disabled={isLastPage} onclick={handleNext}></lightning-button>
        </div>

    </lightning-card>
</template>
```

---

# 15. LWC Pagination State

To manage pagination properly, your LWC must track these core properties:
*   `records = []`: Stores the array of SObjects for the current page.
*   `currentPage = 1`: The state integer pointing to the user's current location.
*   `pageSize = 10`: The constant or user-selected volume of records per chunk.
*   `totalRecords = 0`: The absolute sum of all rows matching the criteria.
*   `totalPages = 0`: Calculated dynamically (`Math.ceil(totalRecords/pageSize)`).
*   `isLoading = false`: Boolean gatekeeper preventing UI interaction during server transit.

---

# 16. Calling Apex with Imperative Apex

```javascript
try {
    const result = await getClaims({ 
        pageNumber: this.currentPage, 
        pageSize: this.pageSize 
    });
} catch (error) { ... }
```
*   **Manual server call:** Imperative Apex allows you to dictate *exactly when* the server is called (e.g., explicitly on a button click).
*   **async/await:** Modern JS pattern that pauses execution until the Promise resolves, making code read synchronously.
*   **Error handling:** Standard `try/catch` block safely handles Apex exceptions or network drops.

---

# 17. Pagination Using Wire Service

Server-side pagination can also be achieved reactively using `@wire`.

```javascript
@wire(getClaims, { 
    pageNumber: '$currentPage', 
    pageSize: '$pageSize' 
})
wiredClaims({ data, error }) {
    if (data) {
        this.records = data.records;
        this.totalRecords = data.totalRecords;
        this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
    } else if (error) {
        this.error = error;
    }
}
```
*   **Reactive parameters (`$`):** When `this.currentPage` changes via `handleNext()`, the wire service automatically triggers a new Apex call.
*   **Cacheable Apex:** Required for `@wire`. Fast reads from Lightning Data Service cache.

---

# 18. Wire vs Imperative Apex for Pagination

| Feature | Wire Service (`@wire`) | Imperative Apex (`async/await`) |
| :--- | :--- | :--- |
| **Trigger** | Automatic (reactive to `$` variables). | Manual (called via JS functions). |
| **Page Navigation** | Just change `currentPage`. | Change `currentPage`, then call function. |
| **Reactivity** | High. | Controlled. |
| **Caching** | Enforced (`cacheable=true`). | Optional. |
| **Error Handling** | Handled in wire provisioner logic. | Handled via `try/catch`. |
| **DML Support** | Cannot perform DML. | Can call non-cacheable DML methods. |
| **Best Use Cases**| Simple, read-only paginated lists. | Complex grids with search, heavy filters, or row actions. |

**Architect Note:** Imperative Apex is overwhelmingly preferred for enterprise datatables because it provides explicit control over loading spinners, debounce timing, and error handling.

---

# 19. Next Page

```javascript
handleNext() {
    if (this.currentPage < this.totalPages) {
        this.currentPage++;
        this.loadRecords();
    }
}
```
*   **Check last page:** Always ensure `currentPage` cannot exceed `totalPages`.
*   **Increment:** Add 1 to state.
*   **Call Apex:** Fetch the next chunk.

---

# 20. Previous Page

```javascript
handlePrevious() {
    if (this.currentPage > 1) {
        this.currentPage--;
        this.loadRecords();
    }
}
```
*   **Prevent below 1:** Page 0 or negative pages will cause Apex OFFSET errors (OFFSET cannot be negative).

---

# 21. First Page

```javascript
handleFirst() {
    this.currentPage = 1;
    this.loadRecords();
}
```
Jumps directly to the start. Offset becomes 0.

---

# 22. Last Page

```javascript
handleLast() {
    this.currentPage = this.totalPages;
    this.loadRecords();
}
```
Jumps directly to the end. Offset becomes `(totalPages - 1) * pageSize`.

---

# 23. Disabling Pagination Buttons

You must prevent users from navigating out of bounds. Use JS getters bound to the `disabled` attribute in HTML.

```javascript
get isFirstPage() {
    return this.currentPage === 1;
}

get isLastPage() {
    return this.currentPage === this.totalPages || this.totalPages === 0;
}
```
```html
<lightning-button label="First" disabled={isFirstPage} onclick={handleFirst}></lightning-button>
<lightning-button label="Prev" disabled={isFirstPage} onclick={handlePrevious}></lightning-button>
<lightning-button label="Next" disabled={isLastPage} onclick={handleNext}></lightning-button>
<lightning-button label="Last" disabled={isLastPage} onclick={handleLast}></lightning-button>
```

---

# 24. Pagination Information

Users need context on their location within the dataset: *"Showing 101–150 of 1,250 records"*

```javascript
get startRecord() {
    return this.totalRecords === 0 ? 0 : ((this.currentPage - 1) * this.pageSize) + 1;
}

get endRecord() {
    return Math.min(this.currentPage * this.pageSize, this.totalRecords);
}
```
*   **Empty Result Handle:** If `totalRecords` is 0, start record is 0, not 1.
*   **Last Page Handle:** `Math.min` ensures that if page 3 ends at 150, but total records is 142, the UI displays "101-142", not "101-150".

---

# 25. Page Size Selection

Allowing users to choose data density.

```html
<lightning-combobox 
    label="Records per page" 
    value={pageSize} 
    options={pageSizeOptions} 
    onchange={handlePageSizeChange}>
</lightning-combobox>
```

```javascript
pageSizeOptions = [
    { label: '10', value: 10 },
    { label: '25', value: 25 },
    { label: '50', value: 50 },
    { label: '100', value: 100 }
];

handlePageSizeChange(event) {
    this.pageSize = parseInt(event.detail.value, 10);
    this.currentPage = 1; // CRITICAL STEP
    this.loadRecords();
}
```
**Recommended behavior:** When page size changes, chunk boundaries are destroyed. You **must** reset `currentPage` to 1. If you were on page 5 with size 10 (record 50), and switch to size 100, page 5 no longer exists.

---

# 26. Page Number Buttons

Implementing direct page navigation: `1 2 3 4 5 ... 25`

This requires calculating an array of visible pages based on `currentPage`. If total pages = 100, you cannot render 100 buttons. 
You must generate a sliding window:
* If page is 1: `[1, 2, 3, 4, 5]`
* If page is 10: `[8, 9, 10, 11, 12]`

Clicking a number passes that specific integer as `pageNumber` to Apex, updating the `OFFSET` dynamically.

---

# 27. Search with Server-Side Pagination

When dealing with large datasets, **filtering cannot happen on the client**. If you search the client array, you only search the current 50 records, missing the other 9,950 in the database.

**Flow:**
User Search → LWC → Apex → SOQL `WHERE Name LIKE :searchKey LIMIT :size OFFSET :offset` → Results

**Example Apex:**
```java
String searchPattern = '%' + searchKey + '%';
String query = 'SELECT Id, Name FROM Warranty_Claim__c WHERE Name LIKE :searchPattern LIMIT :pageSize OFFSET :offset';
```

---

# 28. Filtering with Server-Side Pagination

Filtering by specific dropdowns (Status, Dealer, Date).

**Example:**
```sql
SELECT Id FROM Warranty_Claim__c WHERE Status__c = :statusFilter
```
*   **Total Count Impact:** Filters drastically alter the `totalRecords`.
*   **Reset Requirement:** When a filter changes, the total pages shrink or grow. You **must reset `currentPage = 1`**. If you are on page 10 of "All Claims" and filter to "Rejected", there might only be 2 pages of Rejected claims. Staying on page 10 results in an empty UI.

---

# 29. Sorting with Server-Side Pagination

To sort server-side, you must use **Dynamic SOQL** because bind variables cannot be used for field names or ASC/DESC keywords.

```java
String query = 'SELECT Id FROM Warranty_Claim__c ORDER BY ' + sortField + ' ' + sortDir;
```

**CRITICAL SECURITY WARNING:**
User-provided field names must **NEVER** be inserted directly into Dynamic SOQL. This opens the door to SOQL Injection. You must use a **Whitelist**.

```java
// Secure Whitelisting Example
Set<String> validSortFields = new Set<String>{'Name', 'CreatedDate', 'Status__c', 'Claim_Amount__c'};
if (!validSortFields.contains(sortField)) {
    sortField = 'CreatedDate'; // Fallback to safe default
}
```

---

# 30. Search + Filter + Sort + Pagination

**Complete Server-Side Architecture:**

```text
Search Input (Text)
       ↓
Filter Input (Combobox)
       ↓
Sort Input (Datatable Header)
       ↓
Apex Controller (Validates inputs, builds Dynamic WHERE)
       ↓
COUNT() Query (Executes Dynamic WHERE)
       ↓
LIMIT + OFFSET Query (Executes Dynamic WHERE + ORDER BY)
       ↓
Returns PaginationResult to LWC
```
The server is the single source of truth. Any change in UI state triggers a new consolidated request to Apex.

---

# 31. Dynamic SOQL for Pagination

Dynamic SOQL (`Database.query()`) is mandatory when combinations of Search, Filters, and Sorts are optional or variable.

**Safe Example:**
```java
@AuraEnabled
public static PaginationResult getDynamicClaims(String searchKey, String sortField, String sortDir, Integer pageSize, Integer pageNumber) {
    Integer offst = (pageNumber - 1) * pageSize;
    String searchPattern = String.isNotBlank(searchKey) ? '%' + String.escapeSingleQuotes(searchKey) + '%' : null;
    
    // Secure Whitelisting
    Set<String> validFields = new Set<String>{'Name', 'Status__c', 'CreatedDate'};
    sortField = validFields.contains(sortField) ? sortField : 'CreatedDate';
    sortDir = sortDir == 'asc' ? 'ASC' : 'DESC';

    String baseQuery = 'FROM Warranty_Claim__c ';
    String whereClause = '';
    
    if (searchPattern != null) {
        whereClause = 'WHERE Name LIKE :searchPattern '; // Bind variable used dynamically
    }
    
    // Count Query
    Integer totalCount = Database.countQuery('SELECT COUNT() ' + baseQuery + whereClause);
    
    // Data Query
    String dataQuery = 'SELECT Id, Name, Status__c ' + baseQuery + whereClause + 
                       'ORDER BY ' + sortField + ' ' + sortDir + ' ' +
                       'LIMIT :pageSize OFFSET :offst';
                       
    List<Warranty_Claim__c> records = Database.query(dataQuery);
    
    // Return Wrapper...
}
```

---

# 32. SOQL Injection

*   **What it is:** Malicious users inputting SOQL commands into UI text fields to bypass security or expose data.
*   **Why Dynamic SOQL is dangerous:** `String query = 'SELECT Id FROM Account WHERE Name = \'' + userInput + '\'';` If `userInput` is `test' OR Id != null --`, the query becomes completely open.
*   **Solution 1 (Bind Variables):** Apex resolves variables seamlessly even in string queries. `WHERE Name LIKE :searchPattern` is 100% safe.
*   **Solution 2 (Whitelisting):** For structural elements like `ORDER BY` fields, match input against a hardcoded `Set<String>`.

---

# 33. OFFSET Limitations

`OFFSET` is the easiest way to paginate, but it has strict limits in Salesforce:

1.  **Platform Limit:** `OFFSET` cannot exceed **2,000**. If a user tries to navigate to page 41 (size 50, offset 2000), Salesforce throws an Exception.
2.  **Deep Pagination Problem:** To jump to offset 1,500, the database engine must still retrieve and scan the first 1,500 rows before discarding them and returning the next 50. 
3.  **Scalability:** As you click deeper into pages, database performance degrades linearly. 

**Conclusion:** `OFFSET` is perfect for standard UI datatables where users rarely click past Page 5. It is **unsuitable** for bulk data extraction or traversing hundreds of thousands of records.

---

# 34. Keyset Pagination

Also known as **Cursor-based** or **Seek** pagination. Instead of skipping rows mathematically using `OFFSET`, you remember the exact value of the *last record on the current page*, and start the next query from there.

**Example:**
```sql
SELECT Id, Name, CreatedDate 
FROM Warranty_Claim__c 
WHERE CreatedDate < :lastSeenCreatedDate 
ORDER BY CreatedDate DESC 
LIMIT :pageSize
```
Instead of "Skip 50 records," you say "Give me 50 records older than the last one I just looked at."

---

# 35. OFFSET vs Keyset Pagination

| Feature | OFFSET Pagination | Keyset / Cursor Pagination |
| :--- | :--- | :--- |
| **Query Mechanism** | `OFFSET 500` (Math-based skip). | `WHERE Id > :lastId` (Value-based seek). |
| **Deep Pages** | Fails entirely after record 2,000. | Works infinitely (1,000,000+ records). |
| **Performance** | Slower on deep pages. | Consistent, blazing fast (uses indexes). |
| **Scalability** | Low/Moderate. | Extremely High. |
| **Complexity** | Very simple. | Complex (tracking cursors, handling Prev/Next). |
| **Sorting** | Easy, supports any dynamic field. | Harder, requires indexed sequential fields. |
| **Page Jumping** | Can jump straight to Page 5. | Cannot jump pages; must navigate sequentially. |
| **Best Use Cases** | Standard datatables, filtered views. | Infinite scrolling, huge data volumes. |

---

# 36. Stable Sorting for Keyset Pagination

Keyset pagination requires an absolutely deterministic, unique ordering sequence. 

If sorting by `CreatedDate DESC`, and 10 claims were created in the exact same second, the query boundary becomes blurred. The cursor `CreatedDate < :lastDate` might skip valid records sharing that timestamp.

**Solution: Stable Tie-Breakers**
Always append `Id` to your `ORDER BY` to guarantee uniqueness.

```sql
ORDER BY CreatedDate DESC, Id DESC
```
Your cursor now consists of two variables: `lastDate` and `lastId`.
```sql
WHERE CreatedDate < :lastDate OR (CreatedDate = :lastDate AND Id < :lastId)
```

---

# 37. Keyset Pagination Example

**Scenario:** Infinite scroll Warranty Claims.

**Apex:**
```java
@AuraEnabled
public static List<Warranty_Claim__c> getClaimsKeyset(String lastId, Integer pageSize) {
    String query = 'SELECT Id, Name, Claim_Amount__c FROM Warranty_Claim__c WITH USER_MODE ORDER BY Id ASC LIMIT :pageSize';
    
    if (String.isNotBlank(lastId)) {
        query = 'SELECT Id, Name, Claim_Amount__c FROM Warranty_Claim__c WHERE Id > :lastId WITH USER_MODE ORDER BY Id ASC LIMIT :pageSize';
    }
    
    return Database.query(query);
}
```

**JavaScript:**
```javascript
async handleNext() {
    // Get the ID of the very last record currently rendered
    const lastRecord = this.records[this.records.length - 1];
    const newRecords = await getClaimsKeyset({ 
        lastId: lastRecord.Id, 
        pageSize: 10 
    });
    this.records = newRecords; // Or append for infinite scroll
}
```

---

# 38. StandardSetController

`ApexPages.StandardSetController` was the dominant pagination tool in the Visualforce era. 
*   **Capabilities:** Manages lists up to 10,000 records, natively handles Prev/Next actions, and maintains state.
*   **Modern LWC Context:** It relies on stateful server sessions (View State). Modern LWC architecture is built on stateless `@AuraEnabled` methods. 
*   **Conclusion:** Do not use `StandardSetController` for LWC pagination. Build explicit SOQL LIMIT/OFFSET or Keyset solutions.

---

# 39. Database.QueryLocator

`Database.getQueryLocator(query)` bypasses the 50,000 row SOQL limit and generates a cursor that can process up to 50 million records.

*   **Difference from UI Pagination:** `QueryLocator` is designed exclusively for **Batch Apex** background processing. You cannot serialize a QueryLocator and pass it to an LWC for UI pagination. 
*   **Confusion:** Do not confuse Batch processing chunking with User Interface pagination. 

---

# 40. Pagination and Governor Limits

Server-side pagination helps avoid limits, but does not render you immune to them.
*   **SOQL Rows (50,000):** Paginated `SELECT` only returns 50 rows. Safe. However, `SELECT COUNT()` counts *every matching row*. If 100,000 claims match, `COUNT()` fails.
*   **CPU Time (10s):** Returning fewer records saves LWC rendering and Apex serialization time.
*   **Heap Size (6MB):** Tiny payloads easily stay under the limit.

---

# 41. Query Selectivity

For massive datasets, your queries must be **selective**. The Salesforce Query Optimizer requires selective queries to utilize indexes instead of doing Full Table Scans.
If your base filter is `WHERE Status__c = 'New'` and 95% of claims are 'New', the query is non-selective. Server-side pagination built on non-selective queries will eventually time out.

---

# 42. Indexes and Pagination

Indexes drastically improve server-side pagination performance, especially `COUNT()` and Keyset queries.
*   **Standard Indexes:** Applied automatically to `Id`, `Name`, `CreatedDate`, `OwnerId`, RecordTypes, Lookups.
*   **Custom Indexes:** Applied by marking a field as `External ID` or `Unique`, or by requesting Salesforce Support to index a specific field (e.g., `Claim_Amount__c`).
*   **Pagination Impact:** Sorting and filtering on indexed fields ensures the database engine can skip rows (OFFSET) or seek cursors (Keyset) at maximum speed.

---

# 43. Large Data Volumes (LDV)

Handling data scale:
*   **10,000 records:** Client-side fails. OFFSET is perfect (if filtered below 2000).
*   **100,000 records:** OFFSET fails for deep pages. Keyset pagination strongly recommended. `COUNT()` might start hitting limits unless selective.
*   **1,000,000+ records:** Keyset pagination is mandatory. `COUNT()` dynamic queries will fail; you must cache the total count on a parent record using triggers, or remove the exact "Total Pages" UI entirely, replacing it with an "Infinite Scroll" or simple "Next" button.

---

# 44. Security

Exposing Apex to LWC requires rigorous security implementation.
*   `with sharing`: Class declaration ensures Record-Level Access (Sharing Rules, Territories) is respected.
*   `WITH USER_MODE`: SOQL keyword (modern replacement for `Security.stripInaccessible()`) that enforces Object (CRUD) and Field-Level Security (FLS) directly in the database engine.
*   If a user lacks read access to `Claim_Amount__c`, `WITH USER_MODE` will throw an exception rather than silently returning restricted data.

---

# 45. Loading State

Network latency means Apex calls take time. You must block UI interaction during this window.

```javascript
async loadRecords() {
    this.isLoading = true; // Renders <lightning-spinner>
    try {
        this.records = await getClaims();
    } finally {
        this.isLoading = false; // Removes spinner, regardless of success/fail
    }
}
```

---

# 46. Preventing Duplicate Requests

Rapid clicking on "Next" causes Race Conditions. Request 2 might resolve before Request 1, showing the wrong page.

**Production Approach:**
1.  Show `isLoading` spinner.
2.  Disable pagination buttons `<lightning-button disabled={isLoading}>`.
3.  Add logical gate in JS:
    ```javascript
    handleNext() {
        if (this.isLoading) return;
        // logic...
    }
    ```

---

# 47. Empty State

Handle scenarios where no data exists, or filters yield zero results.

```html
<template if:false={hasRecords}>
    <template if:false={isLoading}>
        <div class="slds-illustration slds-illustration_small">
            <img src={emptyIllustration} alt="No Records" />
            <h3 class="slds-text-heading_medium">No Claims Found</h3>
            <p>Try adjusting your search or filters.</p>
        </div>
    </template>
</template>
```

---

# 48. Error Handling

Errors can stem from FLS violations, SOQL limits, or network drops.

```javascript
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

async loadRecords() {
    try {
        // apex call
    } catch (error) {
        let message = 'Unknown error occurred';
        if (error.body && error.body.message) {
            message = error.body.message;
        }
        this.dispatchEvent(
            new ShowToastEvent({
                title: 'Error loading Claims',
                message: message,
                variant: 'error',
            })
        );
        this.records = []; // Reset UI to safe state
    }
}
```

---

# 49. Pagination After Record Changes

If a user is on Page 5, and deletes the only record on Page 5:
1. Total count decreases.
2. Total pages drops to 4.
3. Current page (5) becomes invalid.

**Resolution Logic (post-delete):**
```javascript
async handleDelete() {
    await deleteRecord(id);
    await this.fetchCount(); // Refreshes totalPages
    if (this.currentPage > this.totalPages && this.totalPages > 0) {
        this.currentPage = this.totalPages; // Move back to valid page
    }
    this.loadRecords();
}
```

---

# 50. Pagination with `lightning-datatable`

`<lightning-datatable>` is a presentational component. It renders whatever array you feed it. Server-side pagination means you swap out the data array on every page turn.

```html
<lightning-datatable
    key-field="Id"
    data={records}
    columns={columns}
    onsort={handleSort}
    sorted-by={sortField}
    sorted-direction={sortDirection}>
</lightning-datatable>

<c-pagination 
    current-page={currentPage} 
    total-pages={totalPages} 
    onpagechange={handlePageChange}>
</c-pagination>
```

---

# 51. Server-Side Sorting with `lightning-datatable`

Clicking a column header fires the `onsort` event.

```javascript
handleSort(event) {
    this.sortField = event.detail.fieldName;
    this.sortDirection = event.detail.sortDirection;
    
    // CRITICAL: Sorting shuffles all records in the DB.
    // You must return the user to Page 1 to see the top results.
    this.currentPage = 1; 
    
    this.loadRecords(); // Sends sortField/Dir to dynamic Apex
}
```

---

# 52. Server-Side Search with `lightning-datatable`

Add a `<lightning-input type="search">` above the datatable. 
Search input changes the dynamic `WHERE` clause in Apex. Again, this completely alters the dataset, so `currentPage` must be reset to 1.

---

# 53. Server-Side Pagination with Debouncing

If a user types "Ford" into search, `onchange` fires 4 times (F, o, r, d). This triggers 4 Apex requests, exhausting server resources and causing race conditions.

**Debouncing** forces the JS to wait until the user stops typing before calling Apex.

```javascript
delayTimeout;

handleSearch(event) {
    const searchString = event.target.value;
    
    // Clear previous timeout
    window.clearTimeout(this.delayTimeout);
    
    // Set new timeout for 300ms
    this.delayTimeout = setTimeout(() => {
        this.searchKey = searchString;
        this.currentPage = 1;
        this.loadRecords();
    }, 300);
}
```

---

# 54. Reusable Pagination Component

Extract pagination UI into a dumb, reusable component (`c-pagination`) to keep your datatable components clean.

**Design:**
*   **Props (`@api`):** `currentPage`, `totalPages`, `totalRecords`.
*   **Events (`CustomEvent`):** Fires `previous`, `next`, or `pagechange`.

**c-pagination.js:**
```javascript
import { LightningElement, api } from 'lwc';

export default class Pagination extends LightningElement {
    @api currentPage;
    @api totalPages;
    @api totalRecords;

    handleNext() {
        this.dispatchEvent(new CustomEvent('pagechange', { detail: this.currentPage + 1 }));
    }
    // ...
}
```

**Parent Component:**
```html
<c-pagination 
    current-page={currentPage} 
    total-pages={totalPages} 
    onpagechange={handlePageChange}>
</c-pagination>
```

---

# 55. Backend Pagination Architecture

Enterprise applications separate concerns:
1.  **LWC (UI):** Renders datatable, handles events, displays errors.
2.  **LWC Controller (JS):** Manages pagination state, invokes Apex.
3.  **Apex Controller:** `@AuraEnabled` endpoints. Validates security (`with sharing`), extracts parameters.
4.  **Service Layer (Apex):** Business logic (e.g., formatting claim statuses).
5.  **Selector Layer (Apex):** Centralized dynamic SOQL generation. Prevents SOQL injection and handles limits/offsets cleanly.

---

# 56. Complete Enterprise Example

**(Conceptual Summary for Warranty Claims)**

*   **UI:** `lightning-datatable` populated by `records` array.
*   **Header:** Search bar (debounced), Status dropdown filter.
*   **Footer:** `c-pagination` component.
*   **JS State:** `searchKey`, `statusFilter`, `sortField`, `sortDir`, `currentPage`, `pageSize`.
*   **Event:** User types in search. Debounce waits 300ms. JS resets `currentPage = 1`. Calls Apex `getPagedClaims`.
*   **Apex Controller:** Receives state wrapper. Calls `ClaimSelector.getPaginated(state)`.
*   **Selector:** Validates `sortField` against whitelist. Appends `WITH USER_MODE`. Executes `COUNT()` query. Executes `SELECT` query with `LIMIT` and `OFFSET`.
*   **Response:** Wrapper returns to JS. Spinner hides. UI updates.

---

# 57. OFFSET-Based vs Keyset-Based Enterprise Example

**Scenario: Warranty Claim List**

**Implementation A: LIMIT + OFFSET**
*   **Apex:** `LIMIT :pageSize OFFSET :offsetVal`
*   **LWC:** Easy page number buttons (1, 2, 3, 4, 5).
*   **Result:** Great UX, fast development. Fails if dealer has >2,000 claims.

**Implementation B: Keyset (Cursor)**
*   **Apex:** `WHERE CreatedDate < :cursorDate OR (CreatedDate = :cursorDate AND Id < :cursorId)`
*   **LWC:** Infinite scroll (Load More button) or Simple Prev/Next. Cannot offer "Jump to Page 5".
*   **Result:** Harder development, slightly restricted UX. Infinite scalability. Excellent performance.

*Selection rule:* Use OFFSET by default. Switch to Keyset only when data volume strictly requires it.

---

# 58. Pagination Decision Guide

```text
Need pagination?
       ↓
How large is dataset?
       ↓
Small (< 500 records) 
       ↓
Client-side pagination is perfectly fine. Fast and easy.
       ↓
Large (> 1,000 records)
       ↓
Server-side pagination is mandatory.
       ↓
Is deep pagination required? (Will user view > 2,000 records?)
       ↓
NO → LIMIT + OFFSET (Standard approach)
       ↓
YES → Keyset / Cursor Pagination (Architectural approach)
```

---

# 59. Common Mistakes

1.  **Loading all records into LWC:** Causes heap errors and browser crashes. *Solution: Backend pagination.*
2.  **Using OFFSET for deep pagination:** Throws exception at 2000. *Solution: Keyset.*
3.  **Missing ORDER BY:** Causes skipping/duplication across pages. *Solution: Always order.*
4.  **Unstable sorting:** Ties in sorting duplicate records. *Solution: Append Id to ORDER BY.*
5.  **SOQL Injection:** Directly injecting search text. *Solution: Use Bind variables.*
6.  **Allowing arbitrary ORDER BY:** Direct injection. *Solution: Whitelist fields.*
7.  **Not resetting page after search/filter:** Stuck on non-existent page 50. *Solution: `currentPage = 1`.*
8.  **Excessive server requests:** Calling apex on every keystroke. *Solution: Debounce.*
9.  **No loading state:** Users click multiple times. *Solution: Spinners and disabled buttons.*
10. **Confusing Batch Apex QueryLocator with UI pagination:** QueryLocator is not for UI.

---

# 60. Debugging Server-Side Pagination

*   **Wrong/Duplicate records on page:** Ensure you have an `ORDER BY`. If you do, ensure it has a unique tie-breaker (like `Id`).
*   **OFFSET Exception:** User navigated past 2,000 records.
*   **Page empty after filter:** You forgot to reset `currentPage` to 1.
*   **Search results wrong:** Ensure your wildcard `%` characters are appended correctly in Apex.
*   **Permission errors:** User lacks FLS on a field queried dynamically. `WITH USER_MODE` correctly caught it.

---

# 61. Performance Optimization

*   **Selective Queries:** Ensure your `WHERE` clauses utilize indexed fields (External IDs, Lookup fields).
*   **Avoid unnecessary COUNT():** If the dataset is 5 million rows, `COUNT()` will fail. Pre-aggregate or remove exact page numbers.
*   **Query only required fields:** Don't `SELECT` rich text fields unless displayed.
*   **Cacheable Apex:** Use `@AuraEnabled(cacheable=true)` for read-only datatables to utilize Lightning Data Service caching.

---

# 62. Best Practices Checklist

*   ✅ Use server-side pagination for large datasets to avoid heap/CPU limits.
*   ✅ Use `LIMIT` and calculate `OFFSET` accurately.
*   ✅ ALWAYS use a deterministic `ORDER BY`.
*   ✅ Reset page to 1 after any search, filter, or sort changes.
*   ✅ Show a loading state and disable navigation buttons during transit.
*   ✅ Use selective SOQL to avoid table scans.
*   ✅ Avoid SOQL injection via bind variables and strict field whitelists.
*   ✅ Respect CRUD/FLS via `WITH USER_MODE`.
*   ✅ Keep pagination logic modular in a reusable UI component.
*   ✅ Use Keyset pagination for infinite scrolling or LDV deep pagination.

---

# 63. Real Project Scenarios (Automotive CRM)

1.  **Warranty Claim List:** Large volume. Moderate depth. Users filter by Dealer. Use **LIMIT/OFFSET**.
2.  **Vehicle Master List:** Massive volume (Millions). Broad searching. Use **Keyset Pagination** or Infinite Scroll.
3.  **Service History (Per Vehicle):** Small volume (< 50 records). Use **Client-side pagination**.
4.  **Claim Line Items:** Small volume per claim. Use **Client-side pagination**.
5.  **Dealer List:** Moderate volume (thousands). Use **LIMIT/OFFSET**.

---

# 64. Interview Questions & Answers

### Beginner Questions
**Q: What is Server-Side Pagination?**
A: Fetching data in small chunks directly from the Salesforce database (using LIMIT and OFFSET), rather than pulling all records to the browser at once.

**Q: Why do we need `ORDER BY` when using `OFFSET`?**
A: Without it, Salesforce does not guarantee the sequence of records. Row order becomes random, breaking the pagination logic.

### Intermediate Questions
**Q: What is a wrapper class in pagination?**
A: A custom Apex class that bundles the List of records and the total record count integer into a single object, requiring only one server call from LWC.

**Q: How do you prevent SOQL injection in dynamic pagination queries?**
A: By using bind variables for search strings, and strictly whitelisting API field names for `ORDER BY` clauses.

### Advanced / Architect Questions
**Q: What are the limitations of OFFSET and how do you overcome them?**
A: OFFSET has a hard platform limit of 2,000 records and degrades performance as it increases. To overcome this for LDV, use Keyset (Cursor-based) pagination, relying on sequential indexed fields like `Id` rather than row skipping.

**Q: Explain stable sorting.**
A: In Keyset pagination, if multiple records share the exact same sorted value (e.g., 10 records created at the exact same second), the cursor fails. Stable sorting appends a guaranteed unique identifier (like `Id`) as a secondary sort parameter to prevent duplicates or omissions.

---

# 65. Revision Summary

*   **Backend Pagination:** Slices data at the database level for performance and memory scaling.
*   **SOQL Tools:** `LIMIT` (chunk size), `OFFSET` (rows to skip), `COUNT()` (total dataset size).
*   **Math:** Offset = `(Page - 1) * Size`.
*   **Apex Pattern:** Wrapper classes bundle data and metadata cleanly.
*   **LWC Pattern:** Imperative Apex with `async/await` is best for complex tables to handle loading states and dynamic parameters.
*   **UX Necessities:** Debounce search inputs, reset to Page 1 on filter changes, implement spinners, handle empty states.
*   **Security:** Prevent SOQL injection on dynamic sorts. Enforce `WITH USER_MODE`.
*   **LDV Scaling:** Shift from `OFFSET` (standard) to Keyset Pagination (cursors) when records exceed platform limits.