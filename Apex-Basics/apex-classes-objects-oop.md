# Apex Classes Objects OOP

## 1. Introduction

### What is Object-Oriented Programming (OOP)?
Object-Oriented Programming (OOP) is a programming paradigm based on the concept of "objects," which can contain data (in the form of fields or properties) and code (in the form of procedures or methods). Rather than writing procedural code that acts on detached variables, OOP models software around real-world entities.

### Why Salesforce Apex Uses OOP
Apex is a strongly typed, object-oriented programming language executed on the Salesforce Lightning Platform. Its syntax is heavily inspired by Java. Salesforce uses OOP to help developers build scalable, maintainable, and reusable enterprise architectures that naturally map to Salesforce standard and custom objects (SObjects).

### Benefits of OOP in Enterprise Applications
* **Modularity:** Code is organized into manageable pieces (classes).
* **Reusability:** Code can be reused through inheritance and composition.
* **Maintainability:** Changes in one module rarely break others if boundaries (interfaces) are respected.
* **Scalability:** Facilitates complex enterprise patterns (e.g., Service Layer, Unit of Work).

### Procedural vs. Object-Oriented
| Feature | Procedural Programming | Object-Oriented Programming |
| :--- | :--- | :--- |
| **Focus** | Functions and logic | Data and objects |
| **State** | Global or local detached variables | Encapsulated within objects |
| **Reusability** | Low (copy/paste or standalone functions) | High (Inheritance, Polymorphism) |
| **Enterprise Fit** | Poor for large systems | Ideal for scalable applications |

*Real-World Example:* In a procedural setup, you'd have a function `processWarranty(vehicleId, claimAmount)`. In OOP, you have a `WarrantyService` class that interacts with a `Vehicle` object and a `WarrantyClaim` object.

---

## 2. What is a Class?

### Definition and Blueprint Concept
A class is a user-defined blueprint or prototype from which objects are created. It represents the set of properties (state) or methods (behavior) common to all objects of one type.

### Memory Representation
When a class is compiled, the definition is stored in the Salesforce metadata repository. No heap memory is allocated for the data variables until an object of that class is instantiated.

### Components of a Class
* **Variables:** Define the state (e.g., `String VIN`, `Decimal price`).
* **Methods:** Define the behavior (e.g., `calculateDepreciation()`).
* **Constructors:** Special methods used to initialize objects.
* **Inner Classes:** Classes defined within another class.
* **Blocks:** Initialization blocks (rarely used in Apex compared to Java).

```apex
public class Vehicle {
    // Variables
    public String vinNumber;
    public Decimal currentValue;
    
    // Constructor
    public Vehicle(String vin, Decimal value) {
        this.vinNumber = vin;
        this.currentValue = value;
    }
    
    // Method
    public void depreciate(Decimal percentage) {
        this.currentValue -= (this.currentValue * (percentage / 100));
    }
}
```

---

## 3. What is an Object?

### Definition
An object is an instance of a class. It is a real-world entity with a specific state and behavior defined by its class.

### Instance Creation and Heap Memory Allocation
When an object is created using the `new` keyword, Salesforce dynamically allocates memory for it on the **Heap**. The reference to this object (the variable name) is stored on the **Stack**.

### State and Behavior
* **State:** The current values of the object's variables (e.g., `vinNumber = '1HGCM8'`, `currentValue = 25000`).
* **Behavior:** The actions the object can perform (e.g., `depreciate()`).

```apex
// Instantiating the object
Vehicle myCar = new Vehicle('1HGCM8', 25000); 
```

---

## 4. Class Structure in Apex

A standard enterprise Apex class consists of several structural components.

* **Static Members:** Belong to the class itself, not any instance. Loaded once.
* **Constructors:** Initialize the state of an instance.
* **Instance Variables:** State held by individual objects.
* **Instance Methods:** Logic executed by individual objects.
* **Inner Classes:** Used for encapsulation, often as wrapper classes.

**Diagram of Class Structure:**
```text
+---------------------------------------------------+
|               Class: DealerManager                |
|---------------------------------------------------|
| - Static Variables (e.g., cached SOQL results)    |
| - Instance Variables (e.g., Dealer ID, Region)    |
| - Constructors (Default, Parameterized)           |
| - Static Methods (e.g., getDealerByCode())        |
| - Instance Methods (e.g., calculateInventory())   |
| - Inner Class (e.g., InventoryWrapper)            |
+---------------------------------------------------+
```

---

## 5. Creating Objects

