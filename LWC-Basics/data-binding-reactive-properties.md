# Data Binding – Reactive Properties

# 1. Introduction

In Salesforce Lightning Web Components (LWC), managing how data is displayed to the user and how user interactions update that data is fundamental to building dynamic applications. This process is governed by **Data Binding** and **Reactivity**.

*   **What Data Binding is:** It is the connection between the component's JavaScript class (data state) and its HTML template (the UI). It determines how variables in JavaScript are rendered in the HTML.
*   **Why Data Binding is important in LWC:** It abstracts away manual DOM manipulation. Instead of writing `document.getElementById('text').innerText = value`, developers declare the relationship, and the framework handles DOM updates automatically.
*   **What Reactivity means:** Reactivity is the engine behind data binding. When a property's value changes in the JavaScript state, a reactive framework automatically detects this change and re-evaluates the HTML template to reflect the new data.
*   **Relationship between state and template:** The JavaScript class holds the "state" (truth), and the HTML template is a visual projection of that state.

**Real-World Example:**
Imagine an Automotive CRM where a Service Agent is viewing a Warranty Claim. The claim's `status` is displayed on the screen. If a background process or an Apex call updates the `status` from 'Draft' to 'Submitted' in the JavaScript controller, LWC's reactivity immediately updates the HTML to show 'Submitted' without any explicit commands to redraw the screen.

---

# 2. What is Data Binding?

**Definition:** Data binding in LWC is the declarative mechanism that maps JavaScript properties and methods to HTML attributes and text nodes. 

LWC utilizes **One-Way Data Binding**. This means data flows in a single, predictable direction: from the JavaScript controller down to the HTML template. 

**UI and JavaScript Relationship:**
The UI is purely a reflection of the JavaScript class. The template cannot autonomously change a JavaScript property. 

**Reactive Rendering Flow Diagram:**

```text
+-------------------------+
|   JavaScript Class      | 
|   (State / Properties)  |
+-------------------------+
            |
            | (Framework detects state change)
            v
+-------------------------+
|     LWC Engine          |
|  (Virtual DOM Diffing)  |
+-------------------------+
            |
            | (Framework updates actual DOM)
            v
+-------------------------+
|     HTML Template       |
|    (Rendered UI)        |
+-------------------------+
```

**Example:**
If we have a JavaScript property `claimNumber = 'CLM-9901'`, data binding allows us to inject `{claimNumber}` into the HTML. When `claimNumber` changes to `'CLM-9902'`, the rendered UI automatically updates.

---

# 3. Reactivity in LWC

**What is a Reactive Property?**
A reactive property is a JavaScript field inside an LWC class that the framework observes. If the framework detects a change in its value, it queues a component rerender. Since LWC Spring '20, **all fields in an LWC class are inherently reactive**.

**How LWC detects changes:**
LWC hooks into JavaScript property assignments. When you use the `=` operator to assign a new value to a property, LWC intercepts this assignment and flags the component as "dirty."

**When a component rerenders:**
Rerendering does not happen instantly on every single line of code execution. LWC batches updates. When a reactive property changes, LWC schedules a microtask to rerender the component. Once the current synchronous JavaScript execution finishes, the engine recalculates the DOM and updates the UI in one efficient pass.

**What causes rerendering:**
*   Assigning a new value to a primitive property (String, Number, Boolean).
*   Reassigning an object or array to a property (changing its memory reference).
*   Updating a property decorated with `@api` or `@track`.

**What does NOT cause rerendering:**
*   Mutating a property inside an un-tracked object/array (e.g., `this.myObj.name = 'New'` does not trigger a render unless `myObj` is decorated with `@track`).
*   Declaring a local variable inside a method (local variables are not component state).

---

# 4. One-Way Data Binding

LWC strictly enforces one-way data binding from **JavaScript → HTML**. To bind data, LWC uses **Template Expressions**.

**Template Expressions:** Denoted by curly braces `{}`. They evaluate a property or getter from the JavaScript class and inject the stringified result into the DOM.

**Example Implementation:**

**HTML (`dataBindingExample.html`):**
```html
<template>
    <!-- Text Property Binding -->
    <p>{message}</p>
</template>
```

