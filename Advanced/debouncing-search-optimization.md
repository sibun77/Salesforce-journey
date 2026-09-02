# Debouncing – Search Optimization

# 1. Introduction

**Search optimization** in Salesforce Lightning Web Components (LWC) refers to the techniques used to ensure that user-initiated search queries perform efficiently, minimizing resource consumption on both the client (browser) and the server (Salesforce). 

Search inputs can create severe performance problems because **search-as-you-type** implementations fire an event for every single keystroke. Without optimization, typing an 8-letter word fires 8 separate Apex calls, leading to a sluggish UI, exhausted network connections, and governor limit breaches.

**Debouncing** is a programming practice used to ensure that time-consuming tasks do not fire too often. It forces a function to wait a certain amount of time before running. If the function is called again before the time expires, the timer resets.

Debouncing is crucial in LWC because every Apex call counts against Salesforce governor limits. By delaying the execution of the search until the user pauses typing, we drastically reduce excessive operations. 

Real-world **Automotive CRM** examples where debouncing improves application performance:
*   **Warranty Claim Search:** Finding historical claims by claim number or VIN.
*   **Work Order Search:** Technicians searching for open jobs.
*   **Dealer Search:** Locating dealerships by name or postal code.
*   **Vehicle Search:** Searching inventory by VIN or model.
*   **Invoice Search:** Looking up billing documents.
*   **Spare Parts Search:** Searching catalogs for components.
*   **Customer Search:** Lookups for CRM accounts.
*   **Shipment Search:** Tracking logistics and parts delivery.

---

# 2. What is Debouncing?

Debouncing guarantees that a function is only executed once after a specified period of inactivity.

**Without debounce:**  
If a user is searching for "Warranty", the `onchange` or `oninput` event triggers immediately on every keystroke.
*   W → Apex Call 1
*   Wa → Apex Call 2
*   War → Apex Call 3
*   Warr → Apex Call 4
*   Warranty → Apex Call 5

**With debounce:**  
The system waits for a pause (e.g., 300ms) before making the call.
*   W (timer starts)
*   Wa (timer resets)
*   War (timer resets)
*   Warr (timer resets)
*   Warranty (timer resets)
*        ↓ Wait 300ms
*   Warranty → Apex Call 1

The difference is staggering: 5 server round-trips vs. 1 server round-trip.

---

# 3. Why is Debouncing Needed?

Implementing search-as-you-type without debouncing causes cascading failures in an enterprise Salesforce application:

*   **Too many Apex calls:** Flooding the Salesforce server with requests.
*   **Too many HTTP/network requests:** Exhausting the browser's connection limits.
*   **Increased server load:** Processing queries for partial words ("Wa") that the user doesn't care about.
*   **Increased SOQL execution:** Burning through the 100 SOQL queries per transaction limit or concurrent Apex limits.
*   **Poor performance:** The UI thread stutters while processing rapid, overlapping asynchronous responses.
*   **Increased latency:** Queued HTTP requests slow down the entire application.
*   **Governor-limit consumption:** Rapid API calls can hit concurrent request limits.
*   **Unnecessary rendering:** The DOM rapidly updates with partial, useless results.
*   **Poor user experience:** The screen flashes, data jumps around, and the app feels unresponsive.

Debouncing solves this by batching the user's intent into a single, deliberate action.

---

# 4. How Debouncing Works

```text
User Input
    ↓
Input Event (e.g., 'W')
    ↓
Clear Previous Timer (if exists)
    ↓
Start New Timer (e.g., 300ms)
    ↓
Wait for Delay
    ↓
No New Input? (User stopped typing)
    ↓
Execute Search
    ↓
Apex / Local Search
    ↓
Display Results
```

**Step-by-step:** When an event fires, the code checks if a timeout is already running. If so, it destroys it. It then creates a brand new timeout. The search only fires when the timeout finally reaches 0 without being interrupted.

---

# 5. Basic Debounce Algorithm

1.  User enters a character in the search box.
2.  Input handler (`onchange`/`onkeyup`) executes.
3.  Existing timer (from previous keystroke) is cleared.
4.  New timer starts (e.g., 300ms).
5.  User enters another character before 300ms elapses.
6.  Previous timer is cleared (canceling the pending search).
7.  New timer starts for another 300ms.
8.  Process repeats as long as the user types quickly.
9.  When the user stops typing...
10. Timer completes its 300ms countdown.
11. Search executes (calls Apex).

---

# 6. JavaScript setTimeout()

`setTimeout()` is a native Web API method that sets a timer which executes a function or specified piece of code once the timer expires.

**Example:**
```javascript
this.searchTimeout = setTimeout(() => {
    this.performSearch();
}, 300);
```

*   **Callback:** The arrow function `() => { this.performSearch(); }` executed after the delay.
*   **Delay:** `300` milliseconds.
*   **Timer ID:** `setTimeout` returns a unique numeric ID (stored in `this.searchTimeout`), identifying the timer.
*   **Asynchronous execution:** `setTimeout` does not pause the browser; it pushes the callback to the Web API and continues executing subsequent code.
*   **Event loop:** Once the timer expires, the callback is pushed to the callback queue, and the event loop executes it when the call stack is empty.

---

# 7. JavaScript clearTimeout()

`clearTimeout()` cancels a timeout previously established by calling `setTimeout()`.

**Example:**
```javascript
clearTimeout(this.searchTimeout);
```

Clearing the previous timer is the **core of debouncing**. If you do not clear the timer, `setTimeout` will simply queue up multiple delayed functions. By clearing it, you ensure that only the *last* keystroke's timer survives.

---

# 8. Basic Debounce Example