### Examples and Line-by-Line Explanation

```apex
// Example 1: Standard SObject
Account acc = new Account(Name = 'Global Motors');
```
* `Account acc`: Declares a reference variable `acc` of type standard SObject `Account` on the Stack.
* `=`: Assignment operator linking the reference to the heap memory address.
* `new`: Keyword that triggers heap memory allocation.
* `Account(Name = 'Global Motors')`: Calls the default SObject constructor and initializes the `Name` field.

```apex
// Example 2: Custom Apex Class
Vehicle myVehicle = new Vehicle('VIN123', 50000);
```
* `Vehicle myVehicle`: Declares reference variable `myVehicle` on the Stack.
* `new`: Allocates Heap memory for a new `Vehicle` instance.
* `Vehicle('VIN123', 50000)`: Calls the parameterized constructor, setting initial state.

---

## 6. Constructors

### Default vs. Parameterized
* **Default Constructor:** Takes no arguments. If you do not define any constructor, Apex provides a no-argument constructor automatically.
* **Parameterized Constructor:** Takes arguments to initialize the object with specific data.

### Constructor Overloading
Having multiple constructors with different parameter lists in the same class.

### Constructor Chaining
Calling one constructor from another within the same class using `this()`. Must be the first line in the constructor.

```apex
public class Dealer {
    public String name;
    public String region;
    
    // Default Constructor
    public Dealer() {
        this('Unknown Dealer', 'Global'); // Constructor Chaining
    }
    
    // Parameterized Constructor (Overloaded)
    public Dealer(String name) {
        this(name, 'Global'); // Constructor Chaining
    }
    
    // Master Parameterized Constructor
    public Dealer(String name, String region) {
        this.name = name;
        this.region = region;
    }
}
```

---

## 7. Instance Variables

### Purpose and Lifecycle
Instance variables store the state of an object. Their lifecycle is tied to the object; they are created when the object is instantiated and destroyed when the object is garbage-collected.

### Memory Allocation
Allocated on the **Heap** for every newly created object.

```apex
public class WarrantyClaim {
    // Instance variable. Unique to EVERY claim created.
    public Decimal claimAmount; 
}
```

---

## 8. Static Variables and Methods

### Static Memory Allocation
Variables and methods marked with `static` belong to the class, not a specific instance. Memory is allocated once per transaction.

### Use Cases
* **Utility Methods:** Methods that don't depend on instance state (e.g., `Math.max()`).
* **Caching:** Storing SOQL results or sets of IDs to prevent querying multiple times in a transaction.
* **Recursion Control:** Using static boolean flags in triggers to prevent infinite loops.

### Comparison: Instance vs. Static

| Feature | Instance | Static |
| :--- | :--- | :--- |
| **Ownership** | Object / Instance | Class |
| **Memory** | Heap (allocated per object) | Static memory (allocated once per transaction) |
| **Access** | `objectReference.member` | `ClassName.member` |
| **Data Sharing**| Unique to each object | Shared across all instances in the transaction |

```apex
public class TransactionUtil {
    // Static variable to control trigger recursion
    public static Boolean isFirstRun = true;
    
    // Static method
    public static String generateTransactionId() {
        return EncodingUtil.convertToHex(Crypto.generateAesKey(128));
    }
}
```

---

## 9. Methods in Apex

### Instance vs. Static Methods
* **Instance Method:** Requires an object. Can access instance variables, `this`, and static variables.
* **Static Method:** Does not require an object. Cannot access instance variables or `this`. Can only access other static members.

### Method Overloading
Creating multiple methods with the *same name* but *different parameter types or counts* within the same class. Return type alone is not sufficient to overload.

```apex
public class InvoiceProcessor {
    
    // Method Overloading
    public void process(Id invoiceId) {
        // process single invoice
    }
    
    public void process(List<Id> invoiceIds) {
        // process bulk invoices
    }
    
    // Method with return type
    public Decimal calculateTax(Decimal amount, Decimal rate) {
        return amount * (rate / 100);
    }
}
```

---

## 10. Encapsulation

### Definition and Data Hiding
Encapsulation is the bundling of data (variables) and methods that operate on that data into a single unit (class). It restricts direct access to some of an object's components, a concept known as data hiding.

### Access Modifiers
| Modifier | Visibility | Enterprise Use Case |
| :--- | :--- | :--- |
| `private` | Only within the same class. | Hiding internal implementation details and helper methods. |
| `protected` | Same class and subclasses. | Base classes in a framework (e.g., Trigger Handlers). |
| `public` | Anywhere within the namespace. | Standard service classes, controllers, and utilities. |
| `global` | Anywhere across namespaces. | Managed packages, REST API classes, Batch classes. |

