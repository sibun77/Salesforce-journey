# Salesforce Lightning Web Components Iteration & Dynamic Lists Handbook

## 1. Introduction

In modern web application development, displaying lists of data is a fundamental requirement. **Iteration** in Lightning Web Components (LWC) is the mechanism by which developers dynamically render collections of data—such as arrays of Salesforce records—into the Document Object Model (DOM).

Without iteration, you would have to hardcode HTML for every single record, which is impossible when dealing with dynamic database queries. LWC provides powerful, declarative template directives (`for:each` and `iterator`) to loop through JavaScript arrays and generate repeating HTML structures efficiently.

**Real-World Automotive CRM Use Cases:**
*   **Warranty Claims:** Displaying a list of claims submitted by a dealer.
*   **Claim Line Items:** Iterating through individual parts claimed under a specific warranty ticket.
*   **Work Orders:** Showing a daily schedule of repair jobs for a service technician.
*   **Vehicles:** Rendering a gallery of customer vehicles in their garage portal.
*   **Spare Parts:** Listing available inventory matching a search query.
*   **Invoices & Shipment Items:** Displaying line items with subtotal calculations in an order tracking interface.

---

## 2. What is Iteration in LWC?

Iteration is the process of looping over a data structure (typically an Array) and executing a block of logic or rendering a block of UI for each item in that collection.

In LWC, you define a JavaScript array containing your data (e.g., a list of SObjects returned from Apex). You then use the `<template>` tag in your HTML file with special directives to tell the LWC framework: *"For every item in this array, stamp out this chunk of HTML."*

**The Relationship Between JavaScript and HTML:**
The LWC engine creates a reactive binding between your JavaScript array and the rendered DOM. If the array changes (items added, removed, or reordered), LWC's rendering engine updates only the necessary DOM elements.

### Flow Diagram: Dynamic Rendering Process

```text
[ JavaScript Array ] (e.g., 3 Vehicle Records)
        │
        ▼
[ LWC Template Engine ]
        │
        ├── Reads directive: for:each={vehicles}
        ├── Loops over array
        │
        ▼
[ Iteration 1 ] ➔ Stamps HTML for Vehicle 1 (Key: V-001)
[ Iteration 2 ] ➔ Stamps HTML for Vehicle 2 (Key: V-002)
[ Iteration 3 ] ➔ Stamps HTML for Vehicle 3 (Key: V-003)
        │
        ▼
[ Rendered DOM ] (List of 3 Vehicle Cards displayed to User)
```

---

## 3. for:each Directive

The `for:each` directive is the standard way to render lists in LWC. It requires three critical pieces: the array, a name for the current item, and a unique key.

**Basic Syntax:**
```html
<template for:each={records} for:item="record">
    <p key={record.Id}>{record.Name}</p>
</template>
```

**Line-by-Line Explanation:**
*   `<template for:each={records} ...>`: The `<template>` tag is a virtual wrapper. It does not render in the DOM. `for:each={records}` tells LWC to iterate over the `records` array defined in the JavaScript file.
*   `for:item="record"`: This defines the variable name (`record`) that will represent the current item in the loop for that specific iteration.
*   `<p key={record.Id}>`: Every element stamped by the loop **must** have a `key` attribute. The key must be a unique, stable string/number identifying that specific item.
*   `{record.Name}`: Data binding mapping the `Name` property of the current `record` object to the text inside the `<p>` tag.
*   `</template>`: Closes the virtual iteration block.

---

## 4. for:item

The `for:item` directive assigns a variable name to the current element being processed in the array. 

*   **Purpose:** It gives you a way to reference the current object's properties (or the primitive value itself) within the inner HTML block.
*   **Naming the current item:** You can name this anything (e.g., `for:item="vehicle"`, `for:item="claim"`, `for:item="part"`). It is best practice to use a singular noun that represents the items in the array.
*   **Accessing properties:** You use standard dot notation `{item.Property}` to access data.

**Example:**
```html
<template for:each={vehicles} for:item="vehicle">
    <div key={vehicle.Id}>
        <p>Model: {vehicle.Model__c}</p>
        <p>VIN: {vehicle.VIN__c}</p>
    </div>
</template>
```

**How it changes:**
During iteration 1, `vehicle` points to `vehicles[0]`. During iteration 2, `vehicle` points to `vehicles[1]`, and so on. The template executes its inner content once for each item, binding the specific `vehicle` properties to that DOM instance.

---

## 5. for:index

The `for:index` directive is an optional attribute you can add to capture the current position (index) of the item in the array.

*   **What it does:** It assigns the zero-based integer index of the loop to a variable.
*   **Zero-based indexing:** The first item is `0`, the second is `1`, etc.
*   **Displaying row numbers:** Useful for generating numbered lists or applying alternating row logic.

**Example:**
```html
<template for:each={records} for:item="record" for:index="index">
    <p key={record.Id}>
        Row {index} - {record.Name}
    </p>
</template>
```

**Why you should NOT use index as a key:**
While it might be tempting to use `key={index}`, **you should avoid this** unless the list is completely static. If an item is deleted from the middle of the array, the indexes of all subsequent items shift. The LWC engine will think the items themselves changed, destroying and recreating DOM elements unnecessarily, which degrades performance and can cause focus/state loss in inputs.

