# REST API – Create REST Services

---

# 1. Introduction

**Application Programming Interfaces (APIs)** act as bridges that allow distinct software systems to communicate, share data, and execute operations seamlessly. A **REST API (Representational State Transfer)** is an architectural style of API that leverages standard HTTP protocols, making it lightweight, scalable, and ideal for web services. 

Integrations are the backbone of modern enterprise architectures. No system lives in isolation. Salesforce provides a robust suite of REST APIs to ensure it can act as a central hub in a multi-cloud, interconnected enterprise. 

**Real-world enterprise integration scenarios:**
* Syncing Automotive CRM Warranty Claims to a backend SAP ERP system.
* Allowing a customer-facing mobile application to surface real-time service appointments.
* Connecting Salesforce with a third-party payment gateway (e.g., Stripe) to process vehicle deposit payments.

---

# 2. What is REST?

**REST (Representational State Transfer)** is a software architectural style dictating how web standards should be used. It relies heavily on HTTP methods and JSON/XML payloads.

### REST Principles
* **Client-Server Architecture:** Clear separation of concerns. The client handles the UI and user state; the server handles data processing and storage.
* **Stateless Communication:** Every request from the client must contain all information needed by the server to fulfill that request. The server does not store client context between requests.
* **Cacheable:** Responses must implicitly or explicitly define themselves as cacheable or not to prevent clients from reusing stale data.
* **Uniform Interface:** A standardized way of communicating (using standard HTTP methods, URIs, and media types).
* **Layered System:** A client cannot ordinarily tell whether it is connected directly to the end server or an intermediary (like a load balancer).

**Resources and Endpoints:**
In REST, data entities are treated as **Resources** (e.g., a `Vehicle` or `Warranty Claim`). An **Endpoint** is the specific URI (Uniform Resource Identifier) where that resource can be accessed (e.g., `/services/apexrest/Warranty/`).

---

# 3. Why REST APIs are Needed

In an enterprise environment, REST APIs are essential for:
* **Third-party integrations:** Connecting Salesforce to external tools (Jira, ServiceNow).
* **Mobile & Web Applications:** Headless architectures where Salesforce acts as the database/backend.
* **ERP Integration (e.g., SAP/Oracle):** Real-time synchronization of inventory, invoices, and logistics.
* **Payment Gateways:** Processing secure transactions.
* **Data Synchronization:** Nightly delta syncs between legacy on-premise databases and Salesforce.

---

# 4. Salesforce REST APIs

Salesforce offers multiple out-of-the-box REST APIs tailored for specific use cases.

| API Name | Description | Use Case |
| :--- | :--- | :--- |
| **Standard REST API** | Core API for interacting with standard and custom objects. | Basic CRUD, querying records, retrieving metadata. |
| **Composite API** | Executes multiple REST API requests in a single call. | Complex data trees, reducing API call consumption. |
| **Bulk API (2.0)** | Asynchronous REST API for large data volumes. | Migrating/updating millions of records. |
| **Tooling API** | Integrates with Salesforce metadata for dev tools. | Building custom development tools or IDEs. |
| **Metadata API** | Used to deploy and retrieve customizations. | CI/CD pipelines, package deployment. |
| **Connect REST API** | Access B2B Commerce, CMS, Chatter, and Communities. | Building custom UI for Chatter or Commerce. |
| **UI API** | Retrieves data and metadata to build dynamic UIs. | Building custom LWC apps off-platform. |
| **GraphQL API** | Single endpoint for tailored data queries. | Reducing over-fetching of data on mobile. |

---

# 5. Standard REST API vs Custom Apex REST API

| Feature | Standard REST API | Custom Apex REST API |
| :--- | :--- | :--- |
| **Setup Required** | None (Out-of-the-box) | Requires Apex Code & Test Classes |
| **Complexity** | Simple CRUD | Complex, multi-object transactional logic |
| **Data Payload** | Dictated by Salesforce | Fully customizable via Wrapper classes |
| **Transaction Control**| Single record or predefined composites | Full control (Rollbacks, Savepoints) |
| **Best For** | External apps reading/writing simple records | Orchestrating complex business processes |