### Benefits
Protects object state from invalid data injection. Enforces validation rules inside setters.

```apex
public class CustomerProfile {
    // Private state
    private String ssn;
    
    // Public getter
    public String getMaskedSsn() {
        if(String.isBlank(this.ssn)) return null;
        return 'XXX-XX-' + this.ssn.right(4);
    }
    
    // Public setter with validation
    public void setSsn(String ssn) {
        if(ssn != null && ssn.length() == 9) {
            this.ssn = ssn;
        } else {
            throw new IllegalArgumentException('Invalid SSN length');
        }
    }
}
```

---

## 11. Inheritance

### Definition and IS-A Relationship
Inheritance allows a new class (child) to acquire the properties and methods of an existing class (parent). It represents an "IS-A" relationship (e.g., a `Truck` IS-A `Vehicle`).

### Keywords
* `virtual`: Must be added to a parent class/method to allow it to be extended/overridden.
* `extends`: Used by the child class to inherit from the parent.

```apex
// Parent Class
public virtual class Vehicle {
    public Decimal price;
    
    public virtual void startEngine() {
        System.debug('Starting generic vehicle engine...');
    }
}

// Child Class
public class Truck extends Vehicle {
    public Integer towingCapacity;
    
    // Method Overriding
    public override void startEngine() {
        System.debug('Starting heavy-duty diesel engine...');
    }
}
```

---

## 12. Polymorphism

### Definition
Polymorphism means "many forms." It allows objects of different classes to be treated as objects of a common superclass.

### Types of Polymorphism
* **Compile-Time (Static):** Method Overloading. The compiler knows which method to call based on the arguments.
* **Runtime (Dynamic):** Method Overriding. The JVM/Apex runtime determines which method to execute based on the actual object type at runtime, not the reference type.

### Dynamic Binding Example
```apex
Vehicle myRide = new Truck(); // Polymorphism in action
myRide.startEngine(); // Outputs: "Starting heavy-duty diesel engine..."
```
Even though the reference type is `Vehicle`, the runtime knows the actual object is a `Truck` and binds the overridden method.

*Apex limitation vs. Java:* Apex does not support method overloading dynamically through late binding as flexibly as Java does in certain reflection scenarios, but standard inheritance polymorphism works identically.

---

## 13. Abstraction

### Definition
Abstraction is the process of hiding the implementation details and showing only functionality to the user. It focuses on *what* the object does rather than *how* it does it.

### Abstract Classes
A class declared with the `abstract` keyword. It cannot be instantiated directly. It can contain both abstract methods (without a body) and concrete methods (with a body).

```apex
public abstract class PaymentProcessor {
    // Concrete method
    public void logTransaction(String txId) {
        System.debug('Transaction Logged: ' + txId);
    }
    
    // Abstract method - Child MUST implement this
    public abstract Boolean processPayment(Decimal amount);
}

public class CreditCardProcessor extends PaymentProcessor {
    public override Boolean processPayment(Decimal amount) {
        // Implementation details hidden from caller
        return true; 
    }
}
```

---

## 14. Interfaces

### Definition
An interface is a pure abstraction. It is a contract that defining method signatures without implementations. A class that `implements` an interface must define all its methods.

### Multiple Interface Support
While Apex only supports single inheritance (a class can only `extend` one class), a class can implement *multiple* interfaces.

### Abstract Class vs. Interface

| Feature | Abstract Class | Interface |
| :--- | :--- | :--- |
| **Instantiable** | No | No |
| **Methods** | Can have implemented and abstract methods | Only abstract method signatures |
| **Variables** | Can have instance variables | No instance variables |
| **Inheritance**| Single inheritance (`extends`) | Multiple implementation (`implements`) |
| **Use Case** | Sharing base logic (e.g., Trigger framework base class) | Defining capabilities (e.g., `Queueable`, `Schedulable`) |

```apex
public interface IValidatable {
    Boolean isValid();
}

public class ClaimValidator implements IValidatable {
    public Boolean isValid() {
        return true; // Implements the contract
    }
}
```

---

## 15. Virtual Classes and Methods

### virtual vs. override
By default, Apex classes are `final` (cannot be extended). To allow inheritance, use `virtual`. To allow a method to be redefined in a child class, mark the parent method `virtual` and the child method `override`.