---

## 6. key Attribute

The `key` attribute is arguably the most important, yet most misunderstood, part of LWC iteration.

**Why key is required:**
LWC strictly mandates the `key` attribute on the first HTML element inside the iteration template. If you omit it, the component will not compile.

**How LWC uses keys (DOM Diffing):**
When an array changes, LWC compares the old list of keys with the new list of keys.
*   If a new key appears, LWC creates a new DOM element.
*   If a key disappears, LWC destroys that DOM element.
*   If a key remains but its position changes, LWC simply moves the existing DOM element without recreating it.

**Stable Identity:**
The key provides a stable identity. `record.Id` is the perfect key because a Salesforce Record ID never changes, regardless of where the record sits in the array.

**Example:**
```html
<div key={record.Id}>...</div> <!-- PERFECT -->
```

**Common Key Mistakes:**
1.  **Using `key={index}`:** If item at index 0 is removed, item 1 becomes index 0. LWC thinks item 0 just changed its data, causing massive UI bugs and performance hits.
2.  **Duplicate keys:** If two items have the same key, LWC throws a runtime error and the UI breaks.
3.  **Placing key on the template:** `<template key={...}>` is invalid. The key goes on the first actual HTML/Component tag *inside* the template.

---

## 7. Complete for:each Example

Here is a complete example displaying a list of Vehicles.

**JavaScript (vehicleList.js):**
```javascript
import { LightningElement } from 'lwc';

export default class VehicleList extends LightningElement {
    vehicles = [
        { Id: '001', Name: 'Mahindra Tractor', Model: '575 DI', Status: 'Active' },
        { Id: '002', Name: 'Tata Truck', Model: 'Signa', Status: 'In Shop' }
    ];
}
```

**HTML (vehicleList.html):**
```html
<template>
    <lightning-card title="Fleet Status" icon-name="custom:custom31">
        <div class="slds-m-around_medium">
            <template for:each={vehicles} for:item="vehicle">
                <!-- Key must be on the first element inside the template -->
                <div key={vehicle.Id} class="slds-box slds-m-bottom_small">
                    <p><strong>Name:</strong> {vehicle.Name}</p>
                    <p><strong>Model:</strong> {vehicle.Model}</p>
                    <p><strong>Status:</strong> {vehicle.Status}</p>
                </div>
            </template>
        </div>
    </lightning-card>
</template>
```

**XML (vehicleList.js-meta.xml):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="[http://soap.sforce.com/2006/04/metadata](http://soap.sforce.com/2006/04/metadata)">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__RecordPage</target>
        <target>lightning__HomePage</target>
    </targets>
</LightningComponentBundle>
```

**Line-by-Line Explanation (HTML):**
*   `<template>`: Root LWC wrapper.
*   `<lightning-card ...>`: Base component for UI styling.
*   `<template for:each={vehicles} for:item="vehicle">`: Starts the loop over the `vehicles` array.
*   `<div key={vehicle.Id} class="...">`: The wrapper `div` for each vehicle. It has the required `key` using the unique `Id`.
*   `<p>...{vehicle.Name}...</p>`: Data binding prints the primitive values.
*   `</template>`: Ends loop.

---

## 8. Iterating Salesforce Records

Usually, you aren't iterating hardcoded arrays. You are iterating data retrieved from Salesforce via Apex, `@wire`, or Lightning Data Service.

**Apex Controller:**
```java
public with sharing class VehicleController {
    @AuraEnabled(cacheable=true)
    public static List<Vehicle__c> getVehicles() {
        return [SELECT Id, Name, Status__c FROM Vehicle__c LIMIT 10];
    }
}
```

**JavaScript:**
```javascript
import { LightningElement, wire } from 'lwc';
import getVehicles from '@salesforce/apex/VehicleController.getVehicles';

export default class WiredVehicles extends LightningElement {
    vehicles = [];

    @wire(getVehicles)
    wiredVehicles({ data, error }) {
        if (data) {
            this.vehicles = data; // Assign Apex result to array
        } else if (error) {
            console.error(error);
        }
    }
}
```

**HTML:**
```html
<template>
    <template for:each={vehicles} for:item="vehicle">
        <div key={vehicle.Id}>
            {vehicle.Name} - {vehicle.Status__c}
        </div>
    </template>
</template>
```

**Data Flow:**
1. Component loads, `@wire` triggers Apex.
2. Apex runs SOQL and returns `List<Vehicle__c>`.
3. The `wiredVehicles` method receives data and assigns it to `this.vehicles`.
4. The assignment triggers LWC's reactivity system.
5. The DOM re-renders, iterating over `vehicles` and stamping HTML using `vehicle.Id` as the key.

---

## 9. Iterating Objects

When iterating SObjects or custom JSON, you often need to access nested properties.

**Accessing properties:**
```html
<p>Name: {record.Name}</p>
<p>Amount: {record.Amount}</p>
```

**Nested Object Access:**
If your query includes parent fields (e.g., `SELECT Id, Account.Name FROM Contact`), you access them via dot notation.
```html
<p>Dealer Name: {record.Account.Name}</p>
```

**Null Considerations:**
In HTML, if `record.Account` is null, `{record.Account.Name}` will fail gracefully and render nothing. However, if you try to process this in JavaScript, it will throw a `TypeError`. Be mindful of data structure consistency.

---

## 10. Iterating Arrays of Primitive Values

Sometimes you iterate over an array of Strings, Numbers, or Booleans rather than Objects.

**JavaScript:**
```javascript
export default class PrimitivesList extends LightningElement {
    categories = ['Salesforce', 'Apex', 'LWC'];
}
```

**HTML:**
```html
<template>
    <ul>
        <template for:each={categories} for:item="category">
            <!-- Since there is no ID, the string itself can be the key (if unique) -->
            <li key={category}>{category}</li>
        </template>
    </ul>