**JavaScript (`dataBindingExample.js`):**
```javascript
import { LightningElement } from 'lwc';

export default class DataBindingExample extends LightningElement {
    // A standard reactive primitive property
    message = 'Hello Salesforce';
}
```

**Line-by-Line Explanation:**
*   `<template>`: The root tag required for all LWC HTML files.
*   `<p>{message}</p>`: The `<p>` tag will display the evaluated value of the `message` property. The `{}` syntax tells the LWC engine to bind to the JS state.
*   `import { LightningElement } ...`: Imports the core class required to build an LWC.
*   `export default class DataBindingExample...`: Defines and exports the component class.
*   `message = 'Hello Salesforce';`: Declares a class field. Because it's a class field, LWC makes it reactive.

---

# 5. Template Expressions

Template expressions (`{}`) connect the HTML to the JS. 

**Capabilities & Limitations:**
| Feature | Supported? | Example |
| :--- | :---: | :--- |
| **Property Binding** | ✅ | `{vehicleName}` |
| **Getter Binding** | ✅ | `{formattedPrice}` |
| **Inline Math** | ❌ | `{price * tax}` (Fails) |
| **Inline String Concatenation** | ❌ | `{firstName + ' ' + lastName}` (Fails) |
| **Inline Ternary Logic** | ❌ | `{isDefective ? 'Yes' : 'No'}` (Fails) |

**Why complex JS expressions are banned in templates:**
LWC enforces a strict separation of concerns. Templates should only dictate *structure* and *presentation*, while JavaScript should dictate *logic*. This restriction makes templates highly performant and easily parsable by the framework, reducing rendering bottlenecks.

**Bad Example (Will not compile):**
```html
<p>Total: {price * 1.2}</p>
```

**Good Example:**
```html
<p>Total: {totalPrice}</p>
```
*(With `totalPrice` computed in JavaScript via a getter).*

---

# 6. Binding HTML Attributes

Just as you can bind text inside a tag, you can bind HTML attributes to dynamically control the behavior and styling of elements.

**Common Attribute Bindings:**
*   `disabled={isDisabled}`
*   `title={helpText}`
*   `value={inputValue}`
*   `class={dynamicClass}`

**Production-Quality Example:**

**HTML:**
```html
<template>
    <button 
        class={submitButtonClass} 
        disabled={isSubmitDisabled} 
        title={buttonTitle}
        onclick={handleSubmit}>
        Submit Warranty Claim
    </button>
</template>
```

**JavaScript:**
```javascript
import { LightningElement } from 'lwc';

export default class WarrantySubmitBtn extends LightningElement {
    hasErrors = true;

    // Dynamically assigns classes based on state
    get submitButtonClass() {
        return this.hasErrors ? 'slds-button slds-button_destructive' : 'slds-button slds-button_brand';
    }

    // Evaluates to a boolean to control the 'disabled' attribute
    get isSubmitDisabled() {
        return this.hasErrors;
    }

    // Dynamic title text
    get buttonTitle() {
        return this.hasErrors ? 'Please fix errors before submitting' : 'Click to submit claim';
    }
}
```

---

# 7. Binding Input Values

To create interactive forms, we must bind the HTML input state to the JavaScript state. This requires binding the `value` attribute and listening to the `onchange` event.

**Example:**
```html
<template>
    <lightning-input
        label="Customer Name"
        value={customerName}
        onchange={handleNameChange}>
    </lightning-input>
    <p>Entered Name: {customerName}</p>
</template>
```

```javascript
import { LightningElement } from 'lwc';

export default class CustomerForm extends LightningElement {
    customerName = 'John Doe'; // Initial State

    handleNameChange(event) {
        // Update JS state with the new value from the UI
        this.customerName = event.target.value; 
    }
}
```

**How data moves:**
1.  **JS → UI (Initial render):** `customerName` ('John Doe') flows into the `lightning-input`'s `value` attribute.
2.  **User Interaction:** The user types 'Jane Doe'.
3.  **UI → JS (Event):** The `onchange` event fires, executing `handleNameChange`.
4.  **State Update:** `event.target.value` ('Jane Doe') is assigned to `this.customerName`.
5.  **Reactivity (JS → UI):** LWC detects the assignment, rerenders the template, and updates the `<p>` tag and input value accordingly.