*Note: `abstract` methods are implicitly `virtual`.*

---

## 16. Inner Classes

### Wrapper Classes
Inner classes are highly utilized in Salesforce as "Wrapper Classes" to bind an SObject with a boolean flag (for UI selection) or to structure complex JSON payloads for REST integrations.

### Static Inner Classes
In Apex, inner classes are static by default (unlike Java), meaning they do not require an instance of the outer class to be instantiated.

```apex
public class DataTableController {
    
    // Wrapper Class (Inner Class)
    public class DealerWrapper {
        public Account dealerRecord;
        public Boolean isSelected;
        
        public DealerWrapper(Account acc) {
            this.dealerRecord = acc;
            this.isSelected = false;
        }
    }
}
```

---

## 17. `this` Keyword

### Usage
* **Current Object Reference:** Refers to the current instance of the class.
* **Variable Disambiguation:** Resolves shadowing when a method parameter has the same name as an instance variable.
* **Constructor Chaining:** `this()` calls another constructor in the same class.

```apex
public class Part {
    public String partNumber;
    
    public Part(String partNumber) {
        // Disambiguation
        this.partNumber = partNumber; 
    }
}
```

---

## 18. `super` Keyword

### Usage
* **Parent Constructor Calls:** `super()` calls the constructor of the parent class.
* **Parent Method Calls:** `super.methodName()` calls the overridden method in the parent class.

```apex
public class EvVehicle extends Vehicle {
    public EvVehicle(Decimal price) {
        super(price); // Calls Vehicle(Decimal price) constructor
    }
    
    public override void startEngine() {
        super.startEngine(); // Executes parent logic first
        System.debug('Engaging electric motors...');
    }
}
```

---

## 19. Object Lifecycle

1.  **Creation:** Memory allocated on Heap via `new`. Constructor executes.
2.  **Usage:** Object resides in Heap. Stack reference is used to access state/methods.
3.  **Dereferencing:** When the variable goes out of scope or is set to `null`, the object is no longer reachable.
4.  **Garbage Collection:** Salesforce automatically reclaims heap memory for dereferenced objects. Developers have no explicit control over when the Garbage Collector (GC) runs.

---

## 20. Memory Management in Apex

### Heap Memory Limits (Governor Limits)
* **Synchronous:** 6 MB
* **Asynchronous:** 12 MB

### Stack Memory
Used for local variables, primitives, references, and method execution frames. Apex has a strict stack depth limit (usually SObjects/collections count towards heap, but recursive method calls blow up the stack, throwing `Maximum stack depth reached` limit).

### Best Practices for Large Collections
To avoid `System.LimitException: Too many SObject rows` or Heap limit errors, iterate over large queries using a SOQL `for` loop:

```apex
// Bad: Loads all records into Heap memory at once
List<Account> allDealers = [SELECT Id, Name FROM Account]; 

// Good: Processes in chunks (200 records), saving Heap space
for(Account dealer : [SELECT Id, Name FROM Account]) {
    // Process dealer
}
```

---

## 21. Apex-Specific OOP Features

### Sharing Modifiers
Apex runs in system context by default. OOP structures must dictate sharing context explicitly.
* `with sharing`: Enforces User's record-level sharing rules.
* `without sharing`: Ignores sharing rules (runs as System).
* `inherited sharing`: Dynamically assumes the sharing context of the class that called it (Best Practice for Service layer).

### SObject Inheritance
While you cannot write `class MyAccount extends Account`, standard SObjects inherit from the generic `SObject` class, allowing dynamic polymorphism.
```apex
SObject genericRecord = new Account(Name = 'Auto Corp');
genericRecord.put('Name', 'Global Auto Corp');
```

---

## 22. OOP Design Principles (SOLID)

| Principle | Meaning | Apex Enterprise Example |
| :--- | :--- | :--- |
| **S**ingle Responsibility | A class should have one reason to change. | `InvoiceService` only calculates invoices. It does not query SObjects directly (that belongs to a Selector class). |
| **O**pen-Closed | Open for extension, closed for modification. | Using interfaces/virtual classes to add new `PaymentTypes` without altering core `PaymentProcessor`. |
| **L**iskov Substitution | Subclasses must be substitutable for base classes. | `ElectricVehicle` should be able to replace `Vehicle` anywhere without causing exceptions. |
| **I**nterface Segregation | Many client-specific interfaces are better than one general interface. | Split `IUserActions` into `IDealerActions` and `ICustomerActions`. |
| **D**ependency Inversion | Depend on abstractions, not concretions. | `OrderManager` depends on `IPaymentStrategy`, not `CreditCardProcessor` directly. |