</template>
```

**Key Handling:** If the primitives are guaranteed to be unique (like category names), use the item itself as the key `key={category}`. If there might be duplicates (e.g., `[10, 10, 20]`), you must transform the array in JavaScript into objects with unique IDs before rendering.

---

## 11. Reactive Arrays

To make the UI update when an array changes, you must understand LWC reactivity. LWC observes array *mutations* differently depending on how you modify them.

**Modern LWC Reactive Behavior (Immutability):**
While LWC can track `.push()`, the safest, most predictable, and standard modern JavaScript pattern is **immutability**—creating a new array reference.

Compare:
```javascript
// Mutating existing array (can sometimes fail to trigger @api setter in child components)
this.records.push(newRecord); 
```
With:
```javascript
// Immutable update (reassigns array reference, guarantees UI update)
this.records = [...this.records, newRecord];
```

By reassigning the array, LWC's reactivity engine instantly knows the state changed, triggers the `render()` lifecycle, and performs DOM diffing via the `key` attribute.

Use `map()` for updating, `filter()` for removing, and the spread operator `[...]` for adding.

---

## 12. Adding Items to an Iterated List

**Scenario:** Add a new Claim Line Item to the UI without querying the database again.

**JavaScript:**
```javascript
export default class ClaimLines extends LightningElement {
    lineItems = [
        { Id: 'L001', Part: 'Brake Pad', Price: 50 }
    ];

    handleAddLine() {
        // Generate a pseudo-random ID for the client-side key
        const newId = 'L' + Date.now(); 
        
        const newLine = {
            Id: newId,
            Part: 'New Part',
            Price: 0
        };

        // Create a new array with the old items PLUS the new item
        this.lineItems = [...this.lineItems, newLine];
    }
}
```

**Explanation:**
*   User clicks the "Add" button.
*   `handleAddLine` fires.
*   We create a `newLine` object with a unique `Id` (critical for the `key`).
*   We use the spread operator `[...this.lineItems, newLine]` to create a new array.
*   LWC detects the new array reference.
*   LWC looks at the keys. It sees 'L001' already exists (keeps DOM intact). It sees `newId` is new, and renders only the new row.

---

## 13. Removing Items from an Iterated List

To remove an item reactively, use the `Array.prototype.filter()` method.

**JavaScript:**
```javascript
handleDelete(event) {
    const itemIdToRemove = event.currentTarget.dataset.id;

    // Filter out the item that matches the ID
    this.items = this.items.filter(
        item => item.Id !== itemIdToRemove
    );
}
```

**Explanation:**
*   `filter()` loops through `this.items`.
*   The arrow function `item => item.Id !== itemIdToRemove` returns `true` to keep the item, and `false` to discard it.
*   `filter` returns a *new* array.
*   Assigning this new array to `this.items` triggers LWC to rerender.
*   LWC notices the removed ID is missing from the keys and deletes that specific DOM node.

---

## 14. Updating Items in an Iterated List

To update a specific item in an array without modifying the entire array or mutating existing objects, use `Array.prototype.map()`.

**JavaScript:**
```javascript
handleApprove(event) {
    const itemIdToApprove = event.currentTarget.dataset.id;

    this.items = this.items.map(item => {
        if (item.Id === itemIdToApprove) {
            // Return a new object with spread operator, overwriting Status
            return {
                ...item,
                Status: 'Approved'
            };
        }
        // Return unchanged item
        return item; 
    });
}
```

**Explanation:**
*   `map()` creates a completely new array.
*   It checks each item. If the ID matches, we create a *new* object `{ ...item, Status: 'Approved' }` copying old properties and updating Status.
*   If the ID doesn't match, we return the original object reference.
*   LWC sees the new array. Thanks to the `key`, it knows which row changed and efficiently updates just the status text for that specific record.

---

## 15. Event Handling Inside for:each

When you have interactive elements (buttons, inputs) inside a loop, you need a way to know *which* record the user interacted with.

**HTML:**
```html
<template for:each={records} for:item="record">
    <lightning-button
        key={record.Id}
        label="View"
        data-id={record.Id}
        onclick={handleView}>
    </lightning-button>
