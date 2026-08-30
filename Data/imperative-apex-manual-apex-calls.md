# Imperative Apex – Manual Apex Calls

# 1. Introduction

In Salesforce Lightning Web Components (LWC), **Imperative Apex** refers to the manual execution of Apex methods using JavaScript. Unlike the Wire Service, which automatically provisions data based on reactive properties, Imperative Apex puts the developer in complete control of *when* and *how* the server is called.

LWC communicates with Apex via secure JSON-based HTTP requests. Manual control is essential when operations depend on user actions (like a button click) rather than component initialization. 

**When to prefer Imperative Apex:**
*   You need to execute **DML operations** (Create Warranty Claim, Update Work Order, Delete Spare Part).
*   You need to invoke an external **HTTP Callout** (Send data to SAP).
*   You need to execute **Search functionality** (Search Claims) on demand.
*   The business logic requires conditional server execution (Submit Invoice only after client-side validation).

---

# 2. What is Imperative Apex?

Imperative Apex is the manual invocation of a server-side Apex method from client-side LWC JavaScript using Promises. It bypasses the reactive data provisioning of the Wire service, allowing you to explicitly call the server exactly when needed.

**LWC to Apex Data Flow:**

```text
User Action (e.g., clicks "Update Claim")
     ↓
LWC JavaScript (handles event, validates data)
     ↓
Imperative Apex Call (invokes imported method)
     ↓
Apex Method (receives parameters)
     ↓
SOQL / DML / Business Logic (executes on Salesforce server)
     ↓
Response (returns data or throws exception)
     ↓
LWC (Promise resolves or rejects)
     ↓
UI Update (shows Toast, updates variables, stops spinner)
```

---

# 3. Why Use Imperative Apex?

Developers use Imperative Apex primarily for **explicit control**:

*   **User-Triggered Operations:** Clicking a "Submit Approval" button.
*   **DML Operations:** You cannot perform DML inside `@AuraEnabled(cacheable=true)` methods; you must use Imperative Apex.
*   **Conditional Execution:** Only calling the server if certain UI criteria are met.
*   **Complex Business Logic:** Processing forms with multi-step validations before saving.
*   **External Callouts:** Triggering an API request to SAP to fetch real-time Shipment Details.

---

# 4. Imperative Apex vs Wire Service

| Feature | Imperative Apex | Wire Service (`@wire`) |
| :--- | :--- | :--- |
| **Execution** | Manual / On-Demand | Automatic / Reactive |
| **Trigger** | JavaScript events, hooks, or functions | Component load or reactive property change |
| **Reactivity** | Not reactive by default | Highly reactive (`$property`) |
| **DML Support** | Yes (Create, Update, Delete) | No (Read-Only) |
| **Caching** | Depends (can call cacheable, mostly non-cacheable) | Required (`cacheable=true`) |
| **Best For** | DML, user actions, external callouts, conditional logic | Fetching initial data, reading records |
| **Error Handling** | explicit `catch` or `try/catch` blocks | Handled via `error` property in wired result |

**Rule of Thumb:** Use Wire Service for reactive read operations. Use Imperative Apex when you need explicit control over execution, DML, or complex procedural logic.

---

# 5. Basic Imperative Apex Syntax

To use Imperative Apex, you must first import the Apex method into your JavaScript file.

```javascript
// Import the Apex method via the @salesforce/apex module
import getAccounts from '@salesforce/apex/AccountController.getAccounts';
```

Then, you invoke it as a function that returns a Promise:

```javascript
getAccounts()
    .then(result => {
        // Executed if the server call is successful
        this.accounts = result;
    })
    .catch(error => {
        // Executed if the server throws an exception
        this.error = error;
    });
```

*   `getAccounts()`: Invokes the Apex method.
*   `.then()`: Handles the successful response (`result`).
*   `.catch()`: Handles any errors returned from the server (`error`).

---

# 6. Apex Method Requirements

For LWC to call an Apex method, the method must meet specific criteria.

```java
public with sharing class AccountController {
    @AuraEnabled
    public static List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account LIMIT 50];
    }
}
```

*   **`@AuraEnabled`:** Exposes the method to Lightning components.
*   **`public` or `global`:** The method must be accessible.
*   **`static`:** LWC does not instantiate the Apex class; it calls methods statically.
*   **Return Type:** Must be a supported data type (primitives, SObjects, Collections, Wrapper classes).