```javascript
searchTimeout; // Property to hold the timer reference

handleSearch(event) {
    const searchKey = event.target.value;

    // Step 1: Clear the existing timer
    clearTimeout(this.searchTimeout);

    // Step 2: Set a new timer
    this.searchTimeout = setTimeout(() => {
        this.performSearch(searchKey); // Step 3: Execute after delay
    }, 300);
}
```

*   **Line 1:** Declares a class property to hold the timer ID.
*   **Line 4:** Grabs the current input value.
*   **Line 7:** Destroys any countdown currently in progress.
*   **Line 10-12:** Starts a fresh 300ms countdown that will eventually call `performSearch`.

---

# 9. Complete Basic LWC Debounce Example

**Requirement:** Search Warranty Claims. 300ms debounce.

**warrantySearch.html**
```html
<template>
    <lightning-card title="Warranty Claim Search">
        <div class="slds-p-around_medium">
            <lightning-input 
                type="search" 
                label="Search Claims by VIN" 
                onchange={handleSearch}>
            </lightning-input>

            <template if:true={isLoading}>
                <lightning-spinner alternative-text="Loading" size="small"></lightning-spinner>
            </template>

            <template if:true={error}>
                <p class="slds-text-color_error">{error}</p>
            </template>

            <template if:true={claims}>
                <!-- Display results -->
            </template>
            <template if:true={isEmpty}>
                <p>No claims found.</p>
            </template>
        </div>
    </lightning-card>
</template>
```

**warrantySearch.js**
```javascript
import { LightningElement } from 'lwc';
import searchClaims from '@salesforce/apex/WarrantyController.searchClaims';

export default class WarrantySearch extends LightningElement {
    searchTimeout;
    claims;
    error;
    isLoading = false;

    handleSearch(event) {
        const searchKey = event.target.value;
        
        // Clear previous timer
        clearTimeout(this.searchTimeout);

        // If input is empty, clear results and skip Apex
        if (!searchKey) {
            this.claims = null;
            return;
        }

        // Start new debounce timer
        this.searchTimeout = setTimeout(async () => {
            this.isLoading = true;
            try {
                this.claims = await searchClaims({ searchKey });
                this.error = undefined;
            } catch (error) {
                this.error = error.body.message;
                this.claims = undefined;
            } finally {
                this.isLoading = false;
            }
        }, 300);
    }

    get isEmpty() {
        return this.claims && this.claims.length === 0;
    }
}
```

*   **Important lines:** The `clearTimeout` prevents multiple calls. The `isLoading` state is set *inside* the timeout, so the spinner only appears when the actual network request begins, preventing UI flickering while typing.

---

# 10. Debounce Delay

Choosing the right delay is a balancing act between responsiveness and server load. There is no universal ideal delay.

| Delay | Response Speed | Number of Requests | User Experience | Recommended Scenarios |
| :--- | :--- | :--- | :--- | :--- |
| **200ms** | Very Fast | High | Snappy, but may fire mid-word for slow typists. | Local/Client-side array filtering. |
| **300ms** | Fast | Medium | Optimal balance. Feels immediate, stops most mid-word calls. | Standard server-side Apex search. |
| **500ms** | Noticeable delay | Low | User clearly sees a pause before search begins. | Heavy queries, complex SOQL. |
| **750ms** | Slow | Very Low | Sluggish UI. | External API callouts (e.g., SAP integrations). |
| **1000ms**| Very Slow | Minimal | Frustrating for users. | Background syncs, auto-saving drafts. |

---

# 11. Debouncing vs Throttling

| Feature | Debounce | Throttle |
| :--- | :--- | :--- |
| **Definition** | Wait until the user *stops* acting for X ms before firing. | Fire *at most once* every X ms, regardless of continuous action. |
| **Execution behavior** | Groups a rapid burst of events into a single execution at the end. | Ensures a steady, maximum rate of execution. |
| **Search use case** | **Ideal.** We only care about the final word typed. | Poor. Will fire periodically while typing, querying partial words. |
| **Scroll use case** | Poor. Won't update UI until user stops scrolling. | **Ideal.** Updates UI smoothly as the user scrolls. |
| **Resize use case** | Good for recalculating layouts once resizing is done. | Good for real-time responsive adjustments. |
| **LWC Examples** | `lightning-input` search. | `onscroll` for infinite loading. |

**Summary:** Debounce executes *after* the storm. Throttle executes *during* the storm, at a controlled rate.

---

# 12. Search Without Debouncing

```javascript
// BAD PRACTICE
handleSearch(event) {
    this.searchKey = event.target.value;
    this.performSearch(); // Calls Apex immediately
}
```
**Inefficiency:** Every keystroke ("V", "I", "N", "1", "2") initiates an independent server request. If a user types 10 characters in 2 seconds, 10 HTTP requests hit the Salesforce server simultaneously, wasting resources and risking limit exceptions.

---

# 13. Search With Debouncing

```javascript
// GOOD PRACTICE
handleSearch(event) {
    clearTimeout(this.searchTimeout);
    this.searchTimeout = setTimeout(() => {
        this.searchKey = event.target.value;
        this.performSearch(); 
    }, 300);
}
```
**Difference:**
*   **Apex Calls:** Drops from N (characters typed) to 1.
*   **Network traffic:** Reduced by ~90%.
*   **SOQL execution:** Only queries the final, meaningful search string.
*   **User experience:** No stuttering, UI updates cleanly once.

---

# 14. Client-Side Debouncing

```text
Input → Debounce → JavaScript → Filter Local Dataset → Display Results
```
**Example:**
```javascript
this.filteredRecords = this.allRecords.filter(record => 
    record.Name.toLowerCase().includes(searchKey.toLowerCase())
);
```
**When appropriate:** When the dataset is small (e.g., < 1,000 records) and already fully loaded into the browser memory. Debouncing here prevents UI thread locking during complex DOM repaints, even without server calls.