---

## 23. Enterprise Design Patterns

### Service Layer Pattern
Encapsulates complex business logic away from Triggers, Controllers, and Batch classes. Focuses on the "S" in SOLID.

### Factory Pattern
Used to instantiate objects based on dynamic inputs. Reduces tight coupling.
```apex
public class VehicleFactory {
    public static Vehicle getVehicle(String type) {
        if(type == 'TRUCK') return new Truck();
        if(type == 'SEDAN') return new Sedan();
        return null;
    }
}
```

### Singleton Pattern
Ensures a class has only one instance and provides a global point of access to it. Excellent for caching RecordTypes or limits.
```apex
public class RecordTypeCache {
    private static RecordTypeCache instance;
    private Map<String, Id> rtMap;
    
    private RecordTypeCache() { /* private constructor */ }
    
    public static RecordTypeCache getInstance() {
        if(instance == null) {
            instance = new RecordTypeCache();
            instance.loadCache();
        }
        return instance;
    }
}
```

---

## 24. Classes and Objects in Triggers

### Trigger Handler Pattern
Triggers should contain **NO logic**. They should instantiate an object of a TriggerHandler class and delegate execution.

```apex
// Trigger (Procedural entry point)
trigger WarrantyClaimTrigger on WarrantyClaim__c (before insert, after insert) {
    WarrantyClaimTriggerHandler handler = new WarrantyClaimTriggerHandler();
    
    if(Trigger.isBefore && Trigger.isInsert) {
        handler.onBeforeInsert(Trigger.new);
    }
}

// Handler Class (OOP Logic)
public class WarrantyClaimTriggerHandler {
    public void onBeforeInsert(List<WarrantyClaim__c> newClaims) {
        WarrantyClaimService.validateClaims(newClaims);
    }
}
```

---

## 25. Classes and Objects in LWC

### Apex Controllers and DTOs (Data Transfer Objects)
LWC communicates with Apex via `@AuraEnabled` methods. Instead of returning massive SObjects with unused fields, create OOP Wrapper Classes (DTOs) to serialize exactly what the UI needs.

```apex
public with sharing class DealerController {
    
    @AuraEnabled(cacheable=true)
    public static DealerDTO getDealerInfo(Id dealerId) {
        Account acc = [SELECT Id, Name, BillingState FROM Account WHERE Id = :dealerId];
        return new DealerDTO(acc); // Serialization handles object to JSON
    }
    
    // DTO Wrapper Class
    public class DealerDTO {
        @AuraEnabled public String id;
        @AuraEnabled public String title;
        
        public DealerDTO(Account acc) {
            this.id = acc.Id;
            this.title = acc.Name + ' - ' + acc.BillingState;
        }
    }
}
```

---

## 26. Real Project Scenarios: Automotive CRM

Let's tie it all together in an enterprise context: Processing a Warranty Claim with SAP Integration.

**1. The Interface (Abstraction)**
```apex
public interface IClaimStrategy {
    void process(List<WarrantyClaim__c> claims);
}
```

**2. The Implementations (Polymorphism & Strategy Pattern)**
```apex
public class EngineClaimStrategy implements IClaimStrategy {
    public void process(List<WarrantyClaim__c> claims) {
        // High scrutiny, requires SAP parts validation
        SAPIntegrationService.verifyEngineParts(claims);
    }
}

public class CosmeticClaimStrategy implements IClaimStrategy {
    public void process(List<WarrantyClaim__c> claims) {
        // Auto-approve if under $500
        ClaimValidationStrategy.autoApproveLowValue(claims);
    }
}
```

**3. The Factory**
```apex
public class ClaimStrategyFactory {
    public static IClaimStrategy getStrategy(String category) {
        if(category == 'Engine') return new EngineClaimStrategy();
        if(category == 'Cosmetic') return new CosmeticClaimStrategy();
        throw new IllegalArgumentException('Invalid Category');
    }
}
```

**4. The Service Layer (Putting it together)**
```apex
public inherited sharing class WarrantyClaimService {
    public static void routeClaims(List<WarrantyClaim__c> claims) {
        for(WarrantyClaim__c claim : claims) {
            IClaimStrategy strategy = ClaimStrategyFactory.getStrategy(claim.Category__c);
            strategy.process(new List<WarrantyClaim__c>{claim});
        }
    }
}
```
*Design Choice:* The Service layer knows *nothing* about SAP or auto-approvals. It just asks the Factory for an object, and Polymorphism ensures the correct `.process()` method runs.