---

# 7. @AuraEnabled

`@AuraEnabled` is an annotation that acts as a bridge between the Salesforce server and the Lightning component framework (both Aura and LWC). 
*   **Serialization:** It automatically handles the JSON serialization and deserialization of data passing between JavaScript and Apex.
*   **Parameters:** Maps JavaScript object keys to Apex method parameter names.

---

# 8. @AuraEnabled(cacheable=true)

```java
@AuraEnabled(cacheable=true)
```
*   **Meaning:** Marks the method as read-only and allows the results to be cached on the client side.
*   **Wire Service:** Mandatory for methods used with `@wire`.
*   **Imperative Apex:** You *can* call cacheable methods imperatively. However, if the data is cached, the server won't be hit again, yielding faster performance but potentially stale data.
*   **DML Restriction:** You **cannot** perform DML (Insert/Update/Delete) in a cacheable method. Doing so throws a runtime exception.

---

# 9. Promise-Based Apex Calls

Imperative Apex returns a standard JavaScript **Promise**.

```javascript
getAccounts()
    .then(result => {
        this.accounts = result; // Promise resolved
    })
    .catch(error => {
        this.error = error; // Promise rejected
    })
    .finally(() => {
        this.isLoading = false; // Always executes
    });
```
*   **`then()`:** Executes when the Promise resolves (success).
*   **`catch()`:** Executes when the Promise rejects (error/exception).
*   **`finally()`:** Executes regardless of outcome; perfect for turning off loading spinners.

---

# 10. async/await

Modern JavaScript uses `async`/`await` for cleaner, more readable asynchronous code, avoiding "Promise chains".

```javascript
async loadAccounts() {
    try {
        this.isLoading = true;
        // Code execution pauses here until the Promise resolves
        this.accounts = await getAccounts(); 
    } catch (error) {
        // Catches any rejected Promise
        this.error = error;
    } finally {
        // Always executes
        this.isLoading = false;
    }
}
```
*   **`async`:** Marks the function as asynchronous.
*   **`await`:** Pauses execution until the Apex call completes.
*   **`try/catch/finally`:** The synchronous-looking way to handle Promise states.

---

# 11. Passing Parameters to Apex

Parameters are passed from LWC to Apex as a JSON object. **The keys in the JS object must exactly match the Apex parameter names.**

**Apex:**
```java
@AuraEnabled
public static List<Account> getAccounts(String searchKey) { ... }
```

**LWC:**
```javascript
getAccounts({ searchKey: this.searchKey });
```

---

# 12. Multiple Parameters

Pass multiple parameters by adding more keys to the JSON object.

**Apex:**
```java
@AuraEnabled
public static List<Claim__c> getClaims(Id dealerId, String status, String searchKey) { ... }
```

**LWC:**
```javascript
getClaims({
    dealerId: this.dealerId,
    status: this.status,
    searchKey: this.searchKey
});
```
Every key maps precisely to the Apex method signature. Order does not matter; spelling/casing does.

---

# 13. Passing Lists to Apex

You can pass JavaScript arrays to Apex `List` collections.

**LWC:**
```javascript
const selectedIds = ['a01...', 'a02...'];
getClaimsByIds({ claimIds: selectedIds });
```

**Apex:**
```java
@AuraEnabled
public static List<Claim__c> getClaimsByIds(List<Id> claimIds) { ... }
```
The framework automatically deserializes the JS array into the Apex `List<Id>`.

---

# 14. Passing Maps to Apex

Passing Maps requires mapping JS objects to Apex `Map<String, Object>`.

**LWC:**
```javascript
const filterCriteria = {
    Type: 'Warranty',
    Status: 'Pending',
    Amount: 500
};
processFilters({ filters: filterCriteria });
```

**Apex:**
```java
@AuraEnabled
public static void processFilters(Map<String, Object> filters) { 
    String type = (String)filters.get('Type');
}
```
*Limitation:* Maps can be difficult to type-cast safely in Apex. Using Wrapper classes is usually a more robust pattern.

---

# 15. Wrapper Classes

Wrapper classes provide strict typing and a clean contract between LWC and Apex.

**Apex Wrapper:**
```java
public class ClaimRequest {
    @AuraEnabled public String claimNumber;
    @AuraEnabled public Id vehicleId;
    @AuraEnabled public Decimal amount;
}
```