---

# 15. Server-Side Debouncing

```text
Input → Debounce → Apex → SOQL → Results
```
**Importance:** Server-side debouncing is mandatory for large Salesforce datasets (e.g., millions of Warranty Claims). We cannot load all claims to the client. We must debounce to prevent sending thousands of unselective SOQL queries to the database.

---

# 16. Debouncing with Imperative Apex

```javascript
handleSearch(event) {
    const searchKey = event.target.value;
    clearTimeout(this.searchTimeout);

    this.searchTimeout = setTimeout(async () => {
        try {
            const results = await searchRecords({ searchKey });
            this.data = results;
        } catch (error) {
            this.handleError(error);
        }
    }, 300);
}
```
*   **Input event:** Triggers the logic.
*   **Timer:** Delays the execution.
*   **Async function:** Used inside `setTimeout` to enable `await`.
*   **Apex call:** `searchRecords` returns a Promise.
*   **Error handling:** Wrapped in try/catch to manage Apex exceptions.

---

# 17. Debouncing with Wire Service

To debounce a wired method, you debounce the assignment of the reactive parameter.

```javascript
searchKey = '';

handleSearch(event) {
    const rawSearch = event.target.value;
    clearTimeout(this.searchTimeout);

    this.searchTimeout = setTimeout(() => {
        this.searchKey = rawSearch; // This triggers the wire!
    }, 300);
}

@wire(searchClaims, { searchKey: '$searchKey' })
wiredClaims({ data, error }) {
    if (data) { /* render data */ }
}
```
**Mechanism:** The `@wire` service automatically provisions data whenever a reactive property (prefixed with `$`) changes. By delaying the assignment of `this.searchKey`, we delay the wire service invocation.

---

# 18. Imperative Apex vs Wire Service for Debounced Search

| Feature | Imperative Apex | Wire Service |
| :--- | :--- | :--- |
| **Trigger** | Explicit `await` call. | Reactive property change. |
| **Search execution** | Controlled purely by JS logic. | Handled automatically by LWC framework. |
| **Caching** | Optional (`@AuraEnabled(cacheable=true)`). | Mandatory (`cacheable=true`). |
| **refreshApex** | N/A (unless data is cached). | Required if data changes on the server. |
| **Error handling** | Try/catch blocks. | `{ error }` object provisioning. |
| **State management** | Explicit `isLoading = true/false`. | Implicit (harder to show a clean spinner during provision). |
| **Flexibility** | High (easy to chain methods). | Medium (tied to component lifecycle). |
| **Recommended** | Complex searches, infinite scroll, DML updates. | Simple read-only searches with heavy caching. |

---

# 19. Search Input Validation

Before querying, validate the input to prevent useless database hits.

*   **Empty string:** Clear results, don't query.
*   **Whitespace:** `.trim()` the input.
*   **Minimum characters:** Require > 2 chars for large tables.
*   **Special characters:** Sanitize SOQL wildcards if necessary.

```javascript
if (searchKey.length > 0 && searchKey.length < 3) {
    return; // Do not search yet
}
```
Minimum character thresholds prevent non-selective queries (e.g., searching for all claims containing "A").

---

# 20. Empty Search Handling

When a user clears the input (`searchKey === ''`), what should happen?

1.  **Show all records:** Good for small datasets (e.g., list of 50 active users).
2.  **Show initial dataset:** Good for reverting to a "Recent Records" default view.
3.  **Clear results:** (Recommended for global search) Show a blank state or "Enter search term".
4.  **Reload first page:** Good when combined with standard pagination.

---

# 21. Minimum Search Length

```javascript
const searchKey = event.target.value.trim();

// Enforce minimum length of 3
if (searchKey.length > 0 && searchKey.length < 3) {
    this.showError('Please enter at least 3 characters.');
    return;
}
```
**Why?** A SOQL query like `LIKE '%a%'` forces a full table scan. `LIKE '%abc%'` is more selective and less likely to hit governor limits on large tables.

---

# 22. Search Input Normalization

```javascript
const searchKey = event.target.value.trim().toLowerCase();
```
*   **trim():** Removes leading/trailing spaces so `" claim "` becomes `"claim"`.
*   **lowercase:** Standardizes input (though SOQL is case-insensitive, JS filtering is not).
*   **whitespace normalization:** Removing double spaces improves matching accuracy.

---

# 23. Debouncing with SOQL

```apex
@AuraEnabled(cacheable=true)
public static List<Account> searchAccounts(String searchKey) {
    String searchPattern = '%' + String.escapeSingleQuotes(searchKey.trim()) + '%';
    
    return [
        SELECT Id, Name, AccountNumber 
        FROM Account 
        WHERE Name LIKE :searchPattern 
        LIMIT 50
    ];
}
```
*   **LIKE:** Allows partial matching.
*   **Wildcards (%):** `%term%` matches anywhere in the string.
*   **Performance implications:** Leading wildcards (`%term`) prevent the database from using standard indexes efficiently, leading to full table scans.

---

# 24. SOQL Search Optimization

Debouncing reduces *how often* queries run, but it does not make the query itself faster.
*   **Query only required fields:** Don't `SELECT *` (conceptually).
*   **LIMIT results:** Always cap searches (`LIMIT 50`).
*   **Selective filters:** Add other criteria (`Status = 'Active'`).
*   **Index considerations:** Search on indexed fields.
*   **Large data volumes:** For millions of records, SOSL is often better than SOQL `LIKE`.

---

# 25. Query Selectivity