</template>
```

**JavaScript:**
```javascript
handleView(event) {
    const recordId = event.currentTarget.dataset.id;
    console.log('User clicked record:', recordId);
}
```

**Explanation:**
*   `data-id={record.Id}`: We attach a custom HTML5 data attribute (`data-*`) to the button. This embeds the record ID directly into the DOM element.
*   `onclick={handleView}`: Binds the click event.
*   `event.currentTarget`: Refers to the element that the event listener is attached to (the `<lightning-button>`).
*   `.dataset.id`: Retrieves the value of `data-id`. (Note: `data-record-id` becomes `dataset.recordId` in JS via camelCase).

---

## 16. event.target vs event.currentTarget in Loops

This is a critical distinction that causes many bugs in LWC iteration.

| Feature | `event.target` | `event.currentTarget` |
| :--- | :--- | :--- |
| **Definition** | The actual lowest-level DOM element that was physically clicked. | The element that has the `on[event]` handler attached to it. |
| **Nested Elements** | Can be a child span, icon, or path inside the button. | Will ALWAYS be the wrapper element with the listener. |
| **Dataset Reliability** | **Unreliable.** If you click an icon inside a button, target is the icon, which has no dataset. | **Highly Reliable.** Always gives you the dataset attached to the listener element. |

**Why `currentTarget` is safer:**
Imagine a button with an icon inside it:
```html
<button data-id="123" onclick={handleClick}>
    <lightning-icon icon-name="utility:delete"></lightning-icon>
    Delete
</button>
```
If the user clicks exactly on the icon, `event.target` is the `<lightning-icon>`. Calling `event.target.dataset.id` returns `undefined`. But `event.currentTarget` is always the `<button>`, so `event.currentTarget.dataset.id` safely returns `"123"`.

---

## 17. Conditional Rendering Inside Iteration

You frequently need to show/hide elements per row based on data. Use modern `lwc:if`, `lwc:elseif`, and `lwc:else` inside the loop.

**HTML Example:**
```html
<template for:each={claims} for:item="claim">
    <div key={claim.Id} class="claim-card">
        <h3>{claim.ClaimNumber}</h3>

        <!-- Conditional rendering based on claim status -->
        <template lwc:if={claim.isApproved}>
            <span class="badge-green">Approved</span>
        </template>
        
        <template lwc:elseif={claim.isRejected}>
            <span class="badge-red">Rejected</span>
        </template>
        
        <template lwc:else>
            <span class="badge-gray">Pending Review</span>
        </template>
    </div>
</template>
```

**Correct Structure:**
*   The `key` stays on the outermost wrapper (`<div key={claim.Id}>`).
*   The `<template lwc:if>` tags go *inside* that wrapper.
*   **Do not** put the `key` on the `<template lwc:if>` tag itself.
*   Properties evaluated by `lwc:if` (e.g., `claim.isApproved`) must be boolean properties physically present on the JavaScript objects in the array. LWC templates cannot process expressions like `{claim.Status === 'Approved'}`.

---

## 18. Iterating with Getters

Because LWC HTML templates are logic-less (you cannot do `if item.Amount > 100` in HTML), you must prepare data in JavaScript. Getters are perfect for this.

**JavaScript:**
```javascript
export default class WarrantyList extends LightningElement {
    records = [...]; // Data from wire

    // Getter computes a derived list on the fly
    get activeRecords() {
        if (!this.records) return [];
        return this.records.filter(
            record => record.Status === 'Active'
        );
    }
}
```

**HTML:**
```html
<!-- Iterate over the getter, not the raw records -->
<template for:each={activeRecords} for:item="record">
    <p key={record.Id}>{record.Name}</p>
</template>
```

**Why Getters are useful:**
*   **Filtering:** Allows you to keep the source array intact while displaying a filtered view.
*   **Clean Templates:** Moves data manipulation out of HTML and into JS.
*   **Performance Note:** Getters run on every render cycle. Avoid extremely heavy computations inside getters for very large arrays.

---

## 19. Nested Iteration

Sometimes your data is hierarchical, such as a Warranty Claim that has multiple Claim Line Items. You can nest `for:each` loops.

**Data Structure:**
```javascript
claims = [
    {
        Id: 'C1',
        Name: 'Engine Failure',
        lineItems: [
            { Id: 'L1', Name: 'Piston', Price: 200 },
            { Id: 'L2', Name: 'Gasket', Price: 50 }
        ]
    }
];
```

**HTML:**
```html
<template for:each={claims} for:item="claim">
    <!-- Outer loop key -->
    <div key={claim.Id} class="claim-box">
        <h2>{claim.Name}</h2>

        <ul>
            <!-- Inner loop iterating over nested array -->
            <template for:each={claim.lineItems} for:item="line">
                <!-- Inner loop key -->
                <li key={line.Id}>{line.Name} - ${line.Price}</li>
            </template>
        </ul>
    </div>
</template>
```

**Rules for Nested Loops:**
*   Both the outer and inner loops require unique keys.
*   The inner loop variable (`lineItems`) must be a property of the current outer item (`claim.lineItems`).
*   Be mindful of DOM size. Rendering 50 claims, each with 50 line items, creates thousands of DOM nodes and will impact performance.

---

## 20. Iterator Directive

LWC provides an alternative iteration directive called `iterator`. It is specifically used when you need to apply special rendering logic to the **first** or **last** item in the list.

**Basic Syntax:**
```html
<template iterator:it={records}>
    <!-- The key is placed on the element, accessing the value property -->
    <div key={it.value.Id}>
        {it.value.Name}
    </div>