**Apex Method:**
```java
@AuraEnabled
public static void submitClaim(ClaimRequest request) { ... }
```

**LWC:**
```javascript
const req = {
    claimNumber: 'CLM-001',
    vehicleId: 'a03...',
    amount: 1500.00
};
submitClaim({ request: req });
```

---

# 16. Returning Data from Apex

Apex can return almost anything to LWC:
*   **Primitives:** `String`, `Integer`, `Decimal`, `Boolean`
*   **SObjects:** `Account`, `Claim__c`
*   **Collections:** `List<Contact>`, `Map<String, Integer>`
*   **Wrappers:** Custom classes instantiated and populated in Apex.

*Example returning a List of Wrappers:*
```java
@AuraEnabled
public static List<DealerWrapper> getDealers() { ... }
```

---

# 17. Querying Records with Imperative Apex

**Scenario:** Fetch Warranty Claims on button click.

**Apex:**
```java
@AuraEnabled
public static List<Claim__c> getClaims() {
    return [SELECT Id, Name, Status__c FROM Claim__c LIMIT 10];
}
```

**LWC JS:**
```javascript
import getClaims from '@salesforce/apex/ClaimController.getClaims';

async handleFetch() {
    try {
        this.claims = await getClaims();
    } catch (error) {
        console.error(error);
    }
}
```

**LWC HTML:**
```html
<lightning-button label="Load Claims" onclick={handleFetch}></lightning-button>
<lightning-datatable key-field="Id" data={claims} columns={columns}></lightning-datatable>
```

---

# 18. Creating Records

**Scenario:** Create Warranty Claim explicitly via Apex.

**Apex:**
```java
@AuraEnabled
public static Id createClaim(String title, Id vehicleId) {
    Claim__c newClaim = new Claim__c(Name = title, Vehicle__c = vehicleId);
    insert newClaim;
    return newClaim.Id;
}
```

**LWC:**
```javascript
async handleSave() {
    try {
        const recordId = await createClaim({ 
            title: this.claimTitle, 
            vehicleId: this.vehicleId 
        });
        this.showToast('Success', 'Claim Created: ' + recordId, 'success');
    } catch (error) {
        this.showToast('Error', error.body.message, 'error');
    }
}
```

---

# 19. Updating Records

**Scenario:** Update Claim Status.

**Apex:**
```java
@AuraEnabled
public static void updateClaimStatus(Id claimId, String newStatus) {
    update new Claim__c(Id = claimId, Status__c = newStatus);
}
```

**LWC:**
```javascript
async handleApprove() {
    try {
        await updateClaimStatus({ claimId: this.recordId, newStatus: 'Approved' });
        this.showToast('Success', 'Claim Approved', 'success');
    } catch (error) {
        this.showToast('Error', error.body.message, 'error');
    }
}
```

---

# 20. Deleting Records

**Scenario:** Delete a Spare Part Line.

**Apex:**
```java
@AuraEnabled
public static void deleteSparePart(Id partId) {
    delete new Spare_Part__c(Id = partId);
}
```

**LWC:**
```javascript
async handleDelete() {
    if (confirm('Are you sure you want to delete this part?')) {
        try {
            await deleteSparePart({ partId: this.partId });
            this.showToast('Deleted', 'Part removed', 'success');
            // Logic to refresh UI
        } catch (error) {
            this.showToast('Error', error.body.message, 'error');
        }
    }
}
```

---

# 21. Search Using Imperative Apex

**Scenario:** Search Warranty Claims by Claim Number manually.
Imperative Apex is best here because we don't want the search firing automatically on every keystroke (which Wire would do); we want it to fire only when the user clicks "Search".

**Apex:**
```java
@AuraEnabled
public static List<Claim__c> searchClaims(String searchTerm) {
    String queryTerm = '%' + searchTerm + '%';
    return [SELECT Id, Name FROM Claim__c WHERE Name LIKE :queryTerm];
}
```

**LWC JS:**
```javascript
async executeSearch() {
    this.results = await searchClaims({ searchTerm: this.searchInput });
}
```

---

# 22. Imperative Apex with Forms

When using base components like `lightning-input` or `lightning-combobox`, you gather the values, construct a request object, and imperatively send it to Apex.

```javascript
handleSubmit() {
    const claimData = {
        amount: this.template.querySelector('.amount-input').value,
        description: this.template.querySelector('.desc-input').value
    };
    
    saveClaimData({ request: claimData })
        .then(() => this.resetForm())
        .catch(err => this.handleError(err));
}
```