A **selective query** returns a small portion of the total table records (typically < 10% for standard tables, stricter for LDV). 
*   The **Query Optimizer** relies on indexed fields. 
*   If your debounced search uses a non-indexed field, or a leading wildcard (`LIKE '%abc%'`), the query becomes **non-selective** and may fail with a `System.QueryException` on large tables.

---

# 26. Indexes and Search

*   **Standard Indexes:** `Id`, `Name`, `OwnerId`, Lookup/Master-Detail fields, Audit fields.
*   **Custom Indexes:** External IDs, Unique fields, or fields manually indexed by Salesforce Support.
*   **The Leading Wildcard Trap:** Even if `Warranty_Number__c` is indexed, `WHERE Warranty_Number__c LIKE '%123%'` **ignores the index**. 
*   **Solution:** If possible, use trailing wildcards (`LIKE '123%'`) to utilize the index.

---

# 27. Debouncing and Governor Limits

**Without debounce:** User types 10 characters → 10 Apex requests. If 20 users do this simultaneously, you hit the concurrent request limit or max out CPU time.
**With debounce:** User types 10 characters → 1 Apex request.

**Crucial:** Debouncing does NOT remove limits. That single Apex transaction still cannot return >50,000 records, take >10s of CPU time, or make >100 SOQL queries.

---

# 28. Debouncing and Search + Pagination

```text
Search → Debounce → Reset Pagination → Apex → First Page
```
**Important:** When a user changes their search term, the entire dataset changes. You **must** reset the `currentPageNumber` or `offsetSize` to 0. You cannot search for a new term and remain on "Page 5".

---

# 29. Debouncing and Infinite Scroll

```text
Search → Debounce → Reset cursor/page → Clear existing results → Load first batch
```
If you do not clear the existing results array and reset the cursor, a new search will append new, unrelated records to the bottom of the old search results.

---

# 30. Debouncing and Server-Side Pagination

When combined with server-side pagination, the debounced Apex call must pass the current search term *and* the pagination parameters.

```javascript
this.searchTimeout = setTimeout(async () => {
    this.pageNumber = 1; // Reset on new search
    const results = await getRecords({ 
        searchKey: this.searchKey, 
        pageSize: 50, 
        pageNumber: this.pageNumber 
    });
}, 300);
```

---

# 31. Debouncing and Sorting

```text
Search → Debounce → Reset Pagination → Apply Sort → Apex → Results
```
If a user changes the sort direction (e.g., Date ASC to Date DESC), you must reset pagination to the first page, otherwise, they will miss the actual "top" results.

---

# 32. Debouncing and Filtering

Changing a filter (e.g., Status = 'Closed') should instantly (or via its own debounce) trigger a new query. Just like search, applying a new filter invalidates the current data context, requiring a pagination reset.

---

# 33. Search + Filter + Sort + Pagination

**Architecture Flow:**
1.  **User Input:** Enters text.
2.  **Debounce:** Wait 300ms.
3.  **Search State Update:** Store valid search string.
4.  **Pagination:** Reset to Page 1.
5.  **Apex:** Pass (SearchKey, Filters, SortConfig, PaginationState).
6.  **SOQL:** Execute selective query.
7.  **Results:** Update LWC state.

Order matters. You cannot paginate before filtering, and you cannot sort without knowing the filtered dataset size.

---

# 34. Race Conditions

**Problem:** Asynchronous requests do not guarantee ordered returns.
1.  User types "Warranty" (Request A - takes 1000ms).
2.  User immediately types "Claim" (Request B - takes 200ms).
3. Request B finishes first. UI shows "Warranty Claim" results.
4. Request A finishes last. UI overwrites with older "Warranty" results!

This is a classic race condition resulting in a stale response.

---

# 35. Preventing Stale Search Results

**Solution:** Request Sequence Numbers.

```javascript
searchRequestId = 0; // Class property

handleSearch(event) {
    clearTimeout(this.searchTimeout);
    const searchKey = event.target.value;

    this.searchTimeout = setTimeout(async () => {
        // Increment and capture the ID for THIS specific request
        const currentRequestId = ++this.searchRequestId; 
        
        try {
            const result = await getClaims({ searchKey });
            
            // If the component's ID has advanced past this request's ID, ignore it
            if (currentRequestId !== this.searchRequestId) {
                return; // Stale response, discard
            }
            
            this.claims = result;
        } catch(e) {
             if (currentRequestId === this.searchRequestId) {
                 this.error = e;
             }
        }
    }, 300);
}
```

---

# 36. AbortController

In modern JS, `AbortController` can cancel native `fetch()` API requests. 
**Salesforce LWC Context:** The Salesforce `@wire` and Imperative Apex framework **does not currently support `AbortController`**. You cannot physically stop the server from processing the Apex call once it's fired.
**Alternative:** Use the Request ID pattern (Section 35) to ignore the response on the client side.

---

# 37. Loading State

```text
Search → Debounce → isLoading = true → Apex → Results → isLoading = false
```

```javascript
this.searchTimeout = setTimeout(async () => {
    this.isLoading = true; // Set inside timeout
    try {
        this.data = await doSearch({ searchKey });
    } finally {
        this.isLoading = false; // Always clear in finally block
    }
}, 300);
```
Setting `isLoading = true` *before* the timeout causes the spinner to flash rapidly on every keystroke. Setting it *inside* the timeout provides a smooth UX.

---

# 38. Search Button vs Search-As-You-Type

| Feature | Search-As-You-Type (Debounced) | Search Button |
| :--- | :--- | :--- |
| **Trigger** | `onchange` + timeout | `onclick` or Enter key |
| **UX** | Modern, seamless, fast | Traditional, explicit intent |
| **Server Load** | Moderate | Very Low |
| **Best For** | Finding records quickly (Accounts, Contacts) | Heavy, complex reports; slow legacy systems |