</template>
```

**Explanation:**
*   `iterator:it={records}`: Initiates the loop. `it` is the variable name for the iterator object.
*   `it.value`: Contains the actual object from your array (e.g., `it.value.Name`, `it.value.Id`).
*   `it.index`: The current index (0, 1, 2...).
*   `it.first`: A boolean (`true` if it is the first item in the array).
*   `it.last`: A boolean (`true` if it is the last item in the array).

---

## 21. iterator:first

You use `it.first` to render something exclusively for the top item.

**Example:**
```html
<template iterator:it={records}>
    <div key={it.value.Id}>
        <!-- Only shows on the very first loop iteration -->
        <template lwc:if={it.first}>
            <div class="header-banner">Top Result!</div>
        </template>
        
        <p>{it.value.Name}</p>
    </div>
</template>
```
**Practical Use Case:** Adding a special "Featured" badge to the first result in a search list, or injecting a top-border only on the first item of a custom table.

---

## 22. iterator:last

You use `it.last` to render something exclusively for the bottom item.

**Example:**
```html
<template iterator:it={records}>
    <div key={it.value.Id}>
        <p>{it.value.Name}</p>
        
        <!-- Only shows on the very last loop iteration -->
        <template lwc:if={it.last}>
            <div class="footer-summary">End of Results</div>
        </template>
    </div>
</template>
```
**Practical Use Case:** Rendering an "Add New Row" button immediately after the last item in a dynamic form, or removing the bottom border from the final item in a list.

---

## 23. for:each vs iterator

| Feature | `for:each` | `iterator` |
| :--- | :--- | :--- |
| **Syntax** | `<template for:each={arr} for:item="x">` | `<template iterator:it={arr}>` |
| **Item Access** | `{x.Property}` | `{it.value.Property}` |
| **Index Access** | Via `for:index="idx"` (`{idx}`) | Built-in via `{it.index}` |
| **First/Last Info** | ❌ Not possible in HTML | ✅ Built-in (`{it.first}`, `{it.last}`) |
| **Readability** | High (cleaner data binding) | Medium (requires `.value` everywhere) |
| **Primary Use Case** | 95% of standard lists | When UI requires specific first/last logic |

**Rule of Thumb:**
Always default to `for:each`. Only switch to `iterator` if you absolutely need to conditionally render DOM nodes based on whether the item is the first or last in the array.

---

## 24. for:each vs lightning-datatable

Often developers build complex tables using `for:each` when a standard component would be better.

| Feature | `<template for:each>` | `<lightning-datatable>` |
| :--- | :--- | :--- |
| **Use Case** | Custom layouts (Cards, Grids, Tiles) | Strict tabular data (Rows/Columns) |
| **Complexity** | You build everything (HTML/CSS) | Out-of-the-box component |
| **Sorting** | You must write custom JS logic | Built-in column sorting |
| **Row Actions** | Custom buttons with `data-id` | Built-in row level actions menu |
| **Inline Editing** | Extremely difficult to build from scratch | Built-in standard feature |
| **Mobile UX** | Great (can build responsive cards) | Poor (tables scroll horizontally) |
| **Performance** | Good for simple, highly customized UI | Optimized via virtual rendering |

**When to use which:**
*   Use `for:each` when you need a custom UI layout (e.g., a Kanban board, a grid of vehicle images, a timeline).
*   Use `lightning-datatable` when you need an Excel-like grid with features like sorting, inline editing, and row selection.

---

## 25. Iteration with Lightning Base Components

You can loop over standard base components just like regular HTML tags.

**Example with `<lightning-card>` & `<lightning-badge>`:**
```html
<div class="slds-grid slds-wrap">
    <template for:each={dealers} for:item="dealer">
        <div key={dealer.Id} class="slds-col slds-size_1-of-3 slds-p-around_small">
            
            <lightning-card title={dealer.Name} icon-name="standard:account">
                <div class="slds-m-around_medium">
                    <p>Rating: <lightning-badge label={dealer.Rating}></lightning-badge></p>
                    
                    <lightning-button 
                        label="Contact" 
                        data-id={dealer.Id} 
                        onclick={handleContact}
                        class="slds-m-top_small">
                    </lightning-button>
                </div>
            </lightning-card>

        </div>
    </template>
</div>
```
*Note: Always remember the `key` goes on the direct child of the `<template>` (in this case, the wrapper `<div>`), not inside the base component.*

---

## 26. Iterating Dynamic Components

Sometimes the UI elements should change entirely based on the record type. 

**Example:**
In a Claim system, "Parts" might need an input field for quantity, while "Labor" might need an input field for hours.

```html
<template for:each={lineItems} for:item="line">
    <div key={line.Id}>
        
        <template lwc:if={line.isPart}>
            <lightning-input type="number" label="Quantity" value={line.qty}></lightning-input>
        </template>
        
        <template lwc:elseif={line.isLabor}>
            <lightning-input type="number" label="Hours" value={line.hours}></lightning-input>
        </template>

    </div>