---

# 23. Form Validation Before Apex

Never call the server if client-side validation fails.

```javascript
validateInputs() {
    const allValid = [...this.template.querySelectorAll('lightning-input')]
        .reduce((validSoFar, inputCmp) => {
            inputCmp.reportValidity();
            return validSoFar && inputCmp.checkValidity();
        }, true);
        
    if (!allValid) {
        this.showToast('Error', 'Please fix form errors', 'error');
        return false;
    }
    return true;
}

async handleSave() {
    if (!this.validateInputs()) return; // Stop if invalid
    // Proceed to Apex call
}
```

---

# 24. Loading State

Manage UX by using an `isLoading` boolean.

```javascript
this.isLoading = true; // Show Spinner
try {
    await saveClaim(data);
} finally {
    this.isLoading = false; // Hide Spinner always
}
```
Bind `isLoading` in HTML to `<lightning-spinner if:true={isLoading}></lightning-spinner>`.

---

# 25. Preventing Duplicate Apex Calls

If a user double-clicks a button, it can cause duplicate records or lock errors.

```javascript
async handleSave() {
    if (this.isLoading) return; // Prevent double execution
    
    this.isLoading = true;
    try {
        await submitInvoice();
    } catch (e) {
        // Handle
    } finally {
        this.isLoading = false;
    }
}
```
Disabling the submit button via `<lightning-button disabled={isLoading}>` is also a critical best practice.

---

# 26. Error Handling

Apex errors include `DMLException`, `NullPointerException`, and Custom Exceptions. When these occur in an `@AuraEnabled` method, they are sent to LWC as an object.

Structure of a typical error object in LWC:
*   `error.body.message` (Primary error message)
*   `error.statusText`
*   `error.body.exceptionType`

---

# 27. Error Handling in LWC

Reusable error handler:

```javascript
export function getErrorMessage(error) {
    if (error && error.body && error.body.message) {
        return error.body.message; // Standard Apex Error
    } else if (error && error.message) {
        return error.message; // JS Error
    }
    return 'An unexpected error occurred.';
}
```

---

# 28. Apex Exception Handling

Do not let raw system exceptions propagate to the client. Wrap them in an `AuraHandledException`.

```java
@AuraEnabled
public static void processData() {
    try {
        // risk of DMLException
        insert data; 
    } catch (Exception e) {
        // Logs internally, shows clean message to user
        System.debug('Error: ' + e.getMessage()); 
        throw new AuraHandledException('Unable to save data. Please contact support.');
    }
}
```

---

# 29. Imperative Apex with Toast Messages

Combine Apex states with UI feedback.

```javascript
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

async save() {
    try {
        await saveRecord();
        this.dispatchEvent(
            new ShowToastEvent({ title: 'Success', message: 'Saved', variant: 'success' })
        );
        this.closeModal();
    } catch (error) {
        this.dispatchEvent(
            new ShowToastEvent({ title: 'Error', message: error.body.message, variant: 'error' })
        );
        // Form stays open for correction
    }
}
```

---

# 30. Refreshing Data After Imperative Apex

When you update data via Imperative Apex, the LWC cache (or Lightning Data Service cache) does not automatically know about the change.
*   **`refreshApex`:** Used to refresh data provisioned via the `@wire` service to an Apex method.
*   **`notifyRecordUpdateAvailable`:** Tells the Lightning Data Service (LDS) that a specific record ID has changed, updating standard UI components and `@wire(getRecord)` calls.

---

# 31. refreshApex with Imperative Apex

**Pattern:** Read via Wire, Update via Imperative, Refresh via `refreshApex`.

```javascript
import { refreshApex } from '@salesforce/apex';

@wire(getClaims) wiredClaims; // wiredClaims holds the provisioned object

async handleUpdate() {
    await updateClaim();
    // Force the wire to fetch fresh data
    await refreshApex(this.wiredClaims); 
}
```

---

# 32. notifyRecordUpdateAvailable

Use this to notify standard components (like record pages) that your imperative apex changed a record.

```javascript
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';

async handleStatusChange() {
    await updateStatusInApex({ recordId: this.recordId });
    // Tell LDS to update its cache for this specific record
    await notifyRecordUpdateAvailable([{ recordId: this.recordId }]);
}
```