---

# 39. Debounce with lightning-input

```html
<lightning-input
    type="search"
    label="Search Claims"
    value={searchKey}
    onchange={handleSearch}>
</lightning-input>
```
*   **type="search":** Provides the "x" clear button.
*   **onchange:** In LWC `lightning-input`, `onchange` fires on every keystroke (unlike standard HTML inputs where it fires on blur). This is why debouncing is required here.

---

# 40. onchange vs input Event

*   **`onchange` (in standard HTML):** Fires when the input loses focus (blur).
*   **`oninput` (in standard HTML):** Fires on every keystroke.
*   **Salesforce `lightning-input`:** Modifies standard behavior. `onchange` behaves like `oninput`—firing on every keystroke. 
*   For search-as-you-type in LWC, use `onchange` + debounce.
*   For search-on-blur, use `onblur`.

---

# 41. Debounce Utility Function

Instead of repeating `setTimeout` in every component, create a utility using a **closure**.

```javascript
// debounceUtil.js
export function debounce(callback, delay = 300) {
    let timeoutId; // Persistent across calls due to closure

    return function (...args) {
        clearTimeout(timeoutId);

        timeoutId = setTimeout(() => {
            callback.apply(this, args); // Preserves LWC 'this' context
        }, delay);
    };
}
```
This utility maintains the `timeoutId` state privately and ensures the component's `this` context is preserved using `.apply()`.

---

# 42. Reusable Debounce Utility in LWC

Create a headless LWC or static resource module:

**LWC JS:**
```javascript
import { LightningElement } from 'lwc';
import { debounce } from 'c/utils'; // Assuming util is stored in c-utils

export default class SearchApp extends LightningElement {
    
    // Create the debounced version of the search logic once during construction
    debouncedSearch = debounce((searchKey) => {
        this.performServerSearch(searchKey);
    }, 300);

    handleInput(event) {
        this.debouncedSearch(event.target.value);
    }
    
    performServerSearch(key) { /* Apex logic */ }
}
```

---

# 43. Debounce in Parent and Child Components

*   **Option A: Parent handles debounce.** (Dumb child component). The child emits an event on every keystroke. The parent runs the debounce logic before calling Apex. 
*   **Option B: Child handles debounce.** (Smart search component). The child manages the timer and only emits `onsearch` when the timer expires. 
*   **Comparison:** Option B is far superior because it abstracts the complexity, makes the search bar reusable, and reduces unnecessary event bubbling in the DOM.

---

# 44. Reusable Search Component

Design a component that strictly handles input and debouncing, emitting a clean intent.

```html
<c-search-input
    value={searchKey}
    delay="300"
    onsearch={handleSearch}>
</c-search-input>
```
*   `@api delay`: Configurable timer.
*   `CustomEvent('search')`: Fired only *after* debounce completes, with `event.detail` containing the valid search term.

---

# 45. Complete Reusable Search Component

**searchInput.html**
```html
<template>
    <lightning-input 
        type="search" 
        label="Search" 
        value={value} 
        onchange={handleChange}>
    </lightning-input>
</template>
```

**searchInput.js**
```javascript
import { LightningElement, api } from 'lwc';

export default class SearchInput extends LightningElement {
    @api value = '';
    @api delay = 300;
    @api minLength = 2;
    
    timeoutId;

    handleChange(event) {
        const searchTerm = event.target.value.trim();
        clearTimeout(this.timeoutId);

        this.timeoutId = setTimeout(() => {
            if (searchTerm.length >= this.minLength || searchTerm.length === 0) {
                this.dispatchEvent(new CustomEvent('search', {
                    detail: { value: searchTerm }
                }));
            }
        }, this.delay);
    }
}
```

---

# 46. Debouncing and Lightning Data Service

LDS caches data on the client. If you query the exact same record twice, LDS serves it from cache, saving a server trip.
**However**, LDS does *not* solve debouncing for arbitrary text searches (`LIKE`). If you search "W", "Wa", "War", those are distinct queries to the server. Caching and debouncing solve entirely different problems and must be used together.

---

# 47. Debouncing and Platform Cache

*   **Debouncing:** Client-side optimization. Prevents the browser from *asking* the server too often.
*   **Platform Cache:** Server-side optimization. Allows Apex to answer the question quickly by checking RAM instead of querying the database.
*   **Complementary:** Debounce reduces the requests. Platform Cache makes the remaining requests lightning-fast.

---

# 48. Performance Comparison

User types: `"Warranty Claim"` (14 characters) in 2 seconds.

*   **Without debounce:** 14 server executions. Potential UI lockup. Database executes 14 SOQL queries.
*   **With 300ms debounce:** If typing is continuous, the timer resets on every character. Only **1 execution** occurs 300ms after the 'm' is typed. (If the user pauses midway, e.g., "Warranty... Claim", it might result in 2 requests—still a massive optimization).

---

# 49. Browser Event Loop

In JavaScript, `setTimeout` pushes a callback to the **Web API**. When the timer ends, it moves to the **Callback Queue**. The **Event Loop** constantly checks the **Call Stack**. When the stack is empty, it pushes the callback onto the stack to execute. 
If your LWC is busy rendering massive DOM elements, the event loop is blocked, and your 300ms debounce might physically take longer to trigger.

---

# 50. Timing Diagram