---

## 27. Performance Considerations

* **Object Creation Costs:** Instantiating thousands of complex Apex objects in a `for` loop consumes CPU time. Pre-allocate or reuse objects where possible.
* **Static Caching:** Use static variables to cache repeated query results (e.g., getting Record Types by DeveloperName).
* **Pass by Reference:** In Apex, objects are passed to methods by reference. Modifying an object inside a method modifies the original object. This is memory-efficient as objects aren't duplicated.
* **Heap Limit Avoidance:** Set large collections and DTOs to `null` after use to flag them for Garbage Collection earlier in long-running batches.

---

## 28. Best Practices

* **Encapsulation:** Always default to `private` variables with public getters/setters if external access is needed.
* **Loose Coupling:** Depend on Interfaces, not concrete classes.
* **High Cohesion:** A class should do one thing very well (Single Responsibility). Don't mix REST API parsing with SObject DML in the same class.
* **Service Layers:** Centralize business logic in `Service` classes.
* **Inherited Sharing:** Always use `inherited sharing` on Utility and Service classes to prevent data leakage in contexts where standard sharing should apply.

---

## 29. Common Mistakes

| Mistake | Description | Solution |
| :--- | :--- | :--- |
| **God Classes** | One massive class handling triggers, emails, and callouts. | Refactor using SOLID principles; separate into Handlers, Services, and Utilities. |
| **Tight Coupling** | `new SAPService()` hardcoded everywhere. | Use Dependency Injection or Factory patterns. |
| **Overusing Statics**| Using static lists to store data across records, causing cross-contamination. | Use instance variables for state; use static only for constants or caches. |
| **Null References** | Trying to call methods on uninstantiated objects. | Always instantiate lists `List<String> x = new List<String>();` before adding. |

---

## 30. Debugging OOP Code

* **Developer Console:** Use anonymous Apex to instantiate objects and test isolated methods without triggering full system processes.
* **Heap Inspection:** Set debug log levels for `HEAP_ALLOCATE` to track which objects are consuming memory.
* **System.debug():** When debugging an object state, `System.debug(myVehicle)` prints the entire state of the object implicitly calling its properties.
* **Limits Class:** Use `Limits.getHeapSize()` and `Limits.getLimitHeapSize()` inside complex object iterations to proactively prevent crashes.

---

## 31. Interview Questions & Answers

### Beginner
**Q: What is the difference between an Object and a Class?**
**A:** A class is the logical blueprint or template stored in metadata. An object is a physical instance of that class, allocated in heap memory at runtime containing specific data.

### Intermediate
**Q: How does `this` differ from `super` in Apex?**
**A:** `this` refers to the current instance of the class and is used to resolve naming collisions or chain constructors. `super` is used to invoke the constructor or overridden methods of the parent class in an inheritance hierarchy.

### Advanced
**Q: Can we implement method overloading based solely on the return type in Apex?**
**A:** No. Method overloading requires a difference in the method signature (number, type, or order of parameters). The return type is not part of the signature used by the compiler to resolve method calls.

### Architect-Level
**Q: How do you prevent Heap Limit exceptions in a complex object-oriented architecture where deep object graphs are created from SObjects?**
**A:** Use the SOQL `for` loop to process records in chunks. Implement the Flyweight or Proxy patterns for related data. Aggressively clear out maps and lists by setting them to `null` once processed. Use SObject fields directly rather than creating heavy DTO/Wrapper classes if the wrapper isn't strictly necessary for the layer boundary.

---

## 32. Revision Summary

* **Classes/Objects:** Blueprints vs. Memory-allocated instances.
* **Constructors:** Initialize object state; support chaining (`this()`).
* **Methods:** Instance (requires object) vs. Static (class-level).
* **Encapsulation:** Hiding state using `private`, exposing via getters/setters.
* **Inheritance:** "IS-A" relationships using `virtual`, `extends`, `override`.
* **Polymorphism:** Method Overloading (Compile) and Overriding (Runtime).
* **Abstraction/Interfaces:** Hiding implementation details using `abstract` and `implements`.
* **Modifiers:** `with/without/inherited sharing`, `private/public/global`.
* **SOLID & Patterns:** Essential for maintainable enterprise architecture (Service, Factory, Singleton).
* **Memory:** Pay close attention to Heap Limits (6MB/12MB) and SObject loop structures.