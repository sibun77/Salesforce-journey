# Infinite Scroll – Lazy Loading

# 1. Introduction

When building enterprise Salesforce applications, displaying large volumes of data—such as Warranty Claims, Work Orders, Spare Parts, Invoices, Dealers, Customers, Vehicles, Shipments, or Service History—presents a significant challenge. Loading all records at once degrades performance, consumes excessive browser memory, and hits Salesforce governor limits.

**Infinite Scrolling** and **Lazy Loading** are critical design patterns used to solve this. Modern applications use infinite scrolling to provide a seamless user experience, eliminating the friction of traditional clicking through pages (Page 1, Page 2, Page 3). 

In Salesforce Lightning Web Components (LWC), infinite scrolling improves user experience by keeping the UI responsive, reducing initial page load times, and progressively fetching data exactly when the user needs it.

# 2. What is Infinite Scroll?

Infinite Scroll is a user-interface interaction pattern where additional content is loaded continuously as the user approaches the bottom of the currently displayed content. 

Instead of requiring the user to click a "Next Page" button, the application anticipates the user's need for more data and fetches it automatically.

### Flow Diagram

```text
User Scrolls
     ↓
Scroll Threshold Reached
     ↓
LWC Detects Load Request (loadmore event fires)
     ↓
Apex Call (Imperative or Wire)
     ↓
SOQL Retrieves Next Records
     ↓
Records Returned
     ↓
Append to Existing Records (Spread Syntax)
     ↓
User Continues Scrolling
     ↓
Repeat
```

**Step-by-Step Explanation:**
1. **User Scrolls:** The user views the datatable and scrolls down.
2. **Scroll Threshold Reached:** The datatable detects the scroll position is near the bottom.
3. **LWC Detects Load Request:** The `onloadmore` event is triggered.
4. **Apex Call:** LWC invokes an Apex method requesting the next batch.
5. **SOQL Retrieves Next Records:** Apex queries the database using LIMIT and OFFSET/Keyset.
6. **Records Returned:** The new batch of records is sent back to the client.
7. **Append:** The JS controller appends the new records to the existing array.
8. **Repeat:** The cycle continues until no more records exist.

# 3. What is Lazy Loading?

Lazy loading is a data and resource loading strategy where data is loaded only when it is needed, instead of loading everything upfront. 

### Examples

**Initial load:**
Records 1–50 are loaded. The user is at the top of the page.

**User scrolls:**
Records 51–100 are lazy-loaded when the user nears the 50th record.

**User scrolls again:**
Records 101–150 are lazy-loaded.

### Benefits
*   **Reduces Initial Load Time:** Fetching 50 records takes milliseconds; fetching 50,000 takes seconds.
*   **Reduces Response Size:** Smaller JSON payloads over the network.
*   **Saves Browser Memory:** Less data kept in the DOM at initial render.
*   **Reduces Unnecessary Server Processing:** Prevents wasting SOQL rows and CPU time on data the user might never scroll down to see.

# 4. Infinite Scroll vs Lazy Loading

Infinite scroll is primarily a **UI interaction pattern**. Lazy loading is a **data/resource loading strategy**. They are distinct concepts but are often used together to build a seamless experience.

| Feature | Infinite Scroll | Lazy Loading |
| :--- | :--- | :--- |
| **Definition** | UI pattern where data continuously appends as user scrolls. | Strategy of deferring the loading of resources until needed. |
| **Purpose** | Provide a seamless, uninterrupted scrolling experience. | Optimize performance, bandwidth, and initial load times. |
| **Trigger** | User scrolling near the bottom of a container. | Resource entering viewport, component rendering, or user action. |
| **Data loading** | Always incremental/appended. | Can be incremental, or one-time deferred loading (e.g., lazy loading an image). |
| **User experience** | Continuous flow, no pagination clicks. | Faster time-to-interactive. |
| **Server requests** | Multiple sequential requests. | Deferred requests. |
| **Browser memory** | Increases over time as records accumulate. | Kept low initially, scales with loaded resources. |
| **Use cases** | Social feeds, datatables, log viewers, spare parts lists. | Images, modules, hidden tabs, large datasets. |
| **LWC implementation** | `lightning-datatable` with `onloadmore`. | Dynamic imports, conditional rendering (`if:true`), incremental Apex calls. |

# 5. Infinite Scroll vs Traditional Pagination

| Feature | Infinite Scroll | Traditional Pagination |
| :--- | :--- | :--- |
| **Navigation** | Scroll → Load More → Scroll | Page 1 → Page 2 → Page 3 |
| **User experience** | Seamless, continuous, immersive. | Explicit, structured, interruptive. |
| **Page buttons** | None (or occasional fallback "Load More"). | First, Previous, Next, Last, Numbered buttons. |
| **Server requests** | Triggered by scroll events. | Triggered by button clicks. |
| **Data loading** | Appends to existing data (list grows). | Replaces existing data (list size stays constant). |
| **Browser memory** | High (grows continuously). | Low (only holds current page). |
| **Large datasets** | Can cause UI lag if DOM gets too large. | Highly scalable, predictable memory usage. |
| **Complexity** | Higher (requires cursor/scroll management). | Lower (simple offset math). |
| **Mobile experience** | Excellent (natural touch interaction). | Poor (tiny buttons are hard to tap). |
| **Best use cases** | Mobile apps, feeds, discovery-based browsing. | Enterprise tables requiring precise record location, editing, or deep linking. |

# 6. Client-Side vs Server-Side Infinite Scroll