```text
User Typing:  W   a   r   r   a   n   t   y
Timer Action: |   |   |   |   |   |   |   |
              Start   |   |   |   |   |   |
                 Reset|   |   |   |   |   |
                     Reset|   |   |   |   |
                         Reset|   |   |   |
                             Reset|   |   |
                                 Reset|   |
                                     Reset|
                                         Reset
Wait 300ms:                               |-----> Execute Search
```

---

# 51. Debouncing and Throttling Real Examples

**Salesforce Automotive CRM:**

*   **Debounce (Wait to finish):**
    *   Dealer Lookup by Name
    *   Part Search by SKU
    *   Customer Search by Email
*   **Throttle (Limit rate during action):**
    *   Scrolling through a massive list of Warranty Claims (load next batch every 200ms of scrolling).
    *   Resizing a complex vehicle configuration canvas.

---

# 52. Debouncing and Large Data Volumes

When dealing with millions of records (e.g., Claim_Line_Item__c):
```text
Large Dataset
     ↓
Search Input
     ↓
Debounce (Protects from rapid requests)
     ↓
Selective SOQL (Protects from full table scans using indexes)
     ↓
Server-Side Pagination (Protects heap size)
     ↓
Limited Results
```
Debouncing is just step 1 in an LDV strategy.

---

# 53. Debouncing Does Not Replace Pagination

| Strategy | Problem Solved |
| :--- | :--- |
| **Debounce** | Controls *request frequency* (How often we ask). |
| **Pagination** | Controls *payload size* (How much data we get back). |

You can debounce a search that returns 10,000 records (crashing the browser). You can paginate a search without debouncing (crashing the server via request volume). You need both.

---

# 54. Debouncing Does Not Replace Query Optimization

*   **Debouncing** reduces how often queries are triggered.
*   **Query optimization** improves how efficiently each query executes. 
A debounced, poorly written query (`SELECT fields FROM Object WHERE NonIndexed LIKE '%a%'`) will still cause a CPU timeout or non-selective exception, just less frequently.

---

# 55. Debouncing Does Not Replace Caching

*   **Debouncing** prevents redundant requests while typing.
*   **Caching** (`@AuraEnabled(cacheable=true)`) prevents redundant requests when the user types a term, clears it, and types the *same term* again. Both are essential for high-performance apps.

---

# 56. Security

Search input is user input. It must be handled securely on the server.
*   **`with sharing`:** Enforce record-level visibility.
*   **`WITH USER_MODE`:** Enforce CRUD/FLS implicitly in SOQL.
*   **`Security.stripInaccessible()`:** Strip fields the user shouldn't see.
*   **SOQL Injection:** Use bind variables (`:searchKey`), never string concatenation.

---

# 57. SOQL Injection

**UNSAFE (Do not use):**
```apex
// Susceptible to injection if searchKey contains quotes
String query = 'SELECT Id FROM Account WHERE Name LIKE \'%' + searchKey + '%\'';
```

**SAFE (Use Bind Variables):**
```apex
// Bind variables automatically escape input
String searchPattern = '%' + searchKey + '%';
List<Account> accs = [SELECT Id FROM Account WHERE Name LIKE :searchPattern];
```

---

# 58. Error Handling

Implement a reusable catch block to display Apex errors gracefully.

```javascript
try {
    this.results = await searchData({ term });
} catch (error) {
    this.results = [];
    this.error = error.body ? error.body.message : 'An unknown error occurred.';
    // Log to a custom logger framework
}
```

---

# 59. Empty State

Provide user-friendly UI for all states:
*   **No search entered:** Show a placeholder or recent items.
*   **Search too short:** "Type at least 3 characters."
*   **No results:** "No warranties found for '[Term]'."
*   **Error:** "Failed to load results. Try again."
*   **Loading:** Skeleton loaders or `lightning-spinner`.

---

# 60. Retry Behavior

If a debounced search fails (e.g., network timeout), how do they retry?
*   Do not force them to delete and re-type characters.
*   Provide a "Retry" button that calls the Apex method imperatively using the existing `this.searchKey` state variable.

---

# 61. Common Mistakes

1.  **Not using clearTimeout():** Causes all typed partial words to eventually execute. **Fix:** Always clear before setting.
2.  **Creating multiple timers:** By not persisting the timeout ID at the class level. **Fix:** Use `this.timeoutId`.
3.  **Excessively short delay (<100ms):** Defeats the purpose.
4.  **Excessively long delay (>1000ms):** App feels broken.
5.  **Calling Apex on every keystroke:** Hits governor limits.
6.  **Not trimming input:** Searches for "  text  " fail.
7.  **Not handling empty input:** Queries all records accidentally.
8.  **Not using min length:** Queries crash due to non-selectivity.
9.  **Not resetting pagination:** User sees "Page 5" of a new 1-page result.
10. **Not resetting cursor:** Infinite scroll appends unrelated data.
11. **Not handling stale responses:** UI flickers between old/new data.
12. **Not handling errors:** Silent failures confuse users.
13. **Not showing loading state:** User keeps typing aggressively, thinking it's broken.
14. **Loading too many records:** Missing `LIMIT`.
15. **Non-selective SOQL:** Leading wildcards on huge tables.
16. **Unsafe dynamic SOQL:** SOQL injection risks.
17. **Not handling sort/filter changes:** Results become misaligned.
18. **Confusing debounce with throttle.**
19. **Assuming debounce solves governor limits.**
20. **Assuming debounce replaces query optimization.**
21. **Assuming debounce replaces caching.**
22. **Ignoring accessibility:** Missing `aria-live` regions for status updates.

---

# 62. Debugging Debounced Search