---

# 33. Imperative Apex and Lightning Data Service

LDS and Apex operate differently:
*   **UI API (LDS):** Client-side cache aware, respects FLS natively, no Apex code required. Best for single-record CRUD.
*   **Imperative Apex:** Best for multi-record operations, complex transactions, server-side callouts.

When mixing them, rely on `notifyRecordUpdateAvailable` to keep the LDS cache in sync with your Apex DML.

---

# 34. Imperative Apex and Callouts

**Scenario:** Send Warranty Claim to SAP.

**Apex:**
```java
@AuraEnabled
public static String sendToSAP(Id claimId) {
    HttpRequest req = new HttpRequest();
    req.setEndpoint('callout:SAP_Credentials/api/claims/' + claimId);
    req.setMethod('POST');
    HttpResponse res = new Http().send(req);
    return res.getBody();
}
```

**LWC:** Calls `sendToSAP`, waits for the response, and shows a toast. The callout happens synchronously on the server, but asynchronously in the JS Promise.

---

# 35. Imperative Apex and Transactions

A single Imperative Apex call constitutes one Database Transaction.
*   If an exception is thrown and not caught, the entire transaction rolls back.
*   If you catch the exception in Apex and throw an `AuraHandledException`, the transaction *still* rolls back automatically.

---

# 36. Governor Limits

One imperative call equals one Apex transaction. All standard limits apply:
*   100 SOQL Queries
*   150 DML Statements
*   10-second CPU time
*   100 Callouts

If a user rapidly clicks a button making 10 Imperative calls, they initiate 10 independent transactions.

---

# 37. Bulkification

LWC components should be designed to send bulk data to Apex rather than iterating client-side.

**Bad (LWC Loop):**
```javascript
// Creates multiple transactions, risks limits and race conditions
this.rows.forEach(row => updateRecordInApex({ id: row.id }));
```

**Good (Pass List to Apex):**
```javascript
// One transaction
updateRecordsInApex({ records: this.rows });
```

---

# 38. Security

`@AuraEnabled` does not bypass security, but you must explicitly enforce it:
*   **`with sharing`:** Ensure the class respects record-level access.
*   **CRUD/FLS:** Apex runs in system mode regarding object/field permissions unless explicitly checked.

Use `WITH USER_MODE` or `Security.stripInaccessible()` to enforce CRUD/FLS.

---

# 39. SOQL Security

Secure your Imperative Apex queries:

```java
@AuraEnabled
public static List<Claim__c> getSecureClaims() {
    // WITH USER_MODE ensures the query respects object and field level security
    return [SELECT Id, Name, Amount__c FROM Claim__c WITH USER_MODE];
}
```

---

# 40. Imperative Apex Lifecycle

```text
User Action (Click "Approve")
     ↓
Validate Input (JS checks validity)
     ↓
Set Loading State (isLoading = true)
     ↓
Call Apex (await approveClaim())
     ↓
Server Processing (DML, Callouts)
     ↓
Success / Error (try/catch blocks)
     ↓
Update UI (Toast messages)
     ↓
Refresh Data (refreshApex or notifyRecordUpdateAvailable)
     ↓
Reset Loading State (finally { isLoading = false })
```

---

# 41. Sequential Apex Calls

If Call B depends on the result of Call A, use `await` sequentially.

```javascript
async processComplexLogic() {
    const claim = await saveClaim();
    // Waits for saveClaim to finish, uses its result
    await updateInvoice(claim.Id); 
}
```

---

# 42. Parallel Apex Calls

If calls are independent, execute them concurrently using `Promise.all()` to save time.

```javascript
async loadDashboardData() {
    // Both server calls fire simultaneously
    const [claims, invoices] = await Promise.all([
        getClaims(),
        getInvoices()
    ]);
    this.claimsData = claims;
    this.invoicesData = invoices;
}
```

---

# 43. Conditional Apex Calls

```javascript
async fetchDetails() {
    // Only call server if we have a valid ID
    if (this.recordId) {
        this.details = await getRecordData({ recordId: this.recordId });
    }
}
```

---

# 44. Imperative Apex with Datatable

```javascript
async handleBulkUpdate() {
    const selectedRows = this.template.querySelector('lightning-datatable').getSelectedRows();
    
    try {
        await updateClaims({ claims: selectedRows });
        this.showToast('Success', 'Updated', 'success');
        await refreshApex(this.wiredData); // refresh the table
    } catch (e) {
        // handle
    }
}
```