---

# 8. Two-Way Data Flow in LWC

LWC does **not** support automatic two-way data binding (like `[(ngModel)]` in Angular or `v-model` in Vue). 

Instead, it achieves two-way data flow via the **Data Down, Actions Up** pattern.

**How it works:**
1.  **Data Down:** JavaScript injects state into HTML via template expressions (One-Way Binding).
2.  **Actions Up:** HTML triggers JavaScript methods via DOM events (e.g., `onclick`, `onchange`).
3.  **State Mutation:** The JS method updates the state.
4.  **Re-render:** LWC automatically flows the new data down to the HTML.

**Clear Flow Diagram:**
```text
      [ LWC JavaScript Controller ] 
      |                           ^
      | (1) Binds Value Down      | (4) Updates Property
      v                           |
[ HTML Template ]             [ Event Handler ]
      |                           ^
      | (2) User types            | (3) Fires 'onchange' Event
      v                           |
   [ Browser DOM (lightning-input) ]
```

---

# 9. Primitive Property Reactivity

Primitives (String, Number, Boolean, Null, Undefined) are the simplest forms of reactive state. 

Assigning a **new value** to a primitive property using the `=` operator triggers a rerender.

**Example:**
```javascript
import { LightningElement } from 'lwc';

export default class StatusIndicator extends LightningElement {
    message = 'Claim Drafted'; // String primitive

    handleClick() {
        this.message = 'Claim Submitted'; // Reassignment
    }
}
```

**Execution Flow:**
1. Component loads. `message` holds `'Claim Drafted'`. UI renders this text.
2. User clicks a button, invoking `handleClick()`.
3. The LWC engine intercepts `this.message = 'Claim Submitted'`.
4. The engine compares the new value to the old value. They differ.
5. The component is marked dirty.
6. The UI rerenders, displaying 'Claim Submitted'.

---

# 10. Object Reactivity