</template>
```

---

## 27. Sorting Iterated Records

LWC arrays do not sort themselves. You must sort them in JavaScript before rendering. 

**JavaScript:**
```javascript
sortRecordsByName() {
    // 1. Create a copy using spread operator to avoid mutating original
    // 2. Use JS array.sort() with localeCompare for strings
    this.records = [...this.records].sort((a, b) => {
        return a.Name.localeCompare(b.Name);
    });
}
```
**Why create a new array?**
Standard `sort()` mutates the array in place. Due to LWC reactivity, mutating an array in place may not reliably trigger a DOM re-render. Creating a new array `[...this.records]` ensures LWC detects the reference change.

---

## 28. Filtering Iterated Records

To filter the list (e.g., a search bar), maintain the *original* data and render a *filtered* version.

**JavaScript:**
```javascript
allRecords = []; // Full dataset from Apex
filteredRecords = []; // Array bound to HTML for:each

// Wire adapter
@wire(getVehicles)
wiredData({ data }) {
    if (data) {
        this.allRecords = data;
        this.filteredRecords = data; // Initially show all
    }
}

handleSearch(event) {
    const searchTerm = event.target.value.toLowerCase();
    
    // Filter from the master list, assign to the display list
    this.filteredRecords = this.allRecords.filter(record => 
        record.Name.toLowerCase().includes(searchTerm)
    );
}
```
**Explanation:** This separates the state (source of truth) from the view model (what is currently displayed).

---

## 29. Pagination with for:each

For small datasets (< 200 records), you can slice the array in JavaScript (**Client-Side Pagination**). For large datasets, use SOQL OFFSET/LIMIT (**Server-Side Pagination**).

**Client-Side Pagination Overview:**
```javascript
fullDataset = [...];
pageSize = 10;
currentPage = 1;

// Getter bound to HTML for:each
get visibleRecords() {
    const start = (this.currentPage - 1) * this.pageSize;
    const end = start + this.pageSize;
    return this.fullDataset.slice(start, end);
}

handleNext() {
    this.currentPage++;
}
```
*Note: If querying thousands of Salesforce records, client-side pagination will crash the browser or breach heap limits. Always prefer server-side strategies for enterprise data.*

---

## 30. Loading, Empty, and Error States

A robust component handles all phases of data retrieval.

**HTML Structure:**
```html
<template>
    <!-- 1. Loading State -->
    <template lwc:if={isLoading}>
        <lightning-spinner alternative-text="Loading..."></lightning-spinner>
    </template>

    <template lwc:else>
        <!-- 2. Error State -->
        <template lwc:if={error}>
            <div class="error-msg">{error}</div>
        </template>

        <!-- 3. Success with Data -->
        <template lwc:elseif={hasRecords}>
            <ul>
                <template for:each={records} for:item="record">
                    <li key={record.Id}>{record.Name}</li>
                </template>
            </ul>
        </template>

        <!-- 4. Empty State -->
        <template lwc:else>
            <div class="empty-state">No vehicles found.</div>
        </template>
    </template>