---

# 45. Imperative Apex with Pagination

Loading 10,000 records into LWC crashes the browser. Use server-side pagination via Imperative Apex using `LIMIT` and `OFFSET` or Keyset Pagination.

**Apex:**
```java
@AuraEnabled
public static List<Claim__c> getClaims(Integer limitSize, Integer offsetSize) { ... }
```
LWC passes the offset as the user clicks "Next Page".

---

# 46. Imperative Apex and Caching

If you call an `@AuraEnabled(cacheable=true)` method imperatively, LWC will check the client-side cache first. If the data exists and is fresh, it returns immediately without hitting the server. 
*   **Pro:** Extremely fast.
*   **Con:** You might get stale data if it was modified elsewhere.

---

# 47. Common Mistakes

*   **Problem:** Method not found. **Cause:** Forgot `@AuraEnabled`. **Solution:** Add the annotation.
*   **Problem:** Undefined parameters in Apex. **Cause:** Parameter names in JS don't perfectly match Apex. **Solution:** Check spelling/case.
*   **Problem:** Promise not resolving. **Cause:** Forgot `await` or `.then()`. **Solution:** Add `await`.
*   **Problem:** Duplicate records. **Cause:** Button clicked twice. **Solution:** Implement `isLoading` flag to disable button.
*   **Problem:** "DML currently not allowed". **Cause:** Performing DML inside `cacheable=true`. **Solution:** Remove `cacheable=true`.

---

# 48. Debugging Imperative Apex

**Debugging Checklist:**
1.  **Network Tab (Browser DevTools):** Look at the `aura` XHR requests. Check the Payload (parameters sent) and the Response (data returned).
2.  **Apex Debug Logs:** Set up a trace flag for the user to catch server-side exceptions.
3.  **Console Logs:** Add `console.log(JSON.parse(JSON.stringify(error)))` in the `catch` block to see exact error structures.

---

# 49. Performance Considerations

*   **Minimize Calls:** Combine multiple small Apex calls into one wrapper-based call.
*   **Selective SOQL:** Filter queries tightly to reduce payload size.
*   **Avoid Over-fetching:** Don't `SELECT *` (via dynamic SOQL); only query fields LWC needs.
*   **Bulkify Server Side:** Pass collections to Apex rather than looping LWC calls.

---

# 50. Best Practices Checklist

*   ✅ **Use Imperative Apex when explicit execution control is required:** E.g., user clicks.
*   ✅ **Use Imperative Apex for create/update/delete operations:** Cannot use Wire for this.
*   ✅ **Use async/await for readable asynchronous code:** Avoids nested Promise hell.
*   ✅ **Always handle success and error states:** Use `try/catch`.
*   ✅ **Use finally for loading-state cleanup:** Ensures spinners disappear even on error.
*   ✅ **Validate user input before calling Apex:** Don't waste server trips for bad data.
*   ✅ **Prevent duplicate submissions:** Disable buttons during server processing.
*   ✅ **Keep Apex bulkified:** Accept Lists instead of single records where applicable.
*   ✅ **Respect CRUD and FLS:** Use `WITH USER_MODE`.
*   ✅ **Use Promise.all for independent Apex calls:** Optimizes network time.

---

# 51. Real Project Scenarios

**1. Create Warranty Claim:** User fills out a form, LWC gathers data, passes wrapper to Apex, Apex inserts `Claim__c`, LWC shows Success toast.
**9. Send Warranty Claim to SAP:** User clicks "Sync SAP", LWC calls Apex, Apex performs HTTP POST callout to SAP, returns SAP ID to LWC, LWC triggers `notifyRecordUpdateAvailable`.
**12. Create Invoice and Line Items:** User selects parts, LWC passes a wrapper containing Invoice header and a List of Line Items to Apex for a single-transaction insert.

---

# 52. Complete End-to-End Example

**Scenario:** Warranty Claim Management (Status Update)

**Apex:**
```java
public with sharing class ClaimController {
    @AuraEnabled
    public static void approveClaim(Id claimId) {
        try {
            Claim__c c = new Claim__c(Id = claimId, Status__c = 'Approved');
            update as user c; 
        } catch (Exception e) {
            throw new AuraHandledException('Approval failed: ' + e.getMessage());
        }
    }
}
```