**Checklist:**
*   **Search triggering multiple times:** Is `timeoutId` declared at the class level? Are you calling `clearTimeout` properly?
*   **Search not triggering:** Is the delay too high? Did you bind `this` correctly in the callback?
*   **Old results replace new:** Implement Request Sequence IDs.
*   **Pagination broken:** Ensure `pageNumber = 1` occurs inside the search execution block.
*   **Spinner stuck:** Ensure `isLoading = false` is in a `finally` block.

---

# 63. Testing Debounced Search

When writing Jest tests, `setTimeout` makes testing difficult because tests run synchronously. You must test:
*   Search is not called immediately.
*   Search is called after the exact timer expires.
*   Multiple inputs clear the previous timer.

**Jest provides Mock Timers** to fast-forward time.

---

# 64. Jest Testing for Debounce

```javascript
import { createElement } from 'lwc';
import SearchApp from 'c/searchApp';

// Enable fake timers
jest.useFakeTimers();

describe('c-search-app debounce logic', () => {
    it('debounces the search input', async () => {
        const element = createElement('c-search-app', { is: SearchApp });
        document.body.appendChild(element);

        const input = element.shadowRoot.querySelector('lightning-input');
        
        // Simulate rapid typing
        input.value = 'W';
        input.dispatchEvent(new CustomEvent('change'));
        input.value = 'Wa';
        input.dispatchEvent(new CustomEvent('change'));
        
        // Fast-forward time by 299ms (less than 300ms delay)
        jest.advanceTimersByTime(299);
        // Assert Apex was NOT called yet

        // Fast-forward to cross the 300ms threshold
        jest.advanceTimersByTime(1);
        // Assert Apex WAS called exactly once with 'Wa'
    });
});
```

---

# 65. Complete Enterprise Example

**Warranty Claim Search Requirements:** 300ms debounce, min 2 chars, Imperative Apex, User Mode, Pagination Reset, Stale-response protection.

**WarrantyController.cls**
```apex
public with sharing class WarrantyController {
    @AuraEnabled
    public static List<Warranty_Claim__c> searchClaims(String searchTerm) {
        String match = '%' + String.escapeSingleQuotes(searchTerm.trim()) + '%';
        return [
            SELECT Id, Claim_Number__c, Status__c, Vehicle_VIN__c 
            FROM Warranty_Claim__c 
            WHERE Claim_Number__c LIKE :match 
            WITH USER_MODE
            LIMIT 50
        ];
    }
}
```

**warrantySearchLwc.js**
```javascript
import { LightningElement } from 'lwc';
import searchClaims from '@salesforce/apex/WarrantyController.searchClaims';

export default class WarrantySearchLwc extends LightningElement {
    searchTimeout;
    searchRequestId = 0;
    claims = [];
    isLoading = false;
    error;

    columns = [
        { label: 'Claim Number', fieldName: 'Claim_Number__c' },
        { label: 'Status', fieldName: 'Status__c' },
        { label: 'VIN', fieldName: 'Vehicle_VIN__c' }
    ];

    handleSearch(event) {
        clearTimeout(this.searchTimeout);
        const searchKey = event.target.value.trim();

        if (searchKey.length > 0 && searchKey.length < 2) {
            return; // Enforce minimum length
        }

        if (searchKey.length === 0) {
            this.claims = []; // Clear state
            return;
        }

        this.searchTimeout = setTimeout(async () => {
            const currentReqId = ++this.searchRequestId;
            this.isLoading = true;
            this.error = undefined;

            try {
                const results = await searchClaims({ searchTerm: searchKey });
                if (currentReqId === this.searchRequestId) {
                    this.claims = results;
                }
            } catch (err) {
                if (currentReqId === this.searchRequestId) {
                    this.error = err.body.message;
                    this.claims = [];
                }
            } finally {
                if (currentReqId === this.searchRequestId) {
                    this.isLoading = false;
                }
            }
        }, 300);
    }
}
```

---

# 66. Complete Search Architecture

```text
User
 ↓ (Types "VIN-998")
lightning-input
 ↓ (Fires onchange)
Input Handler
 ↓ (clearTimeout / setTimeout)
Debounce
 ↓ (Waits 300ms)
Normalize Search Key (.trim())
 ↓ (Check > 2 chars)
Validate Minimum Length
 ↓ (pageNumber = 1)
Reset Pagination
 ↓ (Call imperative method)
Apex
 ↓ (Bind variables)
Secure SOQL (WITH USER_MODE)
 ↓ (WHERE clause)
Search + Filter + Sort
 ↓ (LIMIT / OFFSET)
Pagination
 ↓ (Return JSON)
Response
 ↓ (Check Request ID)
Stale Response Check
 ↓ (this.claims = data)
LWC State
 ↓ (Re-render)
Datatable
```

---

# 67. Client-Side vs Server-Side Search

| Feature | Client-Side Search | Server-Side Search |
| :--- | :--- | :--- |
| **Dataset size** | Small (< 1,000 records) | Large / Infinite |
| **Network requests** | 1 (Initial load only) | 1 per debounced search |
| **Browser memory** | High (Stores full dataset) | Low (Stores only visible page) |
| **SOQL** | Executed once | Executed on every search |
| **Performance** | Instantaneous filtering | Subject to network latency |
| **Scalability** | Poor (Breaks on large data) | Excellent (Highly scalable) |
| **Recommended** | Picklists, small config tables | CRM Data (Accounts, Claims) |

---

# 68. Debounce vs Pagination vs Infinite Scroll

| Concept | Controls | Works Together? |
| :--- | :--- | :--- |
| **Debounce** | Request frequency | Wait 300ms before asking for Page 1. |
| **Pagination** | Records per page (Offset) | Returns 50 records at a time. |
| **Infinite Scroll** | How additional data loads | Scrolling down fetches Page 2 using the same debounced search term. |

---

# 69. Debounce vs Caching vs Query Optimization