</template>
```

---

## 31. Performance Considerations

Iterating over DOM elements is expensive. Keep these rules in mind:

1.  **Stable Keys:** Never use `index` as a key. LWC relies on stable keys (like `record.Id`) to reuse DOM nodes instead of destroying and recreating them during array updates.
2.  **Large Arrays:** The browser limits how many DOM nodes can be efficiently rendered. Rendering 5,000 records simultaneously will freeze the browser. Use pagination or infinite scrolling.
3.  **Expensive Getters:** If a getter is bound to an iteration, it executes on *every* reactive change. Do not put heavy loops inside a getter.
4.  **Nested Loops:** A loop inside a loop multiplies DOM nodes exponentially. Ensure nested arrays are kept small.
5.  **`lwc:dom="manual"`:** If rendering performance is truly critical (e.g., millions of data points), bypass `for:each` entirely and use manual DOM manipulation, though this is rare.

---

## 32. Common Mistakes

| Problem | Cause | Solution | Example |
| :--- | :--- | :--- | :--- |
| **Compilation Error** | Missing `key` attribute. | Add `key={uniqueId}` to the first element inside the template. | `<div key={record.Id}>` |
| **DOM completely refreshes on array update** | Using `index` as the `key`. | Use a stable unique identifier from the data object. | Switch `key={index}` to `key={record.Id}` |
| **Wrong record deleted on click** | Using `event.target.dataset`. | Use `event.currentTarget.dataset` to get the reliable ID of the button wrapper. | `const id = event.currentTarget.dataset.id;` |
| **Component throws duplicate key error** | Two items in array have same ID. | Ensure data source yields unique values. | Check Apex query for duplicate IDs. |
| **UI doesn't update when data changes** | Mutating array in place (`push`). | Use immutability (`spread` operator) to trigger reactivity. | `this.arr = [...this.arr, new];` |

---

## 33. Debugging Iteration

**Practical Debugging Checklist:**

1.  **Blank List?**
    *   Check if the array is actually populated (`console.log(JSON.stringify(this.records))`).
    *   Ensure your template conditionals (`lwc:if`) aren't hiding the block.
2.  **Values not showing? (e.g., `[object Object]` instead of text)**
    *   Ensure you are using dot notation correctly (`{record.Name}`, not just `{record}`).
3.  **Event ID Undefined?**
    *   Ensure your HTML has `data-id={record.Id}`.
    *   Ensure your JS uses `event.currentTarget.dataset.id`.
4.  **UI not updating after add/remove?**
    *   You mutated the array instead of reassigning it. Switch to `map`, `filter`, or `[...]`.

---

## 34. Best Practices Checklist

*   ✅ **Always provide a stable unique key:** Without it, DOM diffing fails.
*   ✅ **Prefer record.Id as key when available:** It guarantees cross-render stability.
*   ✅ **Use for:each for normal list rendering:** It is cleaner and more readable than `iterator`.
*   ✅ **Use iterator when first/last information is required:** The only time `iterator` is superior.
*   ✅ **Keep templates simple:** Move complex boolean logic into JS by adding properties to the object before rendering.
*   ✅ **Use immutable array update patterns where appropriate:** Reassign arrays rather than `.push()` or `.splice()`.
*   ✅ **Avoid unnecessary nested loops:** Flatten data structures in Apex if possible.
*   ✅ **Use server-side pagination for large datasets:** Protect browser memory.
*   ✅ **Use lightning-datatable for data-heavy tabular interfaces:** Don't reinvent the wheel.
*   ✅ **Use event.currentTarget.dataset for reliable record identification:** Prevents target-bubbling bugs.

---

## 35. Real Project Scenarios (Automotive CRM)

1.  **Warranty Claim List:** Rendered using `lightning-datatable` due to the need for column sorting by Date and Status.
2.  **Claim Line Items:** Rendered using `for:each` inside an accordion, where each part (Engine, Brake, Electrical) is a nested `for:each` loop.
3.  **Work Order Services:** Used `for:each` to render draggable cards in a Kanban-style technician schedule.
4.  **Vehicle Gallery:** Used `for:each` to iterate over custom JSON containing image URLs and Vehicle Specs, displayed as `<lightning-card>` tiles.
5.  **Service History Timeline:** Used `iterator` to render a vertical line connecting past service dates, checking `it.last` to stop drawing the connecting line on the final chronological record.

---

## 36. Complete End-to-End Example

**Scenario:** A Warranty Claim screen that iterates over Claim Line Items, allows viewing details, and deleting a line item.

**Apex (ClaimController.cls):**
```java
public with sharing class ClaimController {
    @AuraEnabled(cacheable=true)
    public static List<Claim_Line__c> getLines(Id claimId) {
        return [SELECT Id, Part_Name__c, Quantity__c, Price__c, Status__c 
                FROM Claim_Line__c WHERE Claim__c = :claimId];
    }

    @AuraEnabled
    public static void deleteLineItem(Id lineId) {
        delete new Claim_Line__c(Id = lineId);
    }
}
```

**JavaScript (claimLines.js):**
```javascript
import { LightningElement, api, wire } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import getLines from '@salesforce/apex/ClaimController.getLines';
import deleteLineItem from '@salesforce/apex/ClaimController.deleteLineItem';

export default class ClaimLines extends LightningElement {
    @api recordId; // Passed in from Lightning Record Page
    lineItems = [];

    @wire(getLines, { claimId: '$recordId' })
    wiredLines({ error, data }) {
        if (data) {
            // Map data to append UI-specific boolean for conditional rendering
            this.lineItems = data.map(item => ({
                ...item,
                isApproved: item.Status__c === 'Approved'
            }));
        } else if (error) {
            this.showToast('Error', error.body.message, 'error');
        }
    }

    async handleDelete(event) {
        // 1. Get exact ID using currentTarget
        const lineId = event.currentTarget.dataset.id;

        try {
            // 2. Call Apex to delete in database
            await deleteLineItem({ lineId: lineId });
            
            // 3. Reactively update the UI by filtering out the deleted ID
            this.lineItems = this.lineItems.filter(item => item.Id !== lineId);
            
            this.showToast('Success', 'Line item removed.', 'success');
        } catch (error) {
            this.showToast('Error', 'Could not delete item.', 'error');
        }
    }

    showToast(title, message, variant) {
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
    }
}
```

**HTML (claimLines.html):**
```html
<template>
    <lightning-card title="Claim Line Items" icon-name="custom:custom13">
        <div class="slds-m-around_medium">
            
            <template lwc:if={lineItems.length}>
                <ul class="slds-has-dividers_around-space">
                    
                    <!-- ITERATION START -->
                    <template for:each={lineItems} for:item="line">
                        
                        <!-- KEY on wrapper -->
                        <li key={line.Id} class="slds-item slds-grid slds-grid_vertical-align-center">
                            <div class="slds-col slds-size_3-of-4">
                                <p><strong>{line.Part_Name__c}</strong> (Qty: {line.Quantity__c})</p>
                                <p>Price: ${line.Price__c}</p>
                                
                                <!-- CONDITIONAL RENDERING inside loop -->
                                <template lwc:if={line.isApproved}>
                                    <lightning-badge label="Approved" class="slds-theme_success"></lightning-badge>
                                </template>
                            </div>

                            <div class="slds-col slds-size_1-of-4 slds-text-align_right">
                                <!-- EVENT HANDLING inside loop -->
                                <lightning-button-icon 
                                    icon-name="utility:delete" 
                                    variant="border-filled" 
                                    alternative-text="Delete" 
                                    data-id={line.Id} 
                                    onclick={handleDelete}>
                                </lightning-button-icon>
                            </div>
                        </li>

                    </template>
                    <!-- ITERATION END -->

                </ul>
            </template>

            <!-- EMPTY STATE -->
            <template lwc:else>
                <p>No line items found for this claim.</p>
            </template>

        </div>
    </lightning-card>