### Client-Side Infinite Scroll
Load all (or a massive chunk) of the dataset from the server first.
↓
Store it in a JavaScript array.
↓
Display progressively in the UI as the user scrolls.

### Server-Side Infinite Scroll
Load only a small portion (e.g., 50 records).
↓
User scrolls.
↓
Request the next portion from the server.
↓
Append records.

| Feature | Client-Side | Server-Side |
| :--- | :--- | :--- |
| **Data Retrieval** | All at once. | In batches as needed. |
| **Initial Load** | Slow. | Fast. |
| **Apex Limits** | High risk of hitting 50,000 SOQL row limit. | Low risk (controlled by LIMIT). |
| **Search/Filter** | Extremely fast (done in JS). | Requires server round-trip. |
| **Recommendation** | Only for small datasets (< 1,000 records). | **Recommended for large Salesforce datasets.** |

# 7. How Infinite Scroll Works in LWC

The lifecycle of infinite scroll in a Lightning Web Component follows this pattern:

1.  **Initial Load:** Component connects to DOM. `connectedCallback` or wire service fires, requesting batch 1.
2.  **Display Records:** Records render in `lightning-datatable`.
3.  **User Scrolls:** User drags the scrollbar down.
4.  **loadmore Event:** Datatable fires the `onloadmore` event when the threshold is hit.
5.  **Check isLoading:** JS checks if a request is already in progress to avoid duplicates.
6.  **Check hasMoreRecords:** JS verifies there is actually more data on the server.
7.  **Call Apex:** JS requests Batch 2 via imperative Apex.
8.  **Append Records:** New records arrive and are spread into the existing array.
9.  **Stop Loading:** `isLoading` flag resets, datatable spinner stops.

# 8. lightning-datatable Infinite Loading

LWC provides built-in support for infinite scrolling via `lightning-datatable`.

```html
<lightning-datatable
    key-field="Id"
    data={records}
    columns={columns}
    enable-infinite-loading={hasMoreRecords}
    onloadmore={loadMoreData}
    load-more-offset="20">
</lightning-datatable>
```

**Attributes Explained:**
*   `enable-infinite-loading`: A boolean. When `true`, enables the scroll listener. Bind this to your `hasMoreRecords` variable so it turns off when all data is loaded.
*   `onloadmore`: The event handler triggered when the scroll reaches the offset threshold.
*   `load-more-offset`: The distance (in pixels) from the bottom of the table that triggers the `onloadmore` event.

# 9. onloadmore Event

The `onloadmore` event is central to the infinite scrolling architecture.

*   **When it fires:** When the user scrolls within `load-more-offset` pixels of the datatable's bottom.
*   **Why it fires:** To signal the JavaScript controller to fetch the next batch.
*   **How to handle it:** Pass the event to a handler, extract the target (to stop the spinner later), and call Apex.
*   **Preventing duplicate requests:** Always check an `isLoading` boolean before proceeding.
*   **Stopping infinite loading:** Set `target.isLoading = false` when done.

```javascript
handleLoadMore(event) {
    const { target } = event;
    
    // Prevent duplicate requests
    if (this.isLoading || !this.hasMoreRecords) {
        target.isLoading = false;
        return;
    }

    target.isLoading = true; // Show table spinner
    this.isLoading = true;   // Set JS lock
    
    this.fetchMoreData(target);
}
```

# 10. load-more-offset

The `load-more-offset` determines how aggressively the component prefetches data.

```html
load-more-offset="20"
```

*   **Meaning:** When the user's scrollbar is 20 pixels away from the bottom, fire `onloadmore`.
*   **How it affects UX:** A larger offset (e.g., `100`) fetches data sooner, making the experience smoother for the user, as the data might load before they even hit the absolute bottom. A smaller offset (e.g., `5`) waits until they explicitly hit the bottom.
*   **Choosing a reasonable offset:** `20` to `50` is standard. Too high, and you might accidentally load multiple pages just by scrolling quickly.

# 11. Basic Infinite Scroll State

Managing state accurately is critical.

```javascript
records = [];          // Holds the cumulative list of records to display
pageSize = 50;         // How many records to fetch per Apex call
currentPage = 1;       // Tracks which page we are currently on (for OFFSET)
isLoading = false;     // Locks the system to prevent concurrent/duplicate Apex calls
hasMoreRecords = true; // Boolean flag; turns false when no more data exists on the server
totalRecords = 0;      // Optional: stores the absolute total count in the database
```

# 12. Basic Infinite Scroll Example

**Scenario:** Loading Warranty Claims.

**Apex (WarrantyClaimController.cls):**
```java
public with sharing class WarrantyClaimController {
    @AuraEnabled(cacheable=true)
    public static List<Warranty_Claim__c> getClaims(Integer pageSize, Integer pageNumber) {
        Integer offsetValue = (pageNumber - 1) * pageSize;
        return [
            SELECT Id, Name, Status__c, CreatedDate
            FROM Warranty_Claim__c
            WITH USER_MODE
            ORDER BY CreatedDate DESC
            LIMIT :pageSize OFFSET :offsetValue
        ];
    }
}
```