| Technique | Problem Solved | Example |
| :--- | :--- | :--- |
| **Debouncing** | Too many requests while typing. | `setTimeout()` in LWC. |
| **Caching** | Redundant requests for identical data. | `@AuraEnabled(cacheable=true)` |
| **Query Optimization**| Slow server execution time. | Adding Custom Index, `LIMIT`. |

---

# 70. Real Project Scenarios (Automotive CRM)

1.  **Warranty Claim Search:** LDV (Millions). Requires Server-side debounce (300ms), min 3 chars, server-side pagination, strict indexed SOQL.
2.  **Dealer Lookup:** Small volume (Thousands). Can use Client-side debounce (150ms), load all active dealers on init, filter locally.
3.  **Vehicle Search:** LDV. Server-side search by 17-char VIN. Debounce 500ms to allow pasting or scanning via barcode.
4.  **Work Order Search:** Medium volume. Server-side debounce with infinite scroll.
5.  **Spare Part Search:** LDV. Use SOSL instead of SOQL if searching across part name and description.
6.  **Invoice Search:** Highly secure. Server-side debounce, enforce `WITH USER_MODE`.
7.  **Customer Search:** Server-side debounce. Sort by Recent.
8.  **Shipment Search:** API Callout to SAP. High debounce (750ms) to prevent external API rate limiting.

---

# 71. Decision Guide

```text
Need search?
     ↓
How large is dataset?
     ↓
Small (< 2000) ----------------------> Client-side search (Filter in JS)
     ↓
Large (Millions)
     ↓
Server-side search
     ↓
Does search trigger frequently (typing)?
     ↓
YES ---------------------------------> Implement Debounce (300ms)
     ↓
Need many results?
     ↓
Server-side pagination / Infinite Scroll
     ↓
Need high-volume/deep pagination?
     ↓
Consider Keyset Pagination (Avoid OFFSET > 2000)
```

---

# 72. Best Practices Checklist

*   ✅ **Use debounce for search-as-you-type:** Never hit Apex on every keystroke.
*   ✅ **Use clearTimeout():** The foundation of debouncing.
*   ✅ **Choose a reasonable delay:** 300ms is standard.
*   ✅ **Trim search input:** Eliminate whitespace errors.
*   ✅ **Consider minimum search length:** Protect DB from non-selective queries.
*   ✅ **Handle empty search:** Clear arrays when input is blank.
*   ✅ **Prevent stale responses:** Use sequence IDs to ignore old requests.
*   ✅ **Show loading state:** `isLoading` keeps UX smooth.
*   ✅ **Handle errors:** Catch exceptions and show toasts.
*   ✅ **Reset pagination/cursor:** Always reset to Page 1 on new searches.
*   ✅ **Debounce server-side searches:** Mandatory for LDV.
*   ✅ **Use selective SOQL:** Leverage indexes.
*   ✅ **Use LIMIT:** Never query without bounds.
*   ✅ **Avoid unsafe dynamic SOQL:** Prevent injection via bindings.
*   ✅ **Respect CRUD/FLS:** Use `WITH USER_MODE`.
*   ✅ **Use Jest:** Mock timers to test debounce logic.

---

# 73. Interview Questions & Answers

### Beginner Questions
**Q: What is debouncing?**
A: A technique to delay function execution until a certain amount of time has passed since the last time the event fired.

**Q: What is `setTimeout()`?**
A: A JS method that executes a block of code after a specified delay.

### Intermediate Questions
**Q: What is the difference between debounce and throttle?**
A: Debounce waits for the user to stop the action entirely. Throttle guarantees execution at a regular interval *while* the action is happening. Search needs debounce; scrolling needs throttle.

**Q: How do you debounce a wired Apex call?**
A: You don't debounce the `@wire` directly. You debounce the updating of the reactive property (e.g., `$searchKey`).

### Advanced Questions
**Q: Why must pagination reset after a search change?**
A: Changing the search alters the underlying dataset. If you are on page 3 of "Ford", and search for "Toyota", page 3 of the new dataset might not exist, or will show incorrect offset records.

**Q: How do you prevent stale search results (race conditions)?**
A: By implementing a request counter. Increment the counter when a timeout fires. When the Apex promise resolves, check if the counter matches. If it doesn't, discard the payload.

### Architect-Level Questions
**Q: How would you optimize search for millions of Salesforce records?**
A: I would combine multiple strategies: 
1. Implement a 300-500ms debounce in LWC. 
2. Enforce a minimum of 3 characters. 
3. Use a custom index on the search field. 
4. Avoid leading wildcards to ensure SOQL selectivity (or switch to SOSL). 
5. Implement server-side keyset pagination instead of OFFSET for deep data traversal. 
6. Ensure request sequencing prevents stale responses on the client.

---

# 74. Revision Summary

*   **Debouncing** controls request frequency using `setTimeout()` and `clearTimeout()`.
*   **Search-as-you-Type** requires debouncing to protect limits.
*   **Delay** of ~300ms is standard.
*   **Client vs Server:** Filter small data locally; debounce large data server-side.
*   **LWC:** Debounce imperative Apex via `async/await`. Debounce `@wire` by delaying reactive property assignment.
*   **SOQL:** Use `LIKE` with bind variables. Beware of leading wildcards causing full table scans.
*   **Limits:** Debouncing reduces calls but doesn't fix bad queries. Combine with `LIMIT`, filters, and indexes.
*   **State Management:** Always reset pagination/cursors when search changes. Use Request IDs to fix stale responses.
*   **Security:** Enforce `WITH USER_MODE` and protect against SOQL injection. 
*   **Testing:** Use `jest.useFakeTimers()` to test debounce logic.