**JavaScript:**
```javascript
import { LightningElement, api } from 'lwc';
import approveClaim from '@salesforce/apex/ClaimController.approveClaim';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';

export default class ClaimApprover extends LightningElement {
    @api recordId;
    isLoading = false;

    async handleApprove() {
        if (this.isLoading) return;
        this.isLoading = true;

        try {
            await approveClaim({ claimId: this.recordId });
            
            this.dispatchEvent(
                new ShowToastEvent({ title: 'Success', message: 'Claim Approved', variant: 'success' })
            );
            
            // Refresh the standard record page cache
            await notifyRecordUpdateAvailable([{ recordId: this.recordId }]);
            
        } catch (error) {
            this.dispatchEvent(
                new ShowToastEvent({ title: 'Error', message: error.body.message, variant: 'error' })
            );
        } finally {
            this.isLoading = false;
        }
    }
}
```

**HTML:**
```html
<template>
    <lightning-card title="Claim Actions">
        <div class="slds-p-around_medium">
            <template if:true={isLoading}>
                <lightning-spinner alternative-text="Loading"></lightning-spinner>
            </template>
            <lightning-button 
                label="Approve Claim" 
                variant="brand" 
                onclick={handleApprove} 
                disabled={isLoading}>
            </lightning-button>
        </div>
    </lightning-card>
</template>
```

---

# 53. Interview Questions & Answers

### Beginner Questions
**Q: What is Imperative Apex?**
A: It is the manual invocation of an Apex method from LWC JavaScript using Promises, as opposed to the automatic, reactive Wire service.

**Q: Why is `@AuraEnabled` required?**
A: It exposes the server-side method to the client-side framework and handles JSON serialization/deserialization.

### Intermediate Questions
**Q: How do you pass parameters to Imperative Apex?**
A: By passing a JSON object to the imported Apex function where the keys exactly match the Apex method's parameter names.

**Q: Can cacheable Apex perform DML?**
A: No. `@AuraEnabled(cacheable=true)` strictly enforces read-only operations. To perform DML, you must omit `cacheable=true` and call the method imperatively.

### Advanced Questions
**Q: How do you prevent duplicate Apex calls on button double-clicks?**
A: By implementing an `isLoading` boolean tracker, setting it to true instantly upon click, disabling the button in the UI, and resetting it to false in the `finally` block of the Promise.

**Q: What is the difference between `refreshApex` and `notifyRecordUpdateAvailable`?**
A: `refreshApex` is used to force a re-fetch of data that was provisioned via an Apex `@wire` adapter. `notifyRecordUpdateAvailable` tells Lightning Data Service (LDS) that a specific record was updated out-of-band (via Apex DML), prompting LDS to update its cache and UI components.

### Architect-Level Questions
**Q: How do you optimize multiple independent Imperative Apex calls on component load?**
A: Instead of using sequential `await` statements (which creates a waterfall delay), utilize `Promise.all()` to fire the requests concurrently, significantly reducing total network resolution time.

**Q: How do you enforce security in an Imperative Apex call?**
A: The Apex class should use `with sharing` to enforce record access. Since Apex runs in system mode for CRUD/FLS, you must explicitly enforce field and object permissions using `WITH USER_MODE` in SOQL/DML or `Security.stripInaccessible()`.

---

# 54. Revision Summary

*   **Imperative Apex:** Manual, explicit server calls.
*   **@AuraEnabled:** Required to bridge LWC and Apex.
*   **cacheable=true:** Read-only, client cached; no DML allowed.
*   **Promise:** Returns standard JS Promises (`then`, `catch`, `finally`).
*   **async/await:** Modern syntax for handling asynchronous Apex calls cleanly.
*   **Parameters:** JSON keys must exactly match Apex signature.
*   **Wrapper Classes:** Best practice for passing complex objects/collections.
*   **AuraHandledException:** Wrap Apex exceptions to present clean errors to LWC.
*   **Loading State:** Always manage `isLoading` to prevent duplicates and improve UX.
*   **refreshApex:** Refreshes `@wire` Apex data.
*   **notifyRecordUpdateAvailable:** Syncs LDS cache with Imperative DML changes.
*   **Security:** Always use `with sharing` and `WITH USER_MODE` to secure manual calls.
*   **Performance:** Bulkify inputs, use `Promise.all` for parallel calls, minimize server round-trips.