**Recommendation:** Always use the Standard REST API or Composite API if the requirement is simple CRUD. Fall back to Custom Apex REST API only when complex logic, custom validation, or specific transactional control is required.

---

# 6. Creating a Custom REST API

Creating an Apex REST service requires exposing a global Apex class.

```java
@RestResource(urlMapping='/Warranty/*')
global with sharing class WarrantyAPI {
    
    @HttpGet
    global static void getWarrantyStatus() {
        // Implementation
    }
}
```

**Line-by-line explanation:**
* `@RestResource(urlMapping='/Warranty/*')`: Exposes this class as a REST resource. The wildcard `*` means it captures any URI under `/Warranty/`.
* `global with sharing class WarrantyAPI`: Must be `global`. `with sharing` enforces the user's sharing rules (Security Best Practice).
* `@HttpGet`: Annotates the method to listen for HTTP GET requests.
* `global static void getWarrantyStatus()`: Methods must be `global static`.

---

# 7. @RestResource Annotation

The `@RestResource` annotation is the entry point for custom APIs.
* **Syntax:** `@RestResource(urlMapping='/YourPath/*')`
* **URL Mapping:** Defines the endpoint relative to the Salesforce instance URL. The full path becomes `https://[domain].my.salesforce.com/services/apexrest/YourPath/`.
* **Wildcards (`*`):** Allow for dynamic URL parameters (e.g., passing a Record ID in the URL path).
* **Versioning:** Best practice is to include a version in the mapping: `@RestResource(urlMapping='/v1/Warranty/*')`.

---

# 8. HTTP Methods

| Method | Annotation | Action | Description |
| :--- | :--- | :--- | :--- |
| **GET** | `@HttpGet` | Read | Retrieve data. Should never modify state. |
| **POST** | `@HttpPost` | Create | Create a new resource. |
| **PUT** | `@HttpPut` | Upsert | Create a new resource or fully replace an existing one. |
| **PATCH** | `@HttpPatch`| Update | Partially update an existing resource. |
| **DELETE**| `@HttpDelete`| Delete | Remove a resource. |

---

# 9. HTTP GET

Used to retrieve data.

```java
@HttpGet
global static Warranty__c getWarranty() {
    RestRequest req = RestContext.request;
    // Extracting ID from URL: /services/apexrest/Warranty/a0Bxx0000012345
    String warrantyId = req.requestURI.substring(req.requestURI.lastIndexOf('/') + 1);
    
    return [SELECT Id, Status__c, Vehicle__r.VIN__c FROM Warranty__c WHERE Id = :warrantyId LIMIT 1];
}
```

---

# 10. HTTP POST

Used to create data. The method parameters automatically map to JSON payload keys.

```java
@HttpPost
global static String createWarranty(String vehicleId, String claimType, Decimal amount) {
    Warranty__c newClaim = new Warranty__c(
        Vehicle__c = vehicleId,
        Type__c = claimType,
        Claim_Amount__c = amount,
        Status__c = 'Draft'
    );
    insert newClaim;
    return newClaim.Id;
}
```

---

# 11. HTTP PUT vs PATCH

* **PUT:** Fully replaces the resource. If a field is omitted in the request, it is conceptually nulled out (though in Apex, you must handle this explicitly).
* **PATCH:** Partially updates a resource. Only the fields provided are modified.

```java
@HttpPatch
global static String updateWarrantyStatus(String warrantyId, String newStatus) {
    Warranty__c claim = new Warranty__c(Id = warrantyId, Status__c = newStatus);
    update claim;
    return 'Success';
}
```

---

# 12. HTTP DELETE

Used to delete records.

```java
@HttpDelete
global static void deleteWarranty() {
    RestRequest req = RestContext.request;
    String warrantyId = req.requestURI.substring(req.requestURI.lastIndexOf('/') + 1);
    
    // Using Data Access Object or direct DML
    delete [SELECT Id FROM Warranty__c WHERE Id = :warrantyId];
}
```

---

# 13. RestContext

`RestContext` is a global system object that holds the `RestRequest` and `RestResponse` context for the current transaction.

* `RestContext.request`: Contains everything sent by the client.
* `RestContext.response`: Used to manipulate what is sent back to the client.

*Lifecycle:* Generated automatically by Salesforce when an inbound call hits the endpoint. It is destroyed once the transaction finishes.

---

# 14. RestRequest

Provides access to request data.

* `Headers`: `req.headers.get('Authorization')`
* `Parameters`: `req.params.get('dealerCode')` (Query strings like `?dealerCode=123`)
* `URI`: `req.requestURI`
* `HTTP Method`: `req.httpMethod`
* `Request Body`: `req.requestBody.toString()` (Used heavily for custom JSON deserialization).

---

# 15. RestResponse

Allows manipulation of the outgoing response.

```java
@HttpGet
global static void getCustomResponse() {
    RestResponse res = RestContext.response;
    res.addHeader('Content-Type', 'application/json');
    res.statusCode = 200;
    res.responseBody = Blob.valueOf('{"message": "Success"}');
}
```

---

# 16. JSON Serialization & Deserialization

* **`JSON.serialize(object)`**: Converts Apex objects to JSON strings.
* **`JSON.deserialize(jsonString, ApexType.class)`**: Converts JSON directly into a specified Apex class.
* **`JSON.deserializeUntyped(jsonString)`**: Converts JSON into a generic `Map<String, Object>`. Useful when the payload structure is dynamic or unknown.

```java
// Deserialization Example
String payload = RestContext.request.requestBody.toString();
WarrantyDTO dto = (WarrantyDTO) JSON.deserialize(payload, WarrantyDTO.class);
```

---

# 17. Wrapper Classes

Wrapper (or DTO - Data Transfer Object) classes represent complex, nested JSON payloads safely.

```java
public class WarrantyDTO {
    public String dealerId;
    public VehicleDetails vehicle;
    public List<ClaimLine> lines;
    
    public class VehicleDetails {
        public String vin;
        public Integer mileage;
    }
    
    public class ClaimLine {
        public String partNumber;
        public Decimal cost;
    }
}
```
*Purpose:* Strongly typed data structure. Prevents runtime errors and makes code significantly cleaner than handling generic maps.

---

# 18. Authentication

External systems must authenticate before hitting your REST API.
* **OAuth 2.0:** The industry standard. Uses **Connected Apps** in Salesforce to issue tokens.
* **Flow:** The client authenticates (e.g., using JWT Bearer Flow or Client Credentials Flow) and receives an **Access Token** (Session ID).
* **Bearer Token:** The client passes this token in the HTTP Header of the REST call: `Authorization: Bearer <AccessToken>`.

---

# 19. Security Best Practices

* **`with sharing`:** Enforce record-level visibility.
* **CRUD/FLS Checks:** Apex runs in system mode by default. You MUST check `Schema.sObjectType.Warranty__c.isCreateable()` or use `Security.stripInaccessible()` before DML operations.
* **SOQL Injection:** Always use bind variables (`:varName`), never concatenate strings for dynamic SOQL (`'SELECT Id FROM Object WHERE Name = \'' + name + '\''`).
* **Input Validation:** Validate payload data length, formats, and ranges before processing.

---

# 20. Error Handling

Do not let Apex exceptions bubble up as messy HTML errors. Catch exceptions and return clean HTTP codes.

| Code | Meaning | Usage |
| :--- | :--- | :--- |
| **200** | OK | Standard success. |
| **201** | Created | Successfully created a record (POST). |
| **400** | Bad Request | Client sent invalid JSON or missing fields. |
| **401** | Unauthorized | Invalid/Missing Session ID. |
| **403** | Forbidden | User authenticated, but lacks permissions (FLS/CRUD).|
| **404** | Not Found | Requested record does not exist. |
| **500** | Server Error | Internal Apex Exception. |

---

# 21. Governor Limits

REST APIs share standard synchronous Apex limits. 

| Limit | Maximum | Impact on REST |
| :--- | :--- | :--- |
| **SOQL Queries** | 100 | Limit loops querying child records. |
| **DML Statements** | 150 | Bulkify inserts/updates. |
| **Heap Size** | 6 MB | Cannot process massive JSON files synchronously. |
| **CPU Time** | 10,000 ms | Complex JSON parsing can consume CPU. |
| **Request Size** | ~6 MB | Maximum size of `RestRequest.requestBody`. |
| **Response Size**| ~6 MB | Maximum size of `RestResponse.responseBody`. |

---

# 22. API Versioning

APIs evolve. Changing an API structure can break client integrations.
* **URI Versioning (Recommended):** Include the version in the URL (`/services/apexrest/v1.0/Warranty/`). 
* When a breaking change is needed, create a new class mapped to `/v2.0/Warranty/`. Maintain the V1 class for backward compatibility until all clients migrate.

---

# 23. Testing Apex REST Services

To test REST services, you must mock the `RestContext`.

```java
@IsTest
private class WarrantyAPITest {
    @IsTest
    static void testGetWarranty() {
        // 1. Setup Test Data (Use TestDataFactory in real projects)
        Warranty__c w = new Warranty__c(Status__c = 'Draft');
        insert w;
        
        // 2. Mock RestContext
        RestRequest req = new RestRequest();
        RestResponse res = new RestResponse();
        
        req.requestURI = '/services/apexrest/Warranty/' + w.Id;
        req.httpMethod = 'GET';
        
        RestContext.request = req;
        RestContext.response = res;
        
        // 3. Execute
        Test.startTest();
        Warranty__c result = WarrantyAPI.getWarranty();
        Test.stopTest();
        
        // 4. Assert
        System.assertEquals('Draft', result.Status__c, 'Status should match');
    }
}
```

---

# 24. Enterprise Design Patterns

Never write raw business logic inside the `@RestResource` class. Use standard architecture patterns:

1.  **REST Controller (`WarrantyAPI.cls`):** Handles HTTP, URI parsing, and returning status codes.
2.  **Service Layer (`WarrantyService.cls`):** Handles the actual business logic, calculations, and error generation.
3.  **Selector / Repository (`WarrantySelector.cls`):** Handles all SOQL queries.

This ensures code is reusable. The Service Layer can be called by the API, a Lightning Web Component, or a trigger equally.

---

# 25. Real Project Scenarios

**Automotive CRM Scenario:**
* **Requirement:** A Dealer Management System (DMS) needs to submit Warranty Claims to Salesforce.
* **Solution:** A custom `@HttpPost` endpoint.
* **Process:** 1. The DMS calls `/v1/WarrantyClaim/` with a complex JSON payload (Vehicle VIN, Dealer Code, Line Items).
    2. The API uses a Wrapper Class to deserialize.
    3. The Service layer queries to ensure the Vehicle is under active warranty (Selector pattern).
    4. If active, it creates the `Warranty__c` and child `Claim_Line__c` records (Bulkified).
    5. Returns a `201 Created` with the Salesforce IDs.

---

# 26. Performance Optimization

* **Bulk Processing:** Design APIs to accept a `List<WarrantyDTO>` rather than a single record to reduce network latency and API call consumption.
* **Selective SOQL:** Ensure filters (e.g., `WHERE VIN__c = ...`) use indexed fields.
* **Asynchronous Processing:** If the payload requires massive CPU time or external callouts, parse the JSON, save it to a staging object, return a `202 Accepted` to the client, and process the data via a Batch or Queueable.

---

# 27. Common Mistakes

| Mistake | Impact | Solution |
| :--- | :--- | :--- |
| **No CRUD/FLS Checks** | Security vulnerability. | Use `Security.stripInaccessible()`. |
| **Returning raw Exception messages**| Exposes backend architecture. | Catch exceptions, return friendly errors & log the real error. |
| **SOQL/DML in Loops** | Instant Governor Limit crash. | Map and process collections before DML. |
| **Poor URL Naming** | Confusing for integrators. | Use nouns, not verbs (`/Warranty/` not `/CreateWarranty/`). |

---

# 28. Best Practices Checklist

* ✅ **Use RESTful naming conventions:** Use nouns for endpoints, utilize standard HTTP verbs.
* ✅ **Validate inputs:** Never trust external data. Validate strings, nulls, and formats.
* ✅ **Enforce CRUD/FLS:** Ensure the API user has the right to modify the data.
* ✅ **Handle exceptions properly:** Use `try-catch` blocks and utilize the `RestResponse` object to set `400`/`500` status codes.
* ✅ **Use Wrapper Classes:** Strongly type your data models for reliability.
* ✅ **Bulkify operations:** Design endpoints to handle arrays of data, not just single records.
* ✅ **Secure APIs using OAuth:** Never hardcode credentials; utilize Connected Apps.
* ✅ **Version APIs:** Start with `/v1/` to protect future iterations.
* ✅ **Write comprehensive unit tests:** Test positive, negative, and bulk scenarios.

---

# 29. Debugging REST APIs

* **Postman:** The gold standard for testing APIs. Setup an environment to handle OAuth token generation and store payloads.
* **Workbench:** Contains a built-in REST Explorer for quick, authenticated testing without external tools.
* **Debug Logs:** Set the API Integration User's trace flags in Salesforce to capture the execution context, limit consumption, and variable states.

---

# 30. Interview Questions & Answers

### Beginner Questions
**Q: How do you expose an Apex class as a REST service?**
A: Annotate the class with `@RestResource(urlMapping='...')`, declare the class as `global`, and use annotations like `@HttpGet` or `@HttpPost` on `global static` methods.

### Intermediate Questions
**Q: What is the difference between PUT and PATCH?**
A: PUT is intended for a full replacement/upsert of a resource. PATCH is for partial updates where only the provided fields are modified.

### Advanced Questions
**Q: How do you handle complex, nested JSON payloads in an HTTP POST?**
A: I avoid generic maps and instead create strongly typed Wrapper (DTO) classes mapping perfectly to the JSON structure. I then use `JSON.deserialize(RestContext.request.requestBody.toString(), MyWrapper.class)`.

### Architect-Level Questions
**Q: An external system needs to push 10,000 warranty claims at once. Can they use your custom Apex REST POST method?**
A: Generally, no. Passing 10k records in a synchronous REST call will likely breach the 6MB heap limit, the 10-second CPU limit, or the 150 DML limit. The architectural recommendation is to use the Salesforce Bulk API (2.0) for large data volumes, or design an async ingestion pattern where the REST API accepts the payload, dumps it into Big Objects/Staging, and fires a Batch job.

---

# 31. Revision Summary

* **REST Architecture:** Stateless, client-server, cacheable, uniform interface.
* **Apex REST:** Global class, `@RestResource`, `global static` methods.
* **HTTP Methods:** GET (Read), POST (Create), PUT (Upsert), PATCH (Update), DELETE (Remove).
* **Context:** `RestContext.request` / `RestResponse` handle transaction state and HTTP headers/body.
* **JSON:** Use `JSON.serialize/deserialize` with Wrapper Classes for robust integrations.
* **Security:** OAuth 2.0 (Connected Apps), `with sharing`, and strict CRUD/FLS validation.
* **Limits:** Adhere to 6MB Heap/Request size, 10s CPU time, and standard SOQL/DML limits.
* **Testing:** Manually instantiate and populate `RestRequest` and `RestContext` in `@IsTest` methods.
* **Architecture:** Decouple `@RestResource` controllers from Service Layer business logic. 

---