</template>
```

**Explanation:**
1.  **Wire:** Fetches data, maps it to add a front-end property (`isApproved`), and stores it in `lineItems`.
2.  **HTML `for:each`:** Iterates `lineItems`, assigning `line` to the current item.
3.  **Key:** `key={line.Id}` is placed on the `<li>`.
4.  **Conditionals:** `lwc:if={line.isApproved}` reads the boolean we mapped in JS to show a green badge.
5.  **Data Binding:** Binds `data-id={line.Id}` to the delete button.
6.  **JS Event:** `handleDelete` extracts `dataset.id`, calls server to delete, and then uses `.filter()` to reactively remove the item from the browser without a second server trip.

---

## 37. Interview Questions & Answers

### Beginner Questions

**Q: What is `for:each` in LWC?**
A: It is a template directive used to iterate over a JavaScript array and render a block of HTML for every item in that array.

**Q: What is `for:item`?**
A: It specifies the variable name assigned to the current element being processed in the loop, allowing you to access its properties via dot notation (e.g., `{item.Name}`).

**Q: Why is the `key` attribute required?**
A: The framework requires a unique identifier to optimize DOM rendering. It helps LWC track which specific items have changed, been added, or been removed, avoiding unnecessary destruction and recreation of the entire list.

### Intermediate Questions

**Q: Why should `record.Id` generally be preferred over array index as a `key`?**
A: Array indexes are not tied to the data. If an item is deleted from the middle of the array, the indexes of all subsequent items shift. The LWC DOM engine will interpret this as all subsequent records changing their data, leading to performance degradation and UI bugs. `record.Id` provides a permanent, stable identity.

**Q: How do you handle events inside `for:each`?**
A: You attach a custom data attribute to the interactive element, like `data-id={record.Id}`. On the event handler in JS, you retrieve this value to know which record was interacted with.

**Q: What is the difference between `event.target` and `event.currentTarget`?**
A: `event.target` is the deepest element that triggered the event (like an icon inside a button). `event.currentTarget` is the element that actually has the event listener attached to it. When extracting datasets, `currentTarget` is vastly safer.

### Advanced Questions

**Q: How do you update an array reactively in LWC?**
A: While LWC tracks changes to tracked arrays, best practice is to use immutable patterns. Instead of `array.push()`, use the spread operator `array = [...array, newItem]`. Instead of `array.splice()`, use `array = array.filter(...)`. This guarantees the `@api` and rendering lifecycles detect the new memory reference.

**Q: What is the difference between `for:each` and `iterator`?**
A: `for:each` is the standard, cleaner syntax for simple iteration. `iterator` (`iterator:it={array}`) exposes additional boolean properties, specifically `it.first` and `it.last`, allowing you to execute special rendering logic exclusively for the top or bottom elements of the list.

### Architect-Level Questions

**Q: How do you optimize rendering for a list of 5,000 records in LWC?**
A: Rendering 5,000 DOM rows simultaneously is an anti-pattern. As an architect, I would push for Server-Side Pagination (OFFSET/LIMIT in SOQL) to only send chunks of data (e.g., 50 at a time) to the browser. If the data *must* be on the client, I would implement Client-Side Pagination or use `lightning-datatable` with infinite loading to virtualize the DOM rendering.

**Q: Explain the DOM diffing process when a new object is inserted into the middle of an iterated array.**
A: When a new reference is assigned to the array, LWC triggers the render cycle. It compares the existing DOM keys with the new array's keys. It identifies the existing keys and retains their DOM nodes (moving them down). It identifies the new key and synthesizes a new DOM node, injecting it into the specific position without disrupting the state of the surrounding nodes.

---

## 38. Revision Summary

*   **Iteration:** Dynamic rendering of collections into HTML.
*   **`for:each`**: Standard loop. Needs `for:item="name"`.
*   **`for:index`**: Optional. Gets the zero-based loop index.
*   **`key`**: Mandatory on the first element in the loop. Must be unique/stable. Never use index unless data is static.
*   **Reactive Arrays**: Use immutability. Spread `[...arr]` for Add. `map()` for Update. `filter()` for Delete.
*   **Event Handling**: Use `data-id={record.Id}` in HTML.
*   **Target vs CurrentTarget**: Always use `event.currentTarget.dataset.id` to avoid nested element bubbling bugs.
*   **Conditional Rendering**: Put `lwc:if` *inside* the wrapper div containing the key, never on the wrapper itself.
*   **Getters**: Use JS getters to filter or sort lists before they hit the HTML template.
*   **Nested Iteration**: Both outer and inner loops need unique keys.
*   **`iterator`**: Alternative to `for:each`. Access data via `it.value`. Use it ONLY when you need `it.first` or `it.last`.
*   **`lightning-datatable`**: Use for grids/inline-editing. Use `for:each` for custom cards/tiles.
*   **Performance**: Huge arrays crash browsers. Paginate on the server. Always supply stable keys.