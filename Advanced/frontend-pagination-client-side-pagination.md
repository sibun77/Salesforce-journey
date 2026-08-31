# Frontend Pagination – Client-Side Pagination

# 1. Introduction

Pagination is the process of dividing a large dataset into discrete pages, allowing users to view a limited number of records at a time. In enterprise web applications, displaying thousands of rows on a single screen causes severe performance degradation, browser crashes, and a poor user experience. Pagination solves this by presenting data in digestible chunks.

**Frontend Pagination** (specifically **Client-Side Pagination**) refers to the technique where the entire dataset is retrieved from the server (Salesforce) once, loaded into the browser's memory, and divided into pages using JavaScript. 

Lightning Web Components (LWC) applications frequently require pagination to maintain rendering performance, especially when utilizing components like `<lightning-datatable>`. Even if the browser can handle the memory, rendering DOM elements for thousands of records simultaneously will freeze the UI.

### Salesforce Automotive CRM Examples
Client-side pagination is highly effective for moderately sized lists, such as:
*   **Warranty Claims:** A service center viewing their 200 open claims for the week.
*   **Work Orders:** A technician reviewing their 50 assigned jobs for the month.
*   **Spare Parts:** A dealer searching through a filtered catalog of 500 available engine components.
*   **Invoices:** A finance user reviewing 150 pending dealer invoices.
*   **Dealer Records:** An executive viewing a list of 300 regional dealerships.
*   **Vehicles:** A salesperson browsing 100 available cars in the local lot inventory.
*   **Customers:** A list of 250 recent purchasers for a follow-up campaign.
*   **Shipment Records:** A logistics manager tracking 400 active part shipments.

---

# 2. What is Client-Side Pagination?

Client-side pagination is a UI pattern where the client (browser) takes full responsibility for paginating data. The server's only job is to provide the complete dataset.

**How it works:**
1.  **Retrieve a dataset:** The LWC calls an Apex method or uses Lightning Data Service to fetch records.
2.  **Store the dataset in the browser:** The LWC saves the full response in a JavaScript array (e.g., `allRecords`).
3.  **Divide the dataset into pages:** JavaScript calculates which records belong on which page based on a defined page size.
4.  **Display only the records for the current page:** The LWC extracts a "slice" of the array and binds it to the UI (e.g., `displayedRecords`).
5.  **Change the displayed records when the user navigates:** Clicking "Next" or "Previous" simply extracts a different slice from the already-loaded `allRecords` array without making a new server request.

### Flow Diagram

```text
Salesforce Data (Database)
      ↓
Apex / LDS (Server-Side)
      ↓
LWC (Browser/Client)
      ↓
Complete Dataset (Stored in JS Memory)
      ↓
Pagination Logic (JS Array slicing)
      ↓
Current Page Records (Reassigned array)
      ↓
UI / Datatable (Rendered to DOM)
```

**Step Breakdown:**
*   **Salesforce Data:** The source of truth (e.g., `Warranty_Claim__c` records).
*   **Apex / LDS:** The transport layer bringing data from the database to the client.
*   **Complete Dataset:** A JavaScript property holding all retrieved records.
*   **Pagination Logic:** Mathematical formulas determining start and end indices.
*   **Current Page Records:** The subset of data destined for the current view.
*   **UI / Datatable:** The visual representation of the current page.

---

# 3. Why Use Client-Side Pagination?

### Advantages
*   **Simple implementation:** Requires minimal Apex logic; all logic resides in JavaScript.
*   **Fast page navigation:** Transitioning between pages is instantaneous because no server round-trip is required.
*   **No server request for every page:** Reduces server CPU time and unnecessary network traffic after the initial load.
*   **Good user experience:** Zero latency when clicking "Next", "Previous", or specific page numbers.
*   **Easy integration with `lightning-datatable`:** Passes perfectly into the `data` attribute.
*   **Easy sorting:** You can sort the entire `allRecords` array in JavaScript instantaneously.
*   **Easy filtering:** You can filter the `allRecords` array in JavaScript instantaneously.
*   **Easy searching:** A search box can instantly filter the array without SOQL text matching.
*   **Suitable for small and moderate datasets:** Perfect for lists ranging from a few dozen to a couple of thousand records (depending on field count).

### Disadvantages
*   **Entire dataset must be loaded:** The initial load takes longer because all records are fetched at once.
*   **Browser memory usage:** Holding thousands of complex objects in JavaScript can bloat browser memory, especially on mobile devices.
*   **Initial loading time:** Fetching 2,000 records takes significantly longer than fetching 20.
*   **Not suitable for very large datasets:** If you have 50,000 records, client-side pagination will crash the browser or hit Apex heap limits.
*   **Salesforce query limits still apply:** You cannot bypass the 50,000 SOQL row limit or the 6MB sync heap size limit.
*   **Potential performance issues with large arrays:** Heavy client-side sorting or filtering on massive arrays can block the JavaScript main thread, causing UI stutter.

---

# 4. Client-Side Pagination vs Server-Side Pagination

| Feature | Client-Side Pagination | Server-Side Pagination (Backend) |
| :--- | :--- | :--- |
| **Data loading** | Loads ALL records at once. | Loads ONLY the current page's records. |
| **Server requests** | One initial request. | One request *per page navigation*. |
| **Browser memory** | High (stores the whole dataset). | Low (stores only the current page). |
| **Performance** | Slow initial load, lightning-fast navigation. | Fast initial load, slight delay on navigation. |
| **Scalability** | Poor for large datasets (>2,000 records). | Excellent for massive datasets (Millions). |
| **Initial load** | Heavier, as it transfers the entire payload. | Lighter, minimal data transferred. |
| **Navigation** | Instantaneous (JS array manipulation). | Requires a network round-trip. |
| **SOQL** | `SELECT Id FROM Object` (No limits applied). | `SELECT Id FROM Object LIMIT 10 OFFSET 20`. |
| **LIMIT / OFFSET** | Not used. | Core to the architecture. |
| **Large datasets** | Risks Heap Size limits and UI freezing. | Safe, robust, and necessary. |
| **Complexity** | Low (Handled via JS). | High (Requires Apex state management). |
| **Best use cases** | Related lists, filtered reports, moderate UI grids. | Global searches, massive lists, integrations. |

**Conclusion:** 
Client-side pagination is appropriate when the dataset is reasonably small (e.g., under 1,000-2,000 records). Server-side pagination should generally be preferred when the dataset is large or expected to grow indefinitely.

---

# 5. Basic Pagination Concepts

To build pagination, you must understand state variables:

*   **Total Records:** The total number of items in the complete dataset.
*   **Page Size:** The number of records displayed on a single page.
*   **Current Page:** The page number the user is currently viewing.
*   **Total Pages:** The total number of pages required to display all records.
*   **Start Index:** The array index of the first record on the current page.
*   **End Index:** The array index where the slice should stop (exclusive).
*   **Current Page Records:** The subset of records currently visible.

### Example Scenario
Total Records = 47
Page Size = 10