Objects store complex state. By default (since Spring '20), objects assigned to fields are reactive at the **reference level**.

**Object References vs Property Mutation:**
If you change a property *inside* an object (mutation), the memory reference of the object itself does not change. Therefore, unless instructed otherwise, LWC might not detect the deep mutation. 

**Comparison:**

| Approach | Code | Triggers Render? | Explanation |
| :--- | :--- | :---: | :--- |
| **Mutation (Bad)** | `this.user.name = 'John';` | ❌ | LWC checks if `this.user` is a new object. It isn't. The reference is identical. UI does not update (unless `@track` is used). |
| **Reassignment (Good)** | `this.user = { ...this.user, name: 'John' };` | ✅ | A brand new object is created and assigned to `this.user`. LWC sees a new memory reference and rerenders. |

**Why Immutable Updates (Reassignment) are preferred:**
Using the spread operator (`...`) to create a new object prevents side-effects, makes state transitions predictable, and automatically triggers LWC's default reactivity without needing extra decorators.

---

# 11. Array Reactivity

Arrays behave exactly like objects in memory. Array methods that *mutate* the array in place do not change the array's memory reference.

**Mutating Methods (Do NOT trigger default reactivity):**
*   `push()`
*   `pop()`
*   `splice()`
*   `shift()`

**Non-Mutating / Reassignment Methods (DO trigger reactivity):**
*   Spread operator (`[...array]`)
*   `map()`
*   `filter()`

**Comparison:**

| Approach | Code | Triggers Render? |
| :--- | :--- | :---: |
| **Mutation** | `this.items.push(newItem);` | ❌ (Without `@track`) |
| **Reassignment** | `this.items = [...this.items, newItem];` | ✅ |

**Array Reactivity Example:**
```javascript
// Adding an item to a list of warranty claim line items
addLineItem(newItem) {
    // BAD: Modifies existing array. UI won't update.
    // this.claimLines.push(newItem); 

    // GOOD: Creates a new array. UI updates immediately.
    this.claimLines = [...this.claimLines, newItem]; 
}
```

---

# 12. @track

**What @track is:** A decorator used to make the framework observe deep mutations inside complex objects and arrays.

**Historical Purpose:** Before Spring '20, *no* property was reactive by default. You had to use `@track` on primitive variables, objects, and arrays just to make them reactive.

**Modern LWC Behavior:** Today, all properties are reactive upon *reassignment*. You **do not** need `@track` for primitives, nor do you need it for objects/arrays if you use immutable updates (reassignment).

**When @track is still relevant:**
When you have a deeply nested object or a large array, and you *must* mutate it directly (e.g., `this.items[0].status = 'Approved';`), you must decorate the field with `@track` to force LWC to proxy the object and detect internal mutations.

**Common Misconceptions:**
*   *Myth:* "I need `@track` to make variables show on the UI." -> **False.**
*   *Myth:* "I need `@track` for all objects." -> **False.** (Only needed if you intend to mutate them directly instead of reassigning).

---

# 13. Getters

Getters are JavaScript methods that act as computed properties. They return a dynamically calculated value based on other state properties.

**Purpose:** Since we cannot write logic like `{firstName + ' ' + lastName}` in HTML, we use getters to perform that logic in JavaScript.

**Example:**
```javascript
export default class CustomerHeader extends LightningElement {
    firstName = 'John';
    lastName = 'Doe';

    get fullName() {
        return `${this.firstName}${this.lastName}`;
    }
}
```
**Line-by-Line:**
*   `get fullName()`: Declares a getter function named `fullName`. It is accessed in the template simply as `{fullName}` (without parentheses).
*   `return ${this.firstName}${this.lastName}`: Evaluates the current state and returns a combined string.

---

# 14. Getters with Reactive Properties

Getters are naturally reactive. LWC tracks the reactive properties that a getter relies upon. 

**Dependency Tracking:**
If `fullName` uses `this.firstName`, LWC registers `firstName` as a dependency. If `firstName` is updated, LWC knows it must re-evaluate `fullName` and rerender the parts of the template relying on `{fullName}`.

**Flow:**
`this.firstName = 'Jane';` (Property changes)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓
LWC detects `firstName` changed. Checks what depends on it.
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓
`fullName` getter is flagged for re-evaluation.
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓
Template rerenders, displaying 'Jane Doe'.

---

# 15. Setters

Setters are functions that execute when a property is assigned a value. They are used in conjunction with getters to intercept data, validate it, or perform side effects before saving it to a private backing variable.

**Example:**
```javascript
import { LightningElement, api } from 'lwc';

export default class VehicleMileage extends LightningElement {
    _mileage = 0; // Private backing variable

    @api
    get mileage() {
        return this._mileage;
    }

    set mileage(value) {
        // Validation inside setter
        if (value < 0) {
            console.warn('Mileage cannot be negative');
            this._mileage = 0;
        } else {
            this._mileage = value;
        }
    }
}
```
*Note: A getter must always accompany a setter. The getter returns the internal state, and the setter controls how the state is updated.*

---

# 16. @api Reactive Properties

The `@api` decorator exposes a property publicly, allowing a parent component to pass data down into the child component. 

**Reactivity of @api:**
Public properties are completely reactive. When a parent component passes a new value into an `@api` property, the child component automatically detects the change and rerenders.

**Example:**
```javascript
// childComponent.js
import { LightningElement, api } from 'lwc';

export default class ChildComponent extends LightningElement {
    @api recordId; // Exposed to parent or Lightning Page
}
```
```html
<!-- parentComponent.html -->
<template>
    <!-- When parentRecordId changes in parent, child updates automatically -->
    <c-child-component record-id={parentRecordId}></c-child-component>
</template>
```

---

# 17. Reactive Wire Parameters

The wire service provisions data to an LWC from Salesforce. To make the query dynamic (e.g., fetching data based on the current record), we use reactive wire parameters.

**Example:**
```javascript
import { LightningElement, api, wire } from 'lwc';
import getVehicleData from '@salesforce/apex/VehicleController.getVehicleData';

export default class VehicleViewer extends LightningElement {
    @api recordId; // e.g., '001...'

    @wire(getVehicleData, { vehicleId: '$recordId' })
    vehicleInfo;
}
```

**Explanation:**
*   `$`: The dollar sign prefix tells the `@wire` adapter that `recordId` is a reactive parameter.
*   **How it works:** When `this.recordId` changes, the wire service detects the change via the `$` prefix, automatically re-executes the `getVehicleData` Apex method with the new ID, and provisions the fresh data to `vehicleInfo`.
*   **Difference:** Without `$`, the parameter is passed once as a static string `'$recordId'`, which would fail.

---

# 18. Conditional Rendering

Modern LWC handles conditional rendering using the `lwc:if`, `lwc:elseif`, and `lwc:else` directives. These evaluate a reactive boolean expression to insert or remove DOM nodes.

*(Note: The legacy `if:true` / `if:false` directives are deprecated and should not be used in new development).*

**Example:**
```html
<template>
    <template lwc:if={isClaimApproved}>
        <p class="success">Warranty Claim Approved!</p>
    </template>
    <template lwc:elseif={isClaimRejected}>
        <p class="error">Warranty Claim Rejected.</p>
    </template>
    <template lwc:else>
        <p>Claim is currently Under Review.</p>
    </template>
</template>
```
When the boolean properties `isClaimApproved` or `isClaimRejected` change in JS, the LWC engine reactively mounts or unmounts these DOM elements.

---

# 19. Iteration and Reactive Arrays

To render lists of data, use `for:each` and `for:item`.

**Example:**
```html
<template>
    <ul>
        <template for:each={claimLineItems} for:item="lineItem">
            <li key={lineItem.Id}>
                {lineItem.PartName} - Quantity: {lineItem.Quantity}
            </li>
        </template>
    </ul>
</template>
```

**Why the `key` attribute is critical:**
The `key` provides a unique identifier (like an Id) for each DOM element in the list. When the reactive array changes (e.g., reordered, items added/removed), the LWC engine uses these keys to efficiently diff the virtual DOM. Instead of destroying and rebuilding the entire list, it only moves, adds, or removes the specific elements that changed, heavily optimizing rendering performance.

---

# 20. Reactive Forms

**Scenario: Warranty Claim Form**

**HTML:**
```html
<template>
    <lightning-card title="New Warranty Claim">
        <div class="slds-p-around_medium">
            <lightning-input 
                name="vin" 
                label="Vehicle VIN" 
                value={claimData.vin} 
                onchange={handleInputChange}>
            </lightning-input>
            
            <lightning-input 
                name="mileage" 
                type="number"
                label="Mileage" 
                value={claimData.mileage} 
                onchange={handleInputChange}>
            </lightning-input>

            <p class="slds-m-top_medium">
                Claim Status: <strong>{claimEligibility}</strong>
            </p>
        </div>
    </lightning-card>
</template>
```

**JavaScript:**
```javascript
import { LightningElement } from 'lwc';

export default class WarrantyClaimForm extends LightningElement {
    // Centralized form state
    claimData = {
        vin: '',
        mileage: 0
    };

    // Central change handler for multiple inputs
    handleInputChange(event) {
        const fieldName = event.target.name;
        const fieldValue = event.target.value;

        // Immutable update pattern to trigger reactivity
        this.claimData = { 
            ...this.claimData, 
            [fieldName]: fieldValue 
        };
    }

    // Computed derived state
    get claimEligibility() {
        if (!this.claimData.vin) return 'Awaiting VIN';
        if (this.claimData.mileage > 60000) return 'Out of Warranty (Over Mileage)';
        return 'Eligible for Evaluation';
    }
}
```
**Explanation:** 
Using `name` on inputs allows a single handler (`handleInputChange`) to dynamically update the correct property in the `claimData` object. We use the spread operator to create a new object, ensuring LWC's reactivity triggers the `claimEligibility` getter to re-evaluate based on live data.

---

# 21. Data Binding with Salesforce Data

Reactivity extends seamlessly to Salesforce data via Lightning Data Service (LDS) and Apex.

When `@wire` provisions new data, it directly triggers a component render.

**Example:**
```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import STATUS_FIELD from '@salesforce/schema/WorkOrder.Status';

export default class WorkOrderStatus extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: [STATUS_FIELD] })
    workOrder;

    // Getter reacts instantly when the wire service provisions new data
    get currentStatus() {
        return getFieldValue(this.workOrder.data, STATUS_FIELD);
    }
}
```

---

# 22. Component-to-Component Data Flow

**Parent ↓ Child (Properties):**
The parent communicates to the child by passing data into the child's `@api` public properties. This is a top-down one-way binding.
`<c-vehicle-card vehicle-data={car}></c-vehicle-card>`

**Child ↓ Parent (Events):**
Because binding is strictly one-way, a child **cannot** update a parent's property directly. To send data back up, the child dispatches a `CustomEvent`, and the parent listens for it.
*Child:* `this.dispatchEvent(new CustomEvent('statuschange', { detail: 'Approved' }));`
*Parent:* `<c-vehicle-card onstatuschange={handleStatus}></c-vehicle-card>`

---

# 23. Common Reactivity Problems

| Problem | Cause | Solution | Example Fix |
| :--- | :--- | :--- | :--- |
| **UI not updating for Object** | Mutating internal properties directly. | Reassign object (or add `@track`). | `this.obj = {...this.obj, val: 2}` |
| **Array list not showing new items** | Using `push()` without `@track`. | Reassign array via spread operator. | `this.list = [...this.list, item]` |
| **Wire adapter not firing** | Missing `$` on parameter. | Add `$` to make it reactive. | `{ recordId: '$recordId' }` |
| **List rendering erratically** | Missing or non-unique `key` in `for:each`. | Bind `key` to a unique ID (like SFDC Id). | `<li key={rec.Id}>` |
| **Template string not parsing** | Using logic inside HTML `{a + b}`. | Move logic to a JS getter. | `get total() { return this.a + this.b; }` |

---

# 24. Performance Considerations

*   **Excessive Rerendering:** If you update multiple reactive properties sequentially in synchronous code, LWC optimizes this by batching them into a single render. However, async property updates (in loops or separate promises) can cause multiple renders.
*   **Expensive Getters:** Getters run *every time* the component renders, even if the render was triggered by an unrelated property. Do not put heavy calculations, array sorting, or data manipulation inside a getter. Pre-calculate these in event handlers or wire handlers.
*   **Immutable Updates:** While reassigning large objects triggers reactivity easily, completely cloning massive arrays (10,000+ items) via spread operator can be memory-heavy. In these extreme edge cases, `@track` with direct mutation is more performant.
*   **DOM Updates:** Rely on `lwc:if` to completely remove heavy DOM trees when they aren't needed, rather than hiding them via CSS `display: none`.

---

# 25. Real Project Scenarios

**Scenario 1: Warranty Claim Form (State & Reactive Inputs)**
*   *Component State:* `claimDetails` object containing customer, VIN, issue description.
*   *UI Flow:* Inputs bind to object via `handleFieldChange`. Object is immutably updated.
*   *Submit:* Sends `this.claimDetails` to Apex.

**Scenario 2: Dynamic Product Lists (Arrays & Iteration)**
*   *Component State:* `sparePartsList` array.
*   *UI Flow:* User searches for a part. Apex returns list. `this.sparePartsList` is assigned the result. `for:each` renders the cards.
*   *Keys:* Uses `Product2.Id` for the loop key ensuring fast updates.

**Scenario 3: Dealer Information (Wire & Getters)**
*   *Component State:* `@api recordId`, `@wire` provisioning Account details.
*   *UI Flow:* Reactive `$recordId` triggers wire. Wire populates `accountRecord`. Getters format address fields nicely before injecting them into the template.

---

# 26. Common Mistakes

1.  **Assuming Two-Way Binding:** Trying to let the child update the parent's property via `this.apiProperty = 'new'` (will throw an error). Always use events.
2.  **Using Unnecessary `@track`:** Putting `@track` on primitive variables (String/Boolean). This does nothing and clutters code.
3.  **Expensive Getters:** Doing `JSON.parse(JSON.stringify(data))` inside a getter. This will crash app performance as it runs on every render cycle.
4.  **Forgetting the `$`:** Writing `@wire(getContact, { id: 'recordId' })`. The wire will fire once with the literal string "recordId" and fail.

---

# 27. Best Practices Checklist

*   ✅ **Prefer simple reactive properties:** Use primitive types where possible. They are cheap and predictably reactive.
*   ✅ **Keep component state predictable:** Group related form data into single objects rather than dozens of separate primitive properties.
*   ✅ **Use getters for derived values:** If a value can be computed from existing state, don't store it in a new variable. Derive it in a getter.
*   ✅ **Use immutable update patterns where appropriate:** `this.obj = { ...this.obj, newProp: 'val' }` prevents hidden state bugs.
*   ✅ **Avoid unnecessary `@track`:** Let LWC's default reactivity handle reassignment. Only use `@track` for deep mutations of complex data.
*   ✅ **Keep template expressions simple:** Logic belongs in JS. Templates are just a display layer.
*   ✅ **Use reactive wire parameters correctly:** Always remember the `$` for dynamic data fetching.
*   ✅ **Keep components small and focused:** Smaller components have smaller DOM trees, making virtual DOM diffing (and thus reactivity) lightning fast.

---

# 28. Interview Questions & Answers

### Beginner Questions

**Q: What is one-way data binding in LWC?**
A: It is the paradigm where data flows from the JavaScript controller down to the HTML template. The template displays data but cannot directly alter the JS property without firing an event.

**Q: Does LWC support two-way data binding?**
A: No, not automatically. It uses one-way data binding combined with DOM event listeners (like `onchange`) to achieve a two-way data flow (Data Down, Actions Up).

### Intermediate Questions

**Q: What causes an LWC component to rerender?**
A: A component rerenders when a primitive property is assigned a new value, an object/array is reassigned to a new memory reference, an `@api` property receives new data from a parent, or an internal property of an `@track` decorated object is mutated.

**Q: How do you update an array reactively?**
A: The best way is to reassign the array using the spread operator: `this.myArray = [...this.myArray, newItem];`. Alternatively, if the array is decorated with `@track`, you can use `this.myArray.push(newItem);`.

### Advanced Questions

**Q: What is the difference between a property and a getter?**
A: A property holds a static piece of state in memory. A getter is a function invoked as a property that computes and returns a dynamic value on the fly. Getters automatically re-evaluate when the reactive properties they depend on change.

**Q: What does `$` mean in `@wire` parameters?**
A: The `$` prefix marks a parameter as dynamically reactive. If the value of the property prefixed with `$` changes, the wire adapter automatically provisions new data using the new parameter value.

### Architect-Level Questions

**Q: Explain the evolution of `@track` and its current necessity.**
A: Prior to Spring '20, all reactive properties required `@track`. After Spring '20, all fields became natively reactive upon reassignment. Today, `@track` is strictly an opt-in deep-observation decorator. It is only required if an architect decides to mutate deeply nested object properties or use array mutating methods (`splice`, `push`) rather than utilizing immutable reassignment patterns, typically for memory optimization on massive datasets.

---

# 29. Revision Summary

*   **Data Binding:** Connects JS state to HTML template.
*   **Reactivity:** Automatically rerenders UI when JS state changes.
*   **One-Way Binding:** JS state flows down to HTML. UI events flow up to JS.
*   **Template Expressions:** `{property}` injects data. No inline JS allowed.
*   **Primitive Reactivity:** Changing String/Number/Boolean auto-triggers render.
*   **Object/Array Reactivity:** Must reassign (e.g., Spread operator) for native reactivity, or use `@track` for deep mutations.
*   **@track:** Only needed for observing internal mutations of complex data.
*   **Getters/Setters:** Used for computed properties and intercepting state changes.
*   **@api:** Makes public properties reactive to parent updates.
*   **Reactive Wire:** `$` triggers dynamic Apex/LDS calls on property changes.
*   **Conditional Rendering:** Use `lwc:if`, `lwc:elseif`, `lwc:else` (skip `if:true`).
*   **Iteration:** Always provide a unique `key` in `for:each` for DOM efficiency.
*   **Performance:** Avoid heavy logic inside getters. Rely on batched state updates and immutable patterns.