**JavaScript (warrantyClaimList.js):**
```javascript
import { LightningElement } from 'lwc';
import getClaims from '@salesforce/apex/WarrantyClaimController.getClaims';

const COLUMNS = [
    { label: 'Claim Name', fieldName: 'Name' },
    { label: 'Status', fieldName: 'Status__c' }
];

export default class WarrantyClaimList extends LightningElement {
    records = [];
    columns = COLUMNS;
    pageSize = 50;
    currentPage = 1;
    isLoading = false;
    hasMoreRecords = true;

    connectedCallback() {
        this.loadInitialData();
    }

    async loadInitialData() {
        this.isLoading = true;
        try {
            const data = await getClaims({ pageSize: this.pageSize, pageNumber: this.currentPage });
            this.records = data;
        } catch (error) {
            console.error(error);
        } finally {
            this.isLoading = false;
        }
    }

    async loadMoreData(event) {
        const { target } = event;
        if (this.isLoading || !this.hasMoreRecords) {
            if(target) target.isLoading = false;
            return;
        }

        this.isLoading = true;
        if(target) target.isLoading = true;
        this.currentPage++;

        try {
            const data = await getClaims({ pageSize: this.pageSize, pageNumber: this.currentPage });
            if (data.length < this.pageSize) {
                this.hasMoreRecords = false; // Reached the end
            }
            this.records = [...this.records, ...data]; // Append
        } catch (error) {
            console.error(error);
            this.currentPage--; // Revert page count on failure
        } finally {
            this.isLoading = false;
            if(target) target.isLoading = false;
        }
    }
}
```

**HTML (warrantyClaimList.html):**
```html
<template>
    <div style="height: 400px;">
        <lightning-datatable
            key-field="Id"
            data={records}
            columns={columns}
            enable-infinite-loading={hasMoreRecords}
            onloadmore={loadMoreData}
            load-more-offset="20">
        </lightning-datatable>
    </div>
</template>
```

# 13. Apex for Infinite Scroll

```java
@AuraEnabled(cacheable=true)
public static List<Warranty_Claim__c> getClaims(Integer pageSize, Integer pageNumber) {
    Integer offsetValue = (pageNumber - 1) * pageSize;
    return [
        SELECT Id, Name, Status__c, CreatedDate 
        FROM Warranty_Claim__c 
        ORDER BY CreatedDate DESC 
        LIMIT :pageSize 
        OFFSET :offsetValue
    ];
}
```
*   `pageSize`: Defines the batch limit.
*   `pageNumber`: The current increment.
*   `OFFSET`: Tells SOQL how many records to skip before returning the next batch.
*   `LIMIT`: Restricts the returned set size.
*   `ORDER BY`: **Mandatory**. Without sorting, OFFSET yields unpredictable results.

# 14. Appending Records

**Replacing:**
```javascript
this.records = data; // Destroys previous records
```
**Appending (Spread Syntax):**
```javascript
this.records = [...this.records, ...data]; // Keeps old, adds new
```
Infinite scroll *requires* appending. The datatable expects the full, cumulative array of everything loaded so far to maintain the scroll position and DOM structure. Replacing the array acts like traditional pagination, wiping out the scroll context.

# 15. Tracking Current Page

The `currentPage` state tracks progress to calculate the server-side OFFSET.

```text
Page 1 → Load 50 → currentPage = 2 → Load next 50 → currentPage = 3
```

**Preventing issues:**
*   **Duplicate pages:** Using an `isLoading` lock ensures you don't send two `page=2` requests simultaneously.
*   **Skipped pages:** Increment `currentPage` right before the call. If the call fails, decrement it (`this.currentPage--`) so the user can retry.

# 16. Preventing Duplicate Requests

Duplicate requests occur when a user scrolls rapidly, firing `onloadmore` multiple times before the first Apex transaction completes.

```javascript
if (this.isLoading || !this.hasMoreRecords) {
    return; // Exit early
}
```
*   `this.isLoading`: Acts as a mutex/lock. Sets to `true` when request starts, `false` when it ends.
*   `!this.hasMoreRecords`: Prevents calling Apex if we already know there's no more data in the database.

# 17. hasMoreRecords

`hasMoreRecords` instructs the UI to stop attempting to load data.

**Basic detection:**
```javascript
if (data.length < this.pageSize) {
    this.hasMoreRecords = false;
}
```
*Limitation:* If `pageSize` is 50, and exactly 100 records exist in the database, Batch 2 returns 50. The UI thinks there is more data, attempts Batch 3, gets 0 records, and *then* sets `hasMoreRecords = false`. This wastes one Apex call.

# 18. Total Record Count

Using `COUNT()` solves the limitation above.

```java
SELECT COUNT() FROM Warranty_Claim__c
```

By knowing `totalRecords` upfront, LWC knows exactly when to stop without the "empty final call".
```javascript
hasMoreRecords = this.records.length < this.totalRecords;
```

# 19. Returning Records and Metadata Together

To provide data and count in one transaction, use a wrapper class:

```java
public class InfiniteScrollResult {
    @AuraEnabled public List<Warranty_Claim__c> records;
    @AuraEnabled public Integer totalRecords;
    @AuraEnabled public Boolean hasMoreRecords;
}
```
This simplifies LWC logic. The server calculates whether more records exist based on the count, taking the burden off the client side.

# 20. Infinite Scroll with Imperative Apex

Imperative Apex with `async/await` is the preferred approach for infinite scroll because it provides precise control over *when* the call happens and how the data is appended.

```javascript
async loadMoreData() {
    if (this.isLoading || !this.hasMoreRecords) return;
    this.isLoading = true;

    try {
        const data = await getClaims({ 
            pageNumber: this.currentPage + 1, 
            pageSize: this.pageSize 
        });
        
        this.records = [...this.records, ...data];
        this.currentPage++;
    } catch (error) {
        this.showToast('Error', error.body.message, 'error');
    } finally {
        this.isLoading = false;
    }
}
```

# 21. Infinite Scroll with Wire Service

While possible, using the `@wire` service for infinite loading is overly complex. You have to make `pageNumber` a reactive property (`'$currentPage'`), let the wire execute, detect the change in the wired function, and manually append to a tracked property. 

*Limitation:* Wires are meant to be declarative. Forcing them into an imperative, incremental append pattern often results in duplicated data on refresh and complex `refreshApex` handling.

# 22. Wire vs Imperative Apex

| Feature | Wire Service | Imperative Apex |
| :--- | :--- | :--- |
| **Initial load** | Automatic on component load. | Explicitly called in `connectedCallback`. |
| **Load-more requests** | Awkward (requires changing reactive params). | Natural (call method on scroll event). |
| **State management** | Tricky (wire caches responses). | Straightforward (you control the array). |
| **Caching** | Excellent (LDS Cache). | Requires manual cache bypass or `cacheable=false`. |
| **Best use cases** | Static datasets, forms, standard read. | **Infinite scrolling**, dynamic lists, complex operations. |

# 23. OFFSET-Based Infinite Scroll

Using LIMIT and OFFSET is the standard way to paginate.

*   **Page 1:** `LIMIT 50 OFFSET 0` (Records 1 - 50)
*   **Page 2:** `LIMIT 50 OFFSET 50` (Records 51 - 100)
*   **Page 3:** `LIMIT 50 OFFSET 100` (Records 101 - 150)

# 24. OFFSET Limitations

**Crucial Salesforce Limits:**
*   **Maximum OFFSET is 2,000.** If a user scrolls past 2,000 records, the SOQL query will throw an exception.
*   **Performance:** OFFSET forces the database to scan and discard the skipped rows. `OFFSET 1500` means the DB reads 1,550 rows to return 50. This degrades performance as the user scrolls deeper.
*   **Scalability:** Unsuitable for Large Data Volumes (LDV). 

# 25. Keyset Pagination for Infinite Scroll

Also known as Cursor Pagination or Seek Pagination, this method abandons OFFSET entirely. Instead, it remembers the exact values of the last record loaded and queries *from that point forward*.

```java
SELECT Id, Name, CreatedDate 
FROM Warranty_Claim__c 
WHERE CreatedDate < :lastCreatedDate 
ORDER BY CreatedDate DESC 
LIMIT :pageSize
```

*   **Initial request:** Get top 50.
*   **Store last record cursor:** Remember the `CreatedDate` of the 50th record.
*   **Request next records:** Pass the cursor to Apex.
*   **WHERE condition:** DB seeks directly to that index and pulls the next 50.

# 26. OFFSET vs Keyset for Infinite Scroll

| Feature | OFFSET Pagination | Keyset (Cursor) Pagination |
| :--- | :--- | :--- |
| **Implementation** | Easy (just increment page numbers). | Harder (requires passing specific field values). |
| **Performance** | Slows down as depth increases. | Consistently fast, regardless of depth. |
| **Deep scrolling** | **Fails after 2,000 records.** | Can scroll millions of records. |
| **Stable ordering** | Can duplicate/skip if data is added during scroll. | Highly stable. |
| **Best use cases** | Small lists, quick implementations. | **Enterprise Large Data Volumes (LDV).** |

# 27. Stable Sorting for Keyset Pagination

```java
ORDER BY CreatedDate DESC, Id DESC
```

Deterministic sorting is non-negotiable for Keyset pagination. If multiple Warranty Claims have the exact same `CreatedDate`, the database needs a tie-breaker, otherwise records may be duplicated or skipped across batches. The record `Id` acts as a unique, stable tie-breaker.

# 28. Complete Keyset Infinite Scroll Example

**Scenario:** Warranty Claims (Keyset Pagination)

**Apex:**
```java
public class WarrantyClaimController {
    @AuraEnabled
    public static List<Warranty_Claim__c> getClaimsKeyset(String lastDateStr, Id lastId, Integer pageSize) {
        if (String.isBlank(lastDateStr)) {
            return [SELECT Id, Name, CreatedDate FROM Warranty_Claim__c 
                    ORDER BY CreatedDate DESC, Id DESC LIMIT :pageSize];
        }
        
        DateTime lastDate = (DateTime)JSON.deserialize('"' + lastDateStr + '"', DateTime.class);
        
        return [SELECT Id, Name, CreatedDate FROM Warranty_Claim__c 
                WHERE CreatedDate < :lastDate 
                   OR (CreatedDate = :lastDate AND Id < :lastId)
                ORDER BY CreatedDate DESC, Id DESC LIMIT :pageSize];
    }
}
```

**JavaScript:**
```javascript
export default class WarrantyKeyset extends LightningElement {
    records = [];
    pageSize = 50;
    isLoading = false;
    hasMoreRecords = true;
    lastDate = null;
    lastId = null;

    async loadData(event) {
        if(this.isLoading || !this.hasMoreRecords) return;
        this.isLoading = true;

        try {
            const data = await getClaimsKeyset({
                lastDateStr: this.lastDate,
                lastId: this.lastId,
                pageSize: this.pageSize
            });

            if(data.length < this.pageSize) this.hasMoreRecords = false;

            if(data.length > 0) {
                this.records = [...this.records, ...data];
                const lastRecord = data[data.length - 1];
                this.lastDate = lastRecord.CreatedDate;
                this.lastId = lastRecord.Id;
            }
        } catch (error) {
            console.error(error);
        } finally {
            this.isLoading = false;
            if(event && event.target) event.target.isLoading = false;
        }
    }
}
```

# 29. Search + Infinite Scroll

When implementing Search over Infinite Scroll, applying a new search term means the entire context changes.

**Flow:**
Search Input changes → Debounce → Reset records array to `[]` → Reset cursor/page → Set `hasMoreRecords = true` → Call Apex → Load First Batch.

**Important:** The previous cursor (e.g., `lastId`) must NOT be reused. It belongs to the old dataset context.

# 30. Filtering + Infinite Scroll

Filtering by Status, Dealer, or Date follows the exact same reset rules as Search.

**Flow:**
Filter applied → Reset Pagination & Cursors → Load First Batch → Scroll → Load More (passing the filter to Apex every time).

Filters must be passed as variables into the Apex method so they are included in every subsequent `WHERE` clause.

# 31. Sorting + Infinite Scroll

Server-side sorting must be handled securely via Dynamic SOQL.

```java
String query = 'SELECT Id, Name FROM Warranty_Claim__c ORDER BY ' + sortField + ' ' + sortDirection;
```

When a user clicks a column header to sort:
1.  Reset the current dataset and cursors.
2.  Update the `sortField` and `sortDirection` variables.
3.  Request Batch 1 with the new sort parameters.

# 32. Search + Filter + Sort + Infinite Scroll

**Architecture:**
```text
Search (Keyword) ┐
Filter (Status)  ┼→ Pagination State (Reset Cursor) → Apex Request → Database
Sort (Field/Dir) ┘
```
The client manages the state of Search, Filter, and Sort. When *any* of them change, the UI resets. When scrolling, *all* of them are passed alongside the pagination variables to ensure the next batch matches the user's view.

# 33. Debouncing Search

If a user searches for "Warranty":
W → Wa → War → Warr → Warranty
Without debouncing, LWC fires 5 Apex calls instantly, risking limit exceptions and out-of-order responses.

**Implementation:**
```javascript
searchTimeout;
handleSearch(event) {
    const searchTerm = event.target.value;
    clearTimeout(this.searchTimeout);
    
    this.searchTimeout = setTimeout(() => {
        this.resetPagination();
        this.searchTerm = searchTerm;
        this.loadData();
    }, 300); // Wait 300ms after user stops typing
}
```

# 34. Refreshing Infinite Scroll Data

Sometimes the loaded list needs a full refresh (e.g., via a refresh button or navigating back).
To refresh imperative Apex, you must reset state manually:

```javascript
refreshList() {
    this.records = [];
    this.currentPage = 1; // Or reset cursor variables
    this.hasMoreRecords = true;
    this.loadData();
}
```

# 35. Record Deletion

If a user deletes a record from the datatable:
1.  Call Apex to delete the record in Salesforce.
2.  On success, `this.records = this.records.filter(rec => rec.Id !== deletedId);`.
3.  Decrement total count if tracking it.
*Note:* A full refresh is safer if the deletion heavily affects sorting or filtering logic.

# 36. Record Creation

If a new Warranty Claim is created, simply appending it to `this.records` via JS can violate the current sorting order (e.g., if sorted alphabetically, a new 'Z' claim shouldn't just be slapped at the top).

**Best Practice:** On creation, trigger a full reset and reload of the first batch to guarantee data integrity and sorting rules.

# 37. Loading State

The `isLoading` flag ensures users understand the system is working.

*   `lightning-datatable` has a built-in spinner tied to `onloadmore` (`event.target.isLoading = true`).
*   For the initial load, use a standard `<lightning-spinner if:true={isLoading}>`.

# 38. Error Handling

Errors can stem from FLS, governor limits, or network drops.

**Handling logic:**
If fetching Batch 3 fails, show a toast error but *do not reset the table*. Ensure `isLoading` resets to `false` and `currentPage` is decremented. This allows the user to trigger the scroll again to retry fetching Batch 3.

# 39. Empty State

Differentiate between scenarios:
*   **No initial records:** "No Warranty Claims exist in the system." (Show an empty illustration).
*   **No search results:** "No claims found matching 'ABC'."
*   **All records loaded:** (End of data) No message needed, the scroll spinner simply stops appearing.

# 40. End-of-Data Handling

How to know we are done:
1.  **Returned records < pageSize:** Simplest. If you ask for 50 and get 49, you are at the end. (Drawback: useless final call if exact multiple of 50).
2.  **Total count comparison:** Best UX, but requires an extra `COUNT()` query initially.
3.  **hasMoreRecords boolean:** Server returns metadata. Balances logic nicely.
4.  **Keyset returning 0:** Used in cursor pagination when the seek hits the end.

# 41. Browser Memory

Infinite scroll keeps appending to a JS array. 
`records = [...records, ...data];`

*   **100 records:** Trivial.
*   **10,000 records:** The DOM becomes massive. Rendering thousands of rows in a datatable crashes mobile browsers and lags desktops. 
Infinite scrolling solves *backend* limits, but shifts the burden to *frontend* memory if not managed alongside Virtualization.

# 42. Virtualization vs Infinite Scroll

| Feature | Infinite Scroll | List Virtualization |
| :--- | :--- | :--- |
| **Problem solved** | Fetching data from the server. | Rendering DOM nodes in the browser. |
| **Mechanism** | Appends data continuously. | Only renders the 20 rows currently visible on screen, recycling DOM elements. |
| **Relationship** | Data fetching strategy. | UI rendering strategy. |

`lightning-datatable` actually implements basic virtualization under the hood to prevent DOM collapse, meaning you can safely infinite scroll a few thousand records.

# 43. Performance Optimization

*   **Batch size:** Use 50-100. Too small = too many Apex calls. Too large = slow SOQL/payloads.
*   **Query specific fields:** `SELECT Id, Name` instead of `SELECT fields(ALL)`.
*   **Minimize payload:** Don't return heavily nested related records unless displayed.
*   **Stable sorting:** Avoids DB churn.

# 44. SOQL Optimization

Pagination does not automatically make SOQL fast. 
If your query is `SELECT Id FROM Warranty_Claim__c WHERE Status__c = 'Open'`, and you have 5 million closed claims, the database still performs a massive scan to find the open ones.

Ensure WHERE clauses hit indexed fields. Avoid leading wildcards (`LIKE '%term'`) which bypass indexes.

# 45. Governor Limits

Infinite scroll spreads data retrieval across multiple Apex transactions.
*   **SOQL Rows (50,000):** Eliminated as a risk per transaction.
*   **CPU Time (10s):** Significantly reduced.
*   **Heap Size (6MB):** Payload kept small.

*However*, each scroll fires a new Apex transaction, counting against concurrent execution limits and total API call limits if exposed externally.

# 46. Security

Always enforce sharing and FLS on server-side infinite loading.
*   Use `WITH USER_MODE` or `Security.stripInaccessible()`.
*   Enforce `with sharing` on the controller.
*   Pagination must never bypass sharing rules to reveal records the user shouldn't see.

# 47. Dynamic SOQL Security

When combining Sort/Filter with Infinite Scroll, Dynamic SOQL is common.
**Unsafe (SOQL Injection):**
```java
String query = 'SELECT Id FROM Claim ORDER BY ' + sortField; // Vulnerable
```
**Safe (Whitelisting):**
```java
Set<String> validFields = new Set<String>{'Name', 'CreatedDate'};
if(validFields.contains(sortField)) {
    String query = 'SELECT Id FROM Claim ORDER BY ' + String.escapeSingleQuotes(sortField);
}
```

# 48. Reusable Infinite Scroll Component

Instead of rewriting scroll logic, wrap it.

```html
<c-infinite-scroll 
    records={records} 
    has-more={hasMoreRecords} 
    loading={isLoading} 
    onloadmore={handleLoadMore}>
</c-infinite-scroll>
```
The child component contains the scroll event listener and spinner UI. It dispatches a custom `loadmore` event to the parent, which handles the business logic (Apex calls).

# 49. Infinite Scroll Component Architecture

```text
Parent LWC (WarrantyManager)
     ↓ passes data down
Reusable Infinite Scroll UI Component
     ↓ detects scroll, fires custom event
Parent (WarrantyManager)
     ↓ invokes logic
Apex (Controller)
     ↓ 
Database
```
This abstracts the UI behavior (scrolling, spinners) away from data retrieval (SOQL, cursors).

# 50. Complete Enterprise Example (Keyset Pagination)

**Scenario:** Automotive CRM Warranty Claim Management.

**Apex:**
```java
public with sharing class WarrantyService {
    @AuraEnabled
    public static List<Warranty_Claim__c> getClaims(String statusFilter, String lastDateStr, Id lastId, Integer pageSize) {
        String query = 'SELECT Id, Name, Status__c, CreatedDate, Dealer__r.Name FROM Warranty_Claim__c ';
        List<String> conditions = new List<String>();
        
        if (String.isNotBlank(statusFilter)) {
            conditions.add('Status__c = :statusFilter');
        }
        
        if (String.isNotBlank(lastDateStr)) {
            DateTime lastDate = (DateTime)JSON.deserialize('"' + lastDateStr + '"', DateTime.class);
            conditions.add('(CreatedDate < :lastDate OR (CreatedDate = :lastDate AND Id < :lastId))');
        }
        
        if (!conditions.isEmpty()) query += ' WHERE ' + String.join(conditions, ' AND ');
        
        query += ' WITH USER_MODE ORDER BY CreatedDate DESC, Id DESC LIMIT :pageSize';
        return Database.query(query);
    }
}
```

**JavaScript:**
```javascript
import { LightningElement, track } from 'lwc';
import getClaims from '@salesforce/apex/WarrantyService.getClaims';

export default class WarrantyManager extends LightningElement {
    @track records = [];
    statusFilter = '';
    pageSize = 50;
    lastDate = null;
    lastId = null;
    hasMoreRecords = true;
    isLoading = false;

    connectedCallback() { this.loadData(); }

    handleStatusChange(event) {
        this.statusFilter = event.detail.value;
        this.resetState();
        this.loadData();
    }

    resetState() {
        this.records = [];
        this.lastDate = null;
        this.lastId = null;
        this.hasMoreRecords = true;
    }

    async loadData(event) {
        if (this.isLoading || !this.hasMoreRecords) return;
        this.isLoading = true;

        try {
            const result = await getClaims({
                statusFilter: this.statusFilter,
                lastDateStr: this.lastDate,
                lastId: this.lastId,
                pageSize: this.pageSize
            });

            if (result.length < this.pageSize) this.hasMoreRecords = false;

            if (result.length > 0) {
                this.records = [...this.records, ...result];
                const lastRec = result[result.length - 1];
                this.lastDate = lastRec.CreatedDate;
                this.lastId = lastRec.Id;
            }
        } catch (error) {
            console.error('Error fetching claims', error);
        } finally {
            this.isLoading = false;
            if(event) event.target.isLoading = false;
        }
    }
}
```

**HTML:**
```html
<template>
    <lightning-combobox label="Status" options={statusOptions} onchange={handleStatusChange}></lightning-combobox>
    
    <div style="height: 500px">
        <lightning-datatable
            key-field="Id"
            data={records}
            columns={columns}
            enable-infinite-loading={hasMoreRecords}
            onloadmore={loadData}>
        </lightning-datatable>
    </div>
</template>
```

# 51. OFFSET-Based Enterprise Example

If rewriting the above with OFFSET, you replace the cursor (`lastDate`, `lastId`) with `currentPage`:

```javascript
// JS state
currentPage = 1;

// Apex Call
getClaims({ statusFilter: this.statusFilter, pageNumber: this.currentPage, pageSize: this.pageSize })

// Apex Query
query += ' LIMIT :pageSize OFFSET ' + ((pageNumber - 1) * pageSize);
```
**Limitations of this architecture:**
If you have 10,000 warranty claims, scrolling to record 2050 will crash because OFFSET exceeds the 2,000 limit.

# 52. Keyset-Based Enterprise Example

As demonstrated in Section 50, Keyset pagination uses the actual data values (`CreatedDate` + `Id`) as a cursor. 

**Why it scales better:**
The SOQL engine uses database indexes to jump directly to the specific `CreatedDate` and retrieve the next 50 rows. It never scans the previous records. You can scroll through millions of records with constant, O(1) performance.

# 53. Infinite Scroll vs Backend Pagination

Both use the same backend mechanisms (LIMIT/OFFSET or Keyset). The difference is entirely UI-based.
*   **Traditional Server-Side Pagination:** Fetches Batch 2 and *replaces* Batch 1 in the UI.
*   **Infinite Scroll:** Fetches Batch 2 and *appends* it to Batch 1 in the UI.

# 54. Infinite Scroll vs Frontend Pagination

*   **Frontend Pagination:** Pulls all 1,000 records from Apex instantly, stores in JS, and displays 50 at a time via JS slicing.
*   **Infinite Scroll:** Pulls 50 records. Asks Apex for the next 50 only when needed.

# 55. Infinite Scroll vs Lazy Loading vs Pagination

| Strategy | Location | Loading Mechanism | UI Behavior |
| :--- | :--- | :--- | :--- |
| **Infinite Scroll** | Server-Side | Incremental | Appends continuously |
| **Lazy Loading** | Client or Server | Deferred | Loads resources just-in-time |
| **Client-Side Pagination** | Client-Side | All at once | Navigates discrete chunks in JS |
| **Server-Side Pagination** | Server-Side | Incremental | Replaces view with next discrete chunk |
| **Keyset Pagination** | Server-Side | Incremental | Backend cursor logic (can feed IS or Paging) |

# 56. Accessibility

Infinite scrolling creates massive accessibility (a11y) challenges for screen readers and keyboard users.
*   **Keyboard navigation:** A user tabbing through links gets trapped because the page keeps growing before they reach the footer.
*   **Screen readers:** Cannot easily determine the size of the list.
*   **Alternative:** Always offer a fallback explicit "Load More" button for users who cannot easily scroll, or use standard pagination if strict accessibility compliance is required.

# 57. Mobile Considerations

*   **Touch scrolling:** Natural and intuitive on mobile.
*   **Mobile memory:** Severely constrained compared to desktop. `lightning-datatable` virtualization is critical here.
*   **Batch size:** Lower batch sizes (e.g., 20-30) are better for mobile networks to prevent UI stalling.

# 58. Common Mistakes

1.  **Loading all records at once:** *Problem:* Hits limits. *Solution:* Server-side increments.
2.  **Using client-side pagination for LDV:** *Problem:* Browser crashes. *Solution:* Keyset pagination.
3.  **Replacing records instead of appending:** *Problem:* Scrolling breaks. *Solution:* Use `...` spread syntax.
4.  **Not tracking current page/cursor:** *Problem:* Getting the same 50 records repeatedly. *Solution:* Increment state.
5.  **Duplicate Apex calls:** *Problem:* Wasted limits. *Solution:* Use `isLoading` lock.
6.  **Not stopping when data ends:** *Problem:* Wasted empty calls. *Solution:* `hasMoreRecords`.
7.  **OFFSET deep pagination:** *Problem:* Fails > 2000. *Solution:* Keyset pagination.
8.  **Unstable sorting:** *Problem:* Missing records. *Solution:* Append `Id DESC` to `ORDER BY`.
9.  **Not resetting cursor on Search/Filter/Sort:** *Problem:* Zero results or errors. *Solution:* Reset state variables.
10. **Ignoring Indexes:** *Problem:* Slow SOQL. *Solution:* Filter on indexed fields.

# 59. Debugging Infinite Scroll

**Checklist:**
*   *onloadmore not firing:* Check `height` on the datatable container. If it's not restricted, the datatable expands infinitely and no scrollbar appears.
*   *Duplicate records:* Verify `isLoading` lock is functioning.
*   *Records replacing:* Check your JS assignment. Must be `this.records = [...this.records, ...result]`.
*   *Spinner never stops:* Ensure `target.isLoading = false` is called in a `finally` block.
*   *OFFSET errors:* You crossed the 2,000 threshold. Switch to Keyset.

# 60. Performance Testing

Test with distinct volumes:
*   **100 records:** Verify basic logic.
*   **10,000 records:** Verify `OFFSET` limits or Keyset logic. Monitor DOM node counts.
*   **100,000+ records:** Verify SOQL query time using the Query Plan Tool.

Monitor network payload in Chrome DevTools to ensure you aren't retrieving unused fields.

# 61. Real Project Scenarios (Automotive CRM)

1.  **Warranty Claim List:** Infinite scroll is ideal for adjusters scanning incoming claims. Keyset pagination required due to high volume. Filter by Status = 'Pending'.
2.  **Spare Parts Catalog:** High volume, heavily searched. Debounced search + Keyset pagination.
3.  **Vehicle Service History:** Moderate volume (usually < 100 per vehicle). LIMIT + OFFSET is perfectly acceptable here.
4.  **Dealer List:** Moderate volume. Client-side caching + infinite scroll can work if < 2000 dealers.

# 62. Decision Guide

```text
Need to display many records?
        ↓
Need traditional page navigation? (Direct links to Page 5)
        ↓
YES → Server-Side Pagination
        ↓
NO
 ↓
Would continuous scrolling improve UX?
 ↓
YES → Infinite Scroll
 ↓
How large is dataset?
 ↓
Moderate (< 2,000) → LIMIT + OFFSET suitable
 ↓
Large / Deep Scroll → Keyset / Cursor Pagination
```

# 63. Best Practices Checklist

*    **Load data incrementally:** Never pull 50k rows.
*    **Use server-side loading for large datasets:** Apex handles the heavy lifting.
*    **Use reasonable batch sizes:** 50 records is standard.
*    **Append new records:** `...this.records`.
*    **Track loading state:** `isLoading` prevents duplicates.
*    **Track cursor/page:** Critical for the next batch.
*    **Stop when no more records exist:** Toggle `hasMoreRecords`.
*    **Use stable sorting:** Always tie-break with `Id`.
*    **Reset pagination on UI changes:** Search, Filter, Sort require a fresh start.
*    **Debounce search:** Protects Apex limits.
*    **Query only required fields:** Keeps payload light.
*    **Use keyset pagination for deep scrolling:** Avoids 2,000 OFFSET limit.
*    **Handle empty states:** Inform the user cleanly.
*    **Respect CRUD/FLS:** `WITH USER_MODE`.
*    **Maintain accessibility:** Fallback "Load More" options.

# 64. Interview Questions & Answers

### Beginner Questions
**Q: What is the difference between infinite scroll and lazy loading?**
A: Infinite scroll is a UI pattern where lists append continuously as users scroll. Lazy loading is a broader optimization strategy deferring data/resource loads until absolutely necessary.

**Q: What is `onloadmore` in `lightning-datatable`?**
A: An event fired automatically when the user scrolls near the bottom of the table, indicating it's time to fetch more records.

### Intermediate Questions
**Q: How do you append records in LWC instead of replacing them?**
A: Using the JavaScript spread syntax: `this.records = [...this.records, ...newRecords];`.

**Q: Why use `isLoading` during infinite scrolling?**
A: To act as a lock. Without it, a user scrolling rapidly could fire multiple identical Apex calls, resulting in duplicate records and wasted limits.

**Q: What are the limitations of OFFSET?**
A: Salesforce limits OFFSET to 2,000. It also suffers from performance degradation on deep scrolls because the database must scan and skip rows.

### Advanced / Architect-Level Questions
**Q: Explain Keyset pagination and why it is superior for Large Data Volumes (LDV).**
A: Keyset (cursor) pagination uses the last sorted values (e.g., `CreatedDate` and `Id`) as a cursor to query the next batch (`WHERE CreatedDate < :lastDate`). It scales infinitely with O(1) performance because the database uses indexes to seek directly to the row, bypassing the 2,000 OFFSET limit.

**Q: Why is stable sorting critical in infinite scrolling?**
A: If ordering by a non-unique field (like `CreatedDate`), multiple records may have the exact same value. When fetching the next batch, the database doesn't guarantee order among duplicates, causing records to be skipped or repeated. Using `Id` as a secondary sort (`ORDER BY CreatedDate DESC, Id DESC`) provides determinism.

**Q: How do you manage infinite scroll state when a user applies a new search filter?**
A: The entire pagination state must be reset. The existing `records` array must be cleared, the page/cursor reset to its initial state, `hasMoreRecords` reset to true, and a new call initiated for Batch 1.

# 65. Revision Summary

*   **Infinite Scroll:** UI pattern for continuous appending.
*   **Lazy Loading:** Strategy of deferring load until needed.
*   **LWC Datatable:** Requires `enable-infinite-loading`, `onloadmore`, `load-more-offset`.
*   **State:** Requires `records` array, `pageSize`, `currentPage`/cursors, `isLoading`, `hasMoreRecords`.
*   **Appending:** JS Spread operator `[...]`.
*   **Apex:** Prefer imperative with `async/await` for easier state control over wired methods.
*   **OFFSET:** Easy, but limited to 2,000 rows. Drops performance on deep scrolls.
*   **Keyset Pagination:** Complex, but highly scalable. Uses fields (like `Id`) as cursors.
*   **Stable Sorting:** Crucial. Always add `Id DESC`.
*   **State Reset:** Required when Search, Filter, or Sort parameters change.
*   **Debouncing:** Delaying Apex search queries to prevent limit breaches.
*   **Security:** Always use `WITH USER_MODE` and avoid SOQL injection in dynamic query building.