*   **Page 1** → Records 1–10 (Indices 0 to 9)
*   **Page 2** → Records 11–20 (Indices 10 to 19)
*   **Page 3** → Records 21–30 (Indices 20 to 29)
*   **Page 4** → Records 31–40 (Indices 30 to 39)
*   **Page 5** → Records 41–47 (Indices 40 to 46)

---

# 6. Pagination Formula

Pagination relies on basic arithmetic applied to array indices.

**1. Total pages:**
```javascript
Math.ceil(totalRecords / pageSize)
```
*Example:* `Math.ceil(47 / 10)` = `Math.ceil(4.7)` = `5` pages.

**2. Start index:**
```javascript
(pageNumber - 1) * pageSize
```
*Example (Page 3):* `(3 - 1) * 10` = `20`. The array starts at index 20.

**3. End index:**
```javascript
startIndex + pageSize
```
*Example (Page 3):* `20 + 10` = `30`. The array slicing stops at index 30 (not inclusive).

**4. Current page records:**
```javascript
records.slice(startIndex, endIndex)
```
*Example (Page 3):* `records.slice(20, 30)` will return elements from index 20 up to (but not including) index 30.

---

# 7. JavaScript Array.slice()

The core engine of client-side pagination is `Array.prototype.slice()`.

### Example
```javascript
this.displayedRecords = this.allRecords.slice(startIndex, endIndex);
```

### Explanation
*   **What `slice()` does:** It returns a shallow copy of a portion of an array into a new array object. 
*   **Start index:** The zero-based index at which to start extraction. If start is `0`, it starts at the first element.
*   **End index:** The zero-based index *before* which to end extraction. `slice(0, 10)` extracts elements 0 through 9.
*   **Why `slice()` is useful for pagination:** It perfectly isolates the exact chunk of data needed for the datatable without writing manual `for` loops.
*   **Why the original array is not modified:** `slice()` is non-destructive. `this.allRecords` remains completely intact, ensuring you don't lose data when navigating pages.

---

# 8. Basic LWC Pagination Structure

A robust client-side pagination component follows a strict data flow architecture:

```text
HTML (Template - Renders UI)
 ↓
JavaScript (Controller - Handles state and user interaction)
 ↓
Apex / LDS (Data Fetching)
 ↓
Complete Dataset (allRecords)
 ↓
Pagination Logic (Math & slice())
 ↓
Displayed Records (bound back to HTML)
```

**Responsibilities:**
*   **HTML:** Renders the `lightning-datatable`, "Next", "Previous", and page size dropdown. Dispatches user clicks to JS.
*   **JavaScript:** Orchestrates the flow. Fetches data, calculates indices, slices the array, and updates tracked properties.
*   **Apex:** Executes the SOQL query and returns the list of objects.
*   **Pagination State:** Variables in JS (`currentPage`, `totalPages`) that act as the source of truth for what the user should see.

---

# 9. Basic Client-Side Pagination Example

**Scenario:** Display Warranty Claims.
**Requirements:** Retrieve all claims, show 10 per page, allow Next/Previous navigation, disable buttons when appropriate.

### Apex: `WarrantyClaimController.cls`
```java
public with sharing class WarrantyClaimController {
    @AuraEnabled(cacheable=true)
    public static List<Warranty_Claim__c> getClaims() {
        return [SELECT Id, Name, Status__c, Claim_Amount__c 
                FROM Warranty_Claim__c 
                ORDER BY CreatedDate DESC 
                LIMIT 500];
    }
}
```
*Line-by-line Explanation:*
*   `@AuraEnabled(cacheable=true)`: Makes method accessible to LWC and caches results.
*   `public static List<Warranty_Claim__c>`: Returns a list of sObjects.
*   `SELECT... LIMIT 500`: Retrieves a moderate dataset for client-side pagination.

### JavaScript: `warrantyPagination.js`
```javascript
import { LightningElement, wire } from 'lwc';
import getClaims from '@salesforce/apex/WarrantyClaimController.getClaims';

export default class WarrantyPagination extends LightningElement {
    allRecords = [];
    displayedRecords = [];
    currentPage = 1;
    pageSize = 10;
    totalPages = 0;
    
    @wire(getClaims)
    wiredClaims({ error, data }) {
        if (data) {
            this.allRecords = data;
            this.totalPages = Math.ceil(this.allRecords.length / this.pageSize);
            this.updateDisplayedRecords();
        } else if (error) {
            console.error('Error fetching claims', error);
        }
    }

    updateDisplayedRecords() {
        const startIndex = (this.currentPage - 1) * this.pageSize;
        const endIndex = startIndex + this.pageSize;
        this.displayedRecords = this.allRecords.slice(startIndex, endIndex);
    }

    handlePrevious() {
        if (this.currentPage > 1) {
            this.currentPage--;
            this.updateDisplayedRecords();
        }
    }

    handleNext() {
        if (this.currentPage < this.totalPages) {
            this.currentPage++;
            this.updateDisplayedRecords();
        }
    }

    get isFirstPage() {
        return this.currentPage === 1;
    }

    get isLastPage() {
        return this.currentPage === this.totalPages;
    }
}
```
*Line-by-line Explanation:*
*   `allRecords = []`: Holds the master list.
*   `@wire...`: Fetches data from Apex.
*   `this.allRecords = data`: Stores the complete response.
*   `this.totalPages = ...`: Calculates total pages using `Math.ceil()`.
*   `updateDisplayedRecords()`: Core engine that slices `allRecords` into `displayedRecords`.
*   `handlePrevious() / handleNext()`: Adjusts the `currentPage` and triggers a slice update.
*   `get isFirstPage() / get isLastPage()`: Controls button disabled states.

### HTML: `warrantyPagination.html`
```html
<template>
    <lightning-card title="Warranty Claims">
        <div class="slds-m-around_medium">
            <!-- Datatable -->
            <lightning-datatable
                key-field="Id"
                data={displayedRecords}
                columns={columns}>
            </lightning-datatable>

            <!-- Pagination Controls -->
            <div class="slds-grid slds-grid_align-center slds-m-top_medium slds-grid_vertical-align-center">
                <lightning-button 
                    label="Previous" 
                    onclick={handlePrevious} 
                    disabled={isFirstPage}
                    class="slds-m-right_small">
                </lightning-button>
                
                <span class="slds-m-horizontal_medium">
                    Page {currentPage} of {totalPages}
                </span>
                
                <lightning-button 
                    label="Next" 
                    onclick={handleNext} 
                    disabled={isLastPage}
                    class="slds-m-left_small">
                </lightning-button>
            </div>
        </div>
    </lightning-card>
</template>
```
*Line-by-line Explanation:*
*   `data={displayedRecords}`: Datatable ONLY receives the sliced array, never the full array.
*   `disabled={isFirstPage}`: Prevents user from clicking Previous on Page 1.
*   `Page {currentPage} of {totalPages}`: Provides visual feedback to the user.

---

# 10. Pagination State Management

Maintaining clean state is vital. Your class properties are your state.

*   `allRecords`: The immutable source of truth. Never modify this array directly unless handling CRUD operations.
*   `displayedRecords`: The mutable view. This is reassigned every time the page changes.
*   `currentPage`: Integer representing the user's current view. Must be between `1` and `totalPages`.
*   `pageSize`: Integer determining chunk size (e.g., 10, 20, 50).
*   `totalRecords`: Integer tracking `allRecords.length`. Useful for UI display.
*   `totalPages`: Integer calculated dynamically based on `totalRecords` and `pageSize`.

---

# 11. Calculating Total Pages

```javascript
this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
```

**Edge Cases to handle:**
*   **0 records:** `0 / 10 = 0`. `totalPages` becomes `0`. (UI should handle 0 pages gracefully).
*   **1 record:** `1 / 10 = 0.1`. `Math.ceil(0.1)` = `1` page.
*   **10 records:** `10 / 10 = 1`. `Math.ceil(1)` = `1` page.
*   **11 records:** `11 / 10 = 1.1`. `Math.ceil(1.1)` = `2` pages.
*   **100 records:** `100 / 10 = 10`. `Math.ceil(10)` = `10` pages.

*Note:* Always use `Math.ceil()` (ceiling) because any remainder means you need an additional page to hold the overflowing records.

---

# 12. Updating Displayed Records

Create a central, reusable function for slicing. Do not repeat `slice()` logic in multiple methods.

```javascript
updateDisplayedRecords() {
    const startIndex = (this.currentPage - 1) * this.pageSize;
    const endIndex = startIndex + this.pageSize;

    this.displayedRecords = this.allRecords.slice(startIndex, endIndex);
}
```

*Line-by-line Explanation:*
*   `const startIndex...`: Maps the 1-based page number to a 0-based array index.
*   `const endIndex...`: Determines the ceiling limit for the slice. It is safe if `endIndex` exceeds array length; JavaScript's `slice()` automatically stops at the end of the array.
*   `this.displayedRecords = ...`: Assigning a new array to `displayedRecords` triggers LWC reactivity, updating the DOM.

---

# 13. Next Page

```javascript
handleNext() {
    if (this.currentPage < this.totalPages) {
        this.currentPage++;
        this.updateDisplayedRecords();
    }
}
```
*   **Increment currentPage:** Moves the pointer forward.
*   **Prevent going beyond totalPages:** The `if` condition ensures we don't paginate to empty space.
*   **Update displayed records:** Recalculates the slice and refreshes the UI.

---

# 14. Previous Page

```javascript
handlePrevious() {
    if (this.currentPage > 1) {
        this.currentPage--;
        this.updateDisplayedRecords();
    }
}
```
*   **Decrement currentPage:** Moves the pointer backward.
*   **Prevent going below page 1:** Arrays don't have negative indices. The `if` check protects against this.
*   **Update displayed records:** Refreshes the UI.

---

# 15. First Page

Sometimes jumping to the start is necessary.

```javascript
handleFirst() {
    this.currentPage = 1;
    this.updateDisplayedRecords();
}
```
*   **Set currentPage = 1:** Instantly resets pointer.
*   **Refresh displayed records:** Re-slices from index 0.
*   **Disable:** Handled automatically by the `isFirstPage` getter.

---

# 16. Last Page

```javascript
handleLast() {
    this.currentPage = this.totalPages;
    this.updateDisplayedRecords();
}
```
*   **Set currentPage = totalPages:** Instantly moves pointer to the end.
*   **Refresh displayed records:** Re-slices based on the final page index.
*   **Disable:** Handled automatically by the `isLastPage` getter.

---

# 17. Disabling Pagination Buttons

You must prevent users from clicking "Previous" on page 1, and "Next" on the last page.

```html
<lightning-button-group>
    <lightning-button label="First" onclick={handleFirst} disabled={isFirstPage}></lightning-button>
    <lightning-button label="Previous" onclick={handlePrevious} disabled={isFirstPage}></lightning-button>
    <lightning-button label="Next" onclick={handleNext} disabled={isLastPage}></lightning-button>
    <lightning-button label="Last" onclick={handleLast} disabled={isLastPage}></lightning-button>
</lightning-button-group>
```

Using getters (`isFirstPage`, `isLastPage`) keeps the HTML template clean and leverages LWC's reactivity to automatically evaluate whenever `currentPage` changes.

---

# 18. Getters for Pagination State

Getters dynamically evaluate state.

```javascript
get isFirstPage() {
    return this.currentPage === 1;
}

get isLastPage() {
    return this.currentPage === this.totalPages || this.totalPages === 0;
}
```
*   Why useful? They prevent you from having to manually set `this.isFirstPage = true` every time you update the page. The framework evaluates getters automatically during the render cycle.
*   *Note:* The `|| this.totalPages === 0` handles the edge case of an empty dataset, preventing "Next" from being clickable when there is no data.

---

# 19. Pagination Information

Users expect to see exactly where they are in the dataset: *"Showing 11–20 of 47 records"*

```javascript
get recordStart() {
    return this.totalRecords === 0 ? 0 : ((this.currentPage - 1) * this.pageSize) + 1;
}

get recordEnd() {
    return Math.min(this.currentPage * this.pageSize, this.totalRecords);
}

get paginationInfo() {
    return `Showing ${this.recordStart}–${this.recordEnd} of${this.totalRecords} records`;
}
```

*   **Start record:** `((currentPage - 1) * pageSize) + 1`. (Page 2, Size 10 = ((2-1)*10)+1 = 11).
*   **End record:** `Math.min()` ensures that on the last page (e.g., page 5 of 47 records), it says "Showing 41-47", not "Showing 41-50".
*   **Total records:** Bound to `allRecords.length`.

---

# 20. Page Size

Different users have different screen sizes. Allowing configurable page sizes improves UX.

### HTML
```html
<lightning-combobox
    name="pageSize"
    label="Records per page"
    value={pageSizeString}
    options={pageSizeOptions}
    onchange={handlePageSizeChange}>
</lightning-combobox>
```

### JavaScript Options
```javascript
get pageSizeOptions() {
    return [
        { label: '5', value: '5' },
        { label: '10', value: '10' },
        { label: '20', value: '20' },
        { label: '50', value: '50' }
    ];
}

get pageSizeString() {
    return this.pageSize.toString();
}
```
When page size changes, you must:
1. Update `pageSize`.
2. Reset `currentPage` to 1 (because the previous page boundary is now invalid).
3. Recalculate `totalPages`.
4. Recalculate `displayedRecords`.

---

# 21. Dynamic Page Size

```javascript
handlePageSizeChange(event) {
    // 1. Update pageSize (Convert string back to number)
    this.pageSize = parseInt(event.detail.value, 10);
    
    // 2. Reset currentPage to 1
    this.currentPage = 1;
    
    // 3. Recalculate totalPages
    this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
    
    // 4. Update the sliced array
    this.updateDisplayedRecords();
}
```
*Line-by-line Explanation:*
*   `parseInt`: Combobox values are strings. Math requires numbers.
*   `currentPage = 1`: Resetting prevents a scenario where a user is on page 5 (size 10), changes to size 50, and ends up on a blank page 5.
*   `totalPages`: Must be recalculated because the divisor changed.
*   `updateDisplayedRecords`: Refreshes the UI.

---

# 22. Page Number Buttons

Displaying `1 2 3 4 5` allows quick navigation.

### JavaScript
```javascript
get pageList() {
    let pages = [];
    for (let i = 1; i <= this.totalPages; i++) {
        pages.push({
            pageNumber: i,
            variant: i === this.currentPage ? 'brand' : 'neutral' // Highlight active page
        });
    }
    return pages;
}

handlePageClick(event) {
    this.currentPage = parseInt(event.target.dataset.page, 10);
    this.updateDisplayedRecords();
}
```

### HTML
```html
<template for:each={pageList} for:item="page">
    <lightning-button 
        key={page.pageNumber}
        label={page.pageNumber} 
        data-page={page.pageNumber}
        variant={page.variant}
        onclick={handlePageClick}
        class="slds-m-horizontal_xxx-small">
    </lightning-button>
</template>
```
*Explanation:* We generate an array of objects. `variant='brand'` visually highlights the currently active page. `data-page` passes the target page number to the `handlePageClick` event.

---

# 23. Pagination with lightning-datatable

Integration is seamless. `lightning-datatable` knows nothing about pagination; it only knows about `displayedRecords`.

### Architecture
```text
All Records (500 items)
    ↓
Pagination (Math + slice)
    ↓
displayedRecords (10 items)
    ↓
lightning-datatable (Renders 10 rows)
```

### Example
```html
<lightning-datatable
    key-field="Id"
    data={displayedRecords}
    columns={columns}
    hide-checkbox-column="true">
</lightning-datatable>
```
Because `displayedRecords` is completely replaced by `slice()` every time the page changes, LWC's diffing engine efficiently re-renders the rows in the datatable.

---

# 24. Pagination with Search

Search and pagination often conflict if not implemented properly. 
**Rule:** Filter the dataset *first*, then paginate the *filtered dataset*.

### Flow
```text
All Records (e.g., 500 Claims)
     ↓
Search Filter (e.g., 'Engine')
     ↓
Filtered Records (e.g., 40 Claims)
     ↓
Pagination Math (Total pages based on 40, not 500)
     ↓
Displayed Records (e.g., 10 records of the 40)
```

**Crucial Error:** If you paginate first, and search second, you are only searching the *current page*. Users hate this.

### JavaScript Example
```javascript
handleSearch(event) {
    const searchKey = event.target.value.toLowerCase();
    
    // 1. Filter the complete dataset
    if (searchKey) {
        this.filteredRecords = this.allRecords.filter(record => 
            record.Name.toLowerCase().includes(searchKey)
        );
    } else {
        this.filteredRecords = [...this.allRecords]; // Reset to all
    }
    
    // 2. Reset page & recalculate
    this.totalRecords = this.filteredRecords.length;
    this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
    this.currentPage = 1; // Reset to page 1 for new search results
    
    // 3. Display the filtered page
    this.updateDisplayedRecords();
}

// Modify updateDisplayedRecords to use filteredRecords!
updateDisplayedRecords() {
    const start = (this.currentPage - 1) * this.pageSize;
    const end = start + this.pageSize;
    this.displayedRecords = this.filteredRecords.slice(start, end);
}
```

---

# 25. Pagination with Filtering

Similar to search, filtering (e.g., by Status = 'Approved') requires a distinct `filteredRecords` array.

### Example: Filter Warranty Claims by Status
```javascript
handleStatusFilter(event) {
    const status = event.detail.value;
    
    if (status === 'All') {
        this.filteredRecords = [...this.allRecords];
    } else {
        this.filteredRecords = this.allRecords.filter(claim => claim.Status__c === status);
    }
    
    this.totalRecords = this.filteredRecords.length;
    this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
    this.currentPage = 1;
    this.updateDisplayedRecords();
}
```
Filtering significantly changes `totalRecords`. Always reset `currentPage = 1` because if you are on page 5, and the filter reduces the total pages to 2, staying on page 5 will show a blank table.

---

# 26. Pagination with Sorting

Sorting must happen on the `filteredRecords` array *before* you slice it. If you sort `displayedRecords`, you are only sorting the 10 rows on the current screen.

### Correct Flow
```text
All Records
     ↓
Sort (e.g., Amount DESC)
     ↓
Filtered/Sorted Records
     ↓
Pagination (slice)
     ↓
Displayed Records
```

### JavaScript Example
```javascript
handleSort(event) {
    const { fieldName: sortedBy, sortDirection } = event.detail;
    
    // Clone array to sort
    let cloneData = [...this.filteredRecords];
    
    cloneData.sort((a, b) => {
        let valA = a[sortedBy] ? a[sortedBy] : '';
        let valB = b[sortedBy] ? b[sortedBy] : '';
        let reverse = sortDirection === 'asc' ? 1 : -1;
        return reverse * ((valA > valB) - (valB > valA));
    });

    this.filteredRecords = cloneData;
    this.currentPage = 1; // Good practice, though optional depending on UX rules
    this.updateDisplayedRecords();
}
```
*Why sort before slicing?* So the #1 largest amount in the entire dataset shows up on Page 1, Row 1.

---

# 27. Search + Filter + Sort + Pagination

When combined, the order of operations is vital to a stable application.

### Architecture
```text
Complete Dataset (allRecords)
      ↓
Search & Filter (Produces filteredRecords)
      ↓
Sort (Rearranges filteredRecords)
      ↓
Pagination (Calculates slice based on filteredRecords)
      ↓
Displayed Records (bound to UI)
```

**Order explanation:**
1. You can't sort data that is filtered out, so filter/search first.
2. You must sort the remaining dataset before paginating, so the first page gets the correct top results.
3. Pagination is simply the final viewport window on the properly organized data.

---

# 28. Reactive Properties

In LWC, the UI updates when tracked properties change.
For primitives (`currentPage`, `pageSize`), reassignment triggers a render.
For arrays/objects, modifying internals (like `array.push()`) without `@track` won't trigger a render.

However, `Array.slice()` returns a **new array instance**.
```javascript
this.displayedRecords = this.filteredRecords.slice(start, end);
```
Because you are reassigning the property `this.displayedRecords` to a brand-new array object every time the page changes, LWC reactivity detects the change perfectly. `@track` is not required for `displayedRecords`.

---

# 29. Loading State

Fetching the initial dataset can take time. Provide visual feedback.

### Flow
`Load Data` → `isLoading = true` → `Apex` → `Data Received` → `Pagination` → `isLoading = false`

### Example
```javascript
isLoading = true;

connectedCallback() {
    getClaims()
        .then(data => {
            this.allRecords = data;
            this.filteredRecords = [...data];
            this.initializePagination();
        })
        .finally(() => {
            this.isLoading = false; // Always turn off spinner
        });
}
```
### HTML
```html
<template if:true={isLoading}>
    <lightning-spinner alternative-text="Loading" size="medium"></lightning-spinner>
</template>
```

---

# 30. Empty State

Handle scenarios where there is no data, or a search yields zero results.

### JavaScript
```javascript
get hasRecords() {
    return this.displayedRecords && this.displayedRecords.length > 0;
}
```

### HTML
```html
<template if:true={hasRecords}>
    <lightning-datatable ...></lightning-datatable>
</template>
<template if:false={hasRecords}>
    <div class="slds-illustration slds-illustration_small" aria-hidden="true">
        <img src="/img/chatter/OpenRoad.svg" class="slds-illustration__svg" alt=""/>
        <div class="slds-text-longform">
            <h3 class="slds-text-heading_medium">No records found</h3>
            <p class="slds-text-body_regular">Try adjusting your filters or search criteria.</p>
        </div>
    </div>
</template>
```
Provide a user-friendly illustration rather than an empty, broken datatable grid.

---

# 31. Error Handling

Handle server failures gracefully so the pagination UI doesn't break unexpectedly.

### JavaScript
```javascript
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

// Inside wire or imperative Apex
.catch(error => {
    this.allRecords = [];
    this.filteredRecords = [];
    this.displayedRecords = [];
    this.totalRecords = 0;
    
    this.dispatchEvent(
        new ShowToastEvent({
            title: 'Error loading records',
            message: error.body ? error.body.message : 'Unknown error',
            variant: 'error'
        })
    );
})
```
By resetting arrays to empty, you prevent the pagination math from throwing `NaN` or `undefined` errors.

---

# 32. Pagination After Data Refresh

If you call `refreshApex()` or imperatively fetch data again, the total dataset size might change.

**Scenario:** 
*   Current Page = 5
*   Records are deleted on the server.
*   Data refreshes. Total pages becomes 4.
*   *Danger:* Current Page is still 5, which is now an empty page.

### Solution
```javascript
refreshData() {
    this.isLoading = true;
    getClaims().then(data => {
        this.allRecords = data;
        this.filteredRecords = [...data];
        this.totalRecords = this.filteredRecords.length;
        this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
        
        // Safety Check: Clamp current page to valid range
        this.currentPage = Math.min(this.currentPage, this.totalPages);
        
        // If data is entirely empty, currentPage should be 1, not 0
        if (this.currentPage === 0) this.currentPage = 1;
        
        this.updateDisplayedRecords();
    }).finally(() => { this.isLoading = false; });
}
```

---

# 33. Pagination After Record Deletion

When a user deletes a record directly from the datatable using a row action.

### Flow
`Delete Record (Apex)` → `Update Dataset (Splice JS array)` → `Recalculate Total Records` → `Recalculate Total Pages` → `Correct Current Page` → `Update Displayed Records`

### Example
```javascript
handleRowAction(event) {
    const action = event.detail.action;
    const rowId = event.detail.row.Id;
    
    if (action.name === 'delete') {
        deleteRecord(rowId).then(() => {
            // Remove from local dataset to avoid a full server round-trip
            this.allRecords = this.allRecords.filter(r => r.Id !== rowId);
            this.filteredRecords = this.filteredRecords.filter(r => r.Id !== rowId);
            
            this.totalRecords = this.filteredRecords.length;
            this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
            
            // Adjust page if we deleted the last item on the current last page
            this.currentPage = Math.min(this.currentPage, this.totalPages || 1);
            
            this.updateDisplayedRecords();
        });
    }
}
```

---

# 34. Pagination After Record Creation

When a new record is created (e.g., via a modal in the same component).

### Flow
`Create Record` → `Add Record to Dataset` → `Recalculate Pagination` → `Display Updated Data`

### Logic
If a new Warranty Claim is created, should it appear first? Last?
Normally, you prepend it to the array (`unshift`) so it appears at the top of Page 1.

```javascript
handleNewClaim(newClaimRecord) {
    // Add to top of master list
    this.allRecords.unshift(newClaimRecord);
    this.filteredRecords.unshift(newClaimRecord);
    
    // Recalculate
    this.totalRecords = this.filteredRecords.length;
    this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
    
    // Move to page 1 to show the new record
    this.currentPage = 1;
    this.updateDisplayedRecords();
}
```

---

# 35. Pagination with LDS

Lightning Data Service (LDS) functions like `getRecord` or `getRecords` can supply the data.

*   **LDS Caching:** Data is automatically cached, meaning re-fetches are instantaneous.
*   **Client-Side Processing:** Once LDS returns the array, the pagination math is identical to Apex.

**When to use LDS vs Apex:**
*   Use LDS (`getRecords`) when fetching a small, known list of specific Ids.
*   Use Apex when you need complex SOQL queries (e.g., `ORDER BY`, complex `WHERE` clauses, retrieving hundreds of related list items). Apex is almost always preferred for powering datatables.

---

# 36. Pagination with Imperative Apex

Imperative Apex gives you control over *when* the data is fetched (e.g., on a button click, or during `connectedCallback`).

### Flow
```text
Imperative Apex
     ↓
Retrieve Dataset (Promise resolves)
     ↓
Store in allRecords
     ↓
Client-Side Pagination
```

**Imperative Apex + Client-Side vs Server-Side:**
*   *Imperative + Client-Side:* Fetches 1,000 records once. Slices in JS.
*   *Imperative + Server-Side:* Fetches 50 records. On "Next", imperative Apex is called *again* with `OFFSET 50`.

---

# 37. Pagination with Wire Service

Using `@wire` is reactive. If the underlying data changes and LDS detects it, the wire will provision new data automatically.

### Example
```javascript
@wire(getClaims)
wiredClaims({ data, error }) {
    if (data) {
        this.allRecords = data;
        this.filteredRecords = [...data]; // Initial state
        this.totalRecords = data.length;
        this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
        this.updateDisplayedRecords();
    } else if (error) {
        // Handle error
    }
}
```
*Implications:* Because `@wire` data is immutable, you *must* copy it to `allRecords` or `filteredRecords` if you plan to sort or filter it locally. 

---

# 38. Client-Side Pagination and Governor Limits

**CRITICAL CONCEPT:** Client-side pagination does NOT eliminate Salesforce governor limits.

If your Apex controller executes: `SELECT Id, Name FROM Warranty_Claim__c`
And there are 60,000 claims in the system:
1.  **SOQL Row Limit:** Apex will crash at 50,000 rows.
2.  **Heap Size Limit:** Serializing 50,000 records into JSON might exceed the 6MB synchronous heap limit.
3.  **CPU Time:** Apex CPU time increases parsing massive lists.
4.  **Browser Memory:** The browser must download a massive JSON payload (often causing the LWC to crash mobile browsers).

**Rule:** UI pagination $\neq$ Backend pagination. Client-side pagination only protects the DOM; it does not protect the Salesforce database or network payload.

---

# 39. Dataset Size Considerations

Use this guideline to decide between Client-Side and Server-Side pagination.

| Dataset Size | Recommended Strategy | Reasoning |
| :--- | :--- | :--- |
| **Small (< 500)** | **Client-side pagination** | Fast to load, instant navigation, simple to build. |
| **Moderate (500 - 2,500)**| **Depends on requirements** | Client-side is okay if fields are few. If large text fields, consider Server-side. |
| **Large (2,500 - 10,000)** | **Server-side pagination** | Heap limits and UI payload delays become noticeable. |
| **Very Large (> 10,000)** | **Server-side pagination + optimized SOQL** | Client-side will crash. Requires SOQL `OFFSET` or Keyset pagination. |

*Note:* Exact thresholds depend on the number of fields retrieved. 500 records with 3 text fields is tiny; 500 records with 50 fields including Rich Text is massive.

---

# 40. Performance Considerations

To optimize client-side pagination:
*   **Avoid loading unnecessary fields:** Only `SELECT` fields displayed in the datatable or used for filtering.
*   **Avoid loading unnecessarily large datasets:** Enforce a `LIMIT 1000` in Apex to protect against runaway data growth.
*   **Minimize server calls:** Fetch once.
*   **Avoid expensive client-side transformations:** Don't loop over `allRecords` multiple times doing heavy String manipulation.
*   **Avoid repeated filtering/sorting:** Cache the `filteredRecords` array so you aren't re-sorting every time the page changes.
*   **Keep displayedRecords limited:** Keep `pageSize` under 100 to maintain swift DOM rendering.

---

# 41. Memory Considerations

The `allRecords` array can consume significant JavaScript heap space.

*   **Browser memory:** Every object in the array takes up RAM. 
*   **Large SObjects:** A `Warranty_Claim__c` object with relationships (`Dealer__r.Name`, `Vehicle__r.VIN`) inflates object size.
*   **Large response payloads:** Salesforce sets a ~4MB payload limit for Aura/LWC responses. Huge lists will trigger a "Response size exceeded" error.
*   **Mobile devices:** Salesforce Mobile App webviews have strict memory caps. Large `allRecords` arrays will crash the mobile app.

---

# 42. Pagination Component Design

Instead of writing pagination logic in every LWC, build a reusable `<c-pagination>` component.

### Parent-Child Architecture
```text
Parent Component (e.g., WarrantyClaimList)
      ↓ passes dataset size / current state
Reusable Pagination Component (<c-pagination>)
      ↓ dispatches "pagechange" event on click
Parent Component (updates slicing and UI)
```

**Benefits:**
*   Consistent UI across the app.
*   DRY (Don't Repeat Yourself) principle.
*   Centralized bug fixes for pagination logic.

---

# 43. Reusable Pagination Component

### `pagination.js`
```javascript
import { LightningElement, api } from 'lwc';

export default class Pagination extends LightningElement {
    @api totalRecords = 0;
    @api pageSize = 10;
    @api currentPage = 1;

    get totalPages() {
        return Math.ceil(this.totalRecords / this.pageSize);
    }
    get isFirstPage() { return this.currentPage === 1; }
    get isLastPage() { return this.currentPage === this.totalPages || this.totalPages === 0; }
    get pageInfo() { return `Page ${this.currentPage} of${this.totalPages}`; }

    handlePrevious() {
        if (!this.isFirstPage) {
            this.dispatchEvent(new CustomEvent('pagechange', { detail: this.currentPage - 1 }));
        }
    }
    handleNext() {
        if (!this.isLastPage) {
            this.dispatchEvent(new CustomEvent('pagechange', { detail: this.currentPage + 1 }));
        }
    }
}
```

### `pagination.html`
```html
<template>
    <div class="slds-grid slds-grid_align-center slds-m-top_medium">
        <lightning-button-icon icon-name="utility:chevronleft" onclick={handlePrevious} disabled={isFirstPage}></lightning-button-icon>
        <span class="slds-m-horizontal_medium">{pageInfo}</span>
        <lightning-button-icon icon-name="utility:chevronright" onclick={handleNext} disabled={isLastPage}></lightning-button-icon>
    </div>
</template>
```

---

# 44. Pagination Component API Design

**Public Properties (`@api`):**
*   `@api totalRecords`: Parent tells child how many total items exist.
*   `@api pageSize`: Parent tells child the chunk size.
*   `@api currentPage`: Parent tells child where the pointer is.

**Custom Events:**
*   `pagechange`: Fired when a navigation button is clicked.
*   `event.detail`: Carries the *requested* new page number. The parent component listens for this event, updates its own `currentPage` variable, and runs `updateDisplayedRecords()`.

---

# 45. Accessibility

Pagination components must be accessible to screen readers and keyboard navigation.

*   **Button labels:** Use `aria-label="Previous Page"` on icon-only buttons.
*   **Keyboard navigation:** Native `lightning-button` components handle `tabindex` and `Enter` key automatically.
*   **Disabled states:** `disabled={isFirstPage}` correctly communicates to screen readers that the action is unavailable.
*   **Clear page information:** Adding `aria-live="polite"` to the "Page 1 of 5" text ensures screen readers announce the page change dynamically.

---

# 46. UX Best Practices

*   **Show current page & total pages:** Don't leave the user guessing.
*   **Show record range:** "Showing 1-10 of 50" gives immediate context on dataset size.
*   **Disable invalid navigation:** Never let a user click "Next" into an empty void.
*   **Provide page size selector:** Let power users view 50 records at once.
*   **Maintain sorting/filtering:** Never clear filters just because a user changed pages.
*   **Reset page after search:** If a user searches, jump them back to Page 1.
*   **Show loading state:** Spinners prevent multiple clicks during load.
*   **Show empty state:** A clear illustration is better than a blank datatable.

---

# 47. Common Mistakes

| Problem | Cause | Solution | Example |
| :--- | :--- | :--- | :--- |
| **1. Filtering only current page** | Paginating before filtering. | Create a pipeline: Filter -> Sort -> Paginate. | Search for 'Oil' only finds 'Oil' on Page 2, not Page 5. |
| **2. Sorting only current page** | Sorting `displayedRecords`. | Sort `allRecords` or `filteredRecords` instead. | Clicking "Amount" sorts the 10 visible rows, not the top 10 overall. |
| **3. Blank page after search** | Forgetting to reset `currentPage`. | Set `currentPage = 1` on search/filter. | On page 5. Search yields 2 total pages. Page stays 5 (blank). |
| **4. Array mutation issues** | Using `splice()` instead of `slice()`. | Use `slice()` as it is non-destructive. | `splice()` deletes items from the master array permanently. |
| **5. CPU Timeout / App Crash** | Loading 50,000 records. | Client-side pagination used for massive datasets. | Switch to Server-Side pagination with `OFFSET`. |
| **6. No response on UI update** | Assigning to object index instead of new array. | Reassign entire array: `this.displayedRecords = [...]` | LWC reactivity requires a new object reference to re-render. |

---

# 48. Debugging Client-Side Pagination

**Debugging Checklist:**
*   **Next/Prev not working:** `console.log(this.currentPage, this.totalPages)`. Are they what you expect?
*   **Blank page:** Ensure `currentPage` is not `0`. Ensure `startIndex` is not greater than `filteredRecords.length`.
*   **Last page missing records:** Ensure your `slice` logic is `slice(start, end)`. `slice` automatically stops at array boundary, so `endIndex` exceeding array length is perfectly fine.
*   **Sorting/Filtering failing:** Add `console.log` before and after sorting. Ensure you are sorting `filteredRecords` and then calling `updateDisplayedRecords()`.
*   **Duplicate records:** Ensure datatable `key-field` is mapped to a truly unique ID (like `Id`).

---

# 49. Complete End-to-End Example

**Scenario:** Warranty Claim Management.

### Apex: `WarrantyClaimController.cls`
```java
public with sharing class WarrantyClaimController {
    @AuraEnabled(cacheable=true)
    public static List<Warranty_Claim__c> getClaims() {
        return [SELECT Id, Name, Status__c, Claim_Amount__c, CreatedDate 
                FROM Warranty_Claim__c 
                ORDER BY CreatedDate DESC 
                LIMIT 1000];
    }
}
```

### HTML: `warrantyClaimManager.html`
```html
<template>
    <lightning-card title="Warranty Claims">
        <div class="slds-m-around_medium">
            <!-- Loading Spinner -->
            <template if:true={isLoading}>
                <lightning-spinner size="medium"></lightning-spinner>
            </template>

            <!-- Search and Filter -->
            <div class="slds-grid slds-gutters slds-m-bottom_medium">
                <div class="slds-col slds-size_1-of-3">
                    <lightning-input type="search" label="Search Claims" onchange={handleSearch}></lightning-input>
                </div>
            </div>

            <!-- Datatable -->
            <template if:true={hasRecords}>
                <lightning-datatable
                    key-field="Id"
                    data={displayedRecords}
                    columns={columns}
                    sorted-by={sortBy}
                    sorted-direction={sortDirection}
                    onsort={handleSort}
                    hide-checkbox-column>
                </lightning-datatable>

                <!-- Pagination -->
                <div class="slds-grid slds-grid_align-spread slds-m-top_medium slds-grid_vertical-align-center">
                    <div class="slds-col">
                        <lightning-combobox label="Page Size" value={pageSizeString} options={pageSizeOptions} onchange={handlePageSizeChange} variant="label-hidden"></lightning-combobox>
                    </div>
                    <div class="slds-col slds-grid slds-grid_vertical-align-center">
                        <lightning-button-icon icon-name="utility:chevronleft" onclick={handlePrevious} disabled={isFirstPage}></lightning-button-icon>
                        <span class="slds-m-horizontal_medium">{paginationInfo}</span>
                        <lightning-button-icon icon-name="utility:chevronright" onclick={handleNext} disabled={isLastPage}></lightning-button-icon>
                    </div>
                </div>
            </template>

            <!-- Empty State -->
            <template if:false={hasRecords}>
                <div class="slds-align_absolute-center slds-m-top_large">
                    <p>No claims found.</p>
                </div>
            </template>
        </div>
    </lightning-card>
</template>
```

### JavaScript: `warrantyClaimManager.js`
```javascript
import { LightningElement, wire, track } from 'lwc';
import getClaims from '@salesforce/apex/WarrantyClaimController.getClaims';

const COLUMNS = [
    { label: 'Claim Number', fieldName: 'Name', sortable: true },
    { label: 'Status', fieldName: 'Status__c', sortable: true },
    { label: 'Amount', fieldName: 'Claim_Amount__c', type: 'currency', sortable: true },
];

export default class WarrantyClaimManager extends LightningElement {
    columns = COLUMNS;
    isLoading = true;

    // Master Data
    allRecords = [];
    filteredRecords = [];
    @track displayedRecords = [];

    // Pagination State
    currentPage = 1;
    pageSize = 10;
    totalRecords = 0;
    totalPages = 0;

    // Sorting State
    sortBy;
    sortDirection;

    @wire(getClaims)
    wiredClaims({ data, error }) {
        if (data) {
            this.allRecords = data;
            this.filteredRecords = [...data];
            this.processPagination();
        }
        this.isLoading = false;
    }

    // --- PIPELINE: Search -> Sort -> Paginate ---
    
    handleSearch(event) {
        const searchKey = event.target.value.toLowerCase();
        
        if (searchKey) {
            this.filteredRecords = this.allRecords.filter(claim => 
                claim.Name.toLowerCase().includes(searchKey)
            );
        } else {
            this.filteredRecords = [...this.allRecords];
        }
        
        this.applySorting(); // Maintain sort during search
        this.currentPage = 1; // Reset page
        this.processPagination();
    }

    handleSort(event) {
        this.sortBy = event.detail.fieldName;
        this.sortDirection = event.detail.sortDirection;
        
        this.applySorting();
        this.currentPage = 1; // Reset page
        this.processPagination();
    }

    applySorting() {
        if (this.sortBy) {
            let isReverse = this.sortDirection === 'asc' ? 1 : -1;
            this.filteredRecords.sort((a, b) => {
                let aVal = a[this.sortBy] || '';
                let bVal = b[this.sortBy] || '';
                return isReverse * ((aVal > bVal) - (bVal > aVal));
            });
        }
    }

    processPagination() {
        this.totalRecords = this.filteredRecords.length;
        this.totalPages = Math.ceil(this.totalRecords / this.pageSize);
        
        const start = (this.currentPage - 1) * this.pageSize;
        const end = start + this.pageSize;
        this.displayedRecords = this.filteredRecords.slice(start, end);
    }

    // --- PAGINATION CONTROLS ---

    handlePrevious() {
        if (this.currentPage > 1) {
            this.currentPage--;
            this.processPagination();
        }
    }

    handleNext() {
        if (this.currentPage < this.totalPages) {
            this.currentPage++;
            this.processPagination();
        }
    }

    handlePageSizeChange(event) {
        this.pageSize = parseInt(event.detail.value, 10);
        this.currentPage = 1;
        this.processPagination();
    }

    // --- GETTERS ---

    get isFirstPage() { return this.currentPage === 1; }
    get isLastPage() { return this.currentPage === this.totalPages || this.totalPages === 0; }
    get hasRecords() { return this.displayedRecords.length > 0; }
    
    get paginationInfo() {
        if (this.totalRecords === 0) return '0 records';
        const start = ((this.currentPage - 1) * this.pageSize) + 1;
        const end = Math.min(this.currentPage * this.pageSize, this.totalRecords);
        return `Showing ${start}–${end} of${this.totalRecords}`;
    }
    
    get pageSizeString() { return this.pageSize.toString(); }
    get pageSizeOptions() {
        return [{label:'10', value:'10'}, {label:'25', value:'25'}];
    }
}
```

---

# 50. Complete Processing Flow

The most robust architectural pattern follows this exact pipeline whenever state changes:

```text
1. Fetch Apex / LDS
        ↓
2. Store in allRecords (Immutable master)
        ↓
3. Search (Filter allRecords -> filteredRecords)
        ↓
4. Filter (Dropdowns apply to filteredRecords)
        ↓
5. Sort (Applies to filteredRecords)
        ↓
6. Calculate Total Records (filteredRecords.length)
        ↓
7. Calculate Total Pages (Math.ceil)
        ↓
8. Calculate Start/End Index (Based on currentPage)
        ↓
9. array.slice() (Extracts chunk)
        ↓
10. Store in displayedRecords
        ↓
11. Bind to <lightning-datatable>
```
If you follow this pipeline sequentially, filtering, sorting, and pagination will never conflict.

---

# 51. Client-Side vs Backend Pagination Decision Guide

```text
Need to display records?
        ↓
How large is the dataset?
        ↓
    [< 2,000 records] -----------------> Client-Side Pagination
        ↓                                (Load once, slice in JS)
    [2,000 - 10,000 records] ----------> Server-Side Pagination
        ↓                                (SOQL LIMIT/OFFSET)
    [> 10,000 records / High volume] --> Optimized Server-Side Pagination
                                         + Selective SOQL (Indexed filters)
                                         + Keyset Pagination (WHERE Id > lastId)
```

**Rule of Thumb:** Base the strategy on dataset size, record complexity (number of fields), UI responsiveness expectations, and Salesforce platform limits.

---

# 52. Best Practices Checklist

*   ✅ **Keep complete dataset separate from displayed records:** Protect `allRecords` from mutation.
*   ✅ **Maintain currentPage explicitly:** Always track where the user is.
*   ✅ **Maintain pageSize explicitly:** Allow easy configuration.
*   ✅ **Calculate totalPages dynamically:** Use `Math.ceil()`.
*   ✅ **Use Array.slice() for page extraction:** Safe, immutable subset extraction.
*   ✅ **Apply search/filter/sort before pagination:** Ensures accurate global sorting/searching.
*   ✅ **Reset currentPage when search/filter changes:** Prevents users from getting stuck on blank pages.
*   ✅ **Recalculate pagination after CRUD operations:** Keep UI in sync with dataset.
*   ✅ **Disable invalid navigation buttons:** Prevent errors on Page 1 or final page.
*   ✅ **Handle empty datasets:** Provide user-friendly "No Data" states.
*   ✅ **Handle errors:** Catch Apex exceptions and display toasts.
*   ✅ **Show loading state:** Use spinners for initial load.
*   ✅ **Display record range:** "Showing 10-20 of 50".
*   ✅ **Avoid unnecessary fields:** Keep payload lightweight.
*   ✅ **Avoid loading huge datasets:** Enforce a hard SOQL limit (e.g., `LIMIT 1000`).
*   ✅ **Use server-side pagination for large datasets:** Respect Salesforce governor limits.
*   ✅ **Keep pagination logic reusable:** Use a separate component if used in multiple places.
*   ✅ **Use custom events for reusable pagination components:** Decouple logic from UI.
*   ✅ **Maintain accessibility:** Use ARIA attributes and descriptive labels.

---

# 53. Real Project Scenarios

### 1. Warranty Claim List
*   **Dataset:** Open claims for a specific Service Center (usually < 500).
*   **Approach:** Client-side pagination. Page size: 20. Users need fast sorting by Amount and filtering by Status.

### 2. Work Order List
*   **Dataset:** Technician's weekly tasks (~50 records).
*   **Approach:** Client-side pagination.

### 3. Spare Parts List
*   **Dataset:** Master catalog of 200,000 parts (SAP Integration).
*   **Approach:** Server-side pagination ONLY. Searching must occur via Apex SOSL/SOQL.

### 4. Dealer List
*   **Dataset:** Regional dealers (~300 records).
*   **Approach:** Client-side pagination.

### 5. Vehicle List
*   **Dataset:** All vehicles sold in North America (Millions).
*   **Approach:** Strict Server-side Keyset Pagination with indexed queries.

### 6. Invoice List
*   **Dataset:** Outstanding dealer invoices for the month (~200 records).
*   **Approach:** Client-side pagination.

### 7. Shipment List
*   **Dataset:** Active shipments globally (~5,000 records).
*   **Approach:** Server-side pagination to avoid UI stutter and heap limits.

### 8. Claim Line Items
*   **Dataset:** Items inside a single claim (usually < 20).
*   **Approach:** No pagination needed, or simple client-side pagination with page size 5.

---

# 54. Interview Questions & Answers

### Beginner Questions

**Q: What is pagination?**
A: Breaking a large dataset into smaller chunks (pages) for display in a UI, improving rendering performance and user experience.

**Q: What does Array.slice() do in LWC?**
A: It extracts a portion of an array into a new array object based on a start and end index without modifying the original array. It's the core engine for client-side pagination.

### Intermediate Questions

**Q: What is the difference between frontend and backend pagination?**
A: Frontend (client-side) pagination loads the *entire* dataset into the browser once and uses JavaScript to divide it. Backend (server-side) pagination uses SOQL (`LIMIT` and `OFFSET`) to fetch only the data needed for the current page, making a server call every time the page changes.

**Q: How do you calculate total pages?**
A: `Math.ceil(totalRecords / pageSize)`. You must use `ceil` (ceiling) to ensure any remainder gets its own final page.

**Q: Why should filtering happen before pagination?**
A: If you paginate first and filter second, the filter only applies to the records visible on the current page. The user will miss matching records that exist on other pages.

### Advanced Questions

**Q: How do you handle pagination after deleting a record?**
A: You remove the record from `allRecords`, update `filteredRecords`, recalculate `totalRecords` and `totalPages`. Critically, you must ensure `currentPage` is still valid (e.g., `this.currentPage = Math.min(this.currentPage, this.totalPages)`) and re-run `slice()`.

**Q: Does client-side pagination eliminate Salesforce governor limits?**
A: No. Client-side pagination only protects the browser's DOM rendering. Apex still has to query, serialize, and transmit the entire dataset. It is bound by the 50,000 SOQL row limit and the 6MB synchronous heap limit.

### Architect-Level Questions

**Q: How would you design pagination for an enterprise Salesforce application with a datatable expecting 50,000 records?**
A: Client-side pagination is impossible here due to UI payload and heap limits. I would architect a Server-Side pagination solution. Since `OFFSET` maxes out at 2,000, I would implement "Keyset Pagination" (Cursor-based pagination) using the `WHERE Id > :lastSeenId ORDER BY Id LIMIT 50` pattern. This guarantees scalability regardless of data volume.

---

# 55. Revision Summary

*   **Pagination:** Dividing datasets into manageable chunks.
*   **Frontend/Client-Side Pagination:** Loads all data to the browser, chunks it with JS.
*   **State variables:** `allRecords` (master), `displayedRecords` (UI bound), `currentPage`, `pageSize`, `totalRecords`, `totalPages`.
*   **Math:** `totalPages = Math.ceil(Total / Size)`. `start = (Page - 1) * Size`. `end = start + Size`.
*   **Array.slice():** Non-destructive subset extraction. The heart of the logic.
*   **Navigation:** Increment/decrement `currentPage`, recalculate `slice()`.
*   **Pipeline:** Search -> Filter -> Sort -> Paginate.
*   **Reactivity:** Reassigning `displayedRecords` to the result of `slice()` triggers UI rerender.
*   **Governor Limits:** Client-side pagination does not bypass SOQL or Heap limits.
*   **Dataset threshold:** Best for < 1,000-2,000 records. For larger sets, use Server-Side.
*   **Best Practices:** Always handle empty states, disable invalid navigation, and reset the page to 1 on search/sort.