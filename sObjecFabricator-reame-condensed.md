### SObjectFabricator: Condensed Guide

SObjectFabricator is an Apex library for creating SObject test data quickly and easily in unit tests. It minimizes database interactions (DML/SOQL) and bypasses trigger logic for test setup, allowing precise control over field values, including read-only system fields, formula fields, and rollup summaries, as well as complex parent-child relationships.

#### Key Benefits

- **Faster & More Reliable Tests:** Reduces slow database operations and isolates tests from database state/triggers.
- **Full Control:** Set any field (Id, CreatedDate, formulas, rollups) to exact values.
- **Easy Relationship Setup:** Simplifies creating records with parent and child relationships.
- **Fluent API:** Offers flexible ways to define SObjects.

#### Getting Started

Deploy via the "Deploy to Salesforce" button from the original repository.
Use in Apex tests:

```apex
// Basic instantiation
Account myTestAccount = (Account) new sfab_FabricatedSObject(Account.class).toSObject();

// Instantiate and set a field
Contact myTestContact = (Contact) new sfab_FabricatedSObject(Contact.class)
    .set('LastName', 'Test')
    .toSObject();
```

_Note: The constructor takes a `Type` (e.g., `Account.class`), not an `SObjectType` token._

#### How it Works (Briefly)

SObjectFabricator uses dynamic schema introspection (`sfab_ObjectDescriber`) to understand SObject metadata. It constructs an in-memory representation of the SObject and its relationships. The `.toSObject()` call often uses `JSON.serialize()` and `JSON.deserialize()` to materialize the SObject, enabling the population of normally read-only or system-managed fields in a test context.

#### SObjectFabricator API (`sfab_FabricatedSObject`)

All setter methods return the `sfab_FabricatedSObject` instance for fluent chaining.

##### Constructors

1.  `new sfab_FabricatedSObject(Type sObjectType)`

    - Creates a fabricator for the specified SObject type.
    - Example: `new sfab_FabricatedSObject(Opportunity.class)`

2.  `new sfab_FabricatedSObject(Type sObjectType, Map<SObjectField, Object> fieldValues)`

    - Creates and initializes with field values using `SObjectField` tokens.
    - Example:
        ```apex
        Map<SObjectField, Object> accValues = new Map<SObjectField, Object>{
            Account.Name => 'Bulk Account',
            Account.Industry => 'Tech'
        };
        Account acc = (Account) new sfab_FabricatedSObject(Account.class, accValues).toSObject();
        ```

3.  `new sfab_FabricatedSObject(Type sObjectType, Map<String, Object> fieldValues)`
    - Creates and initializes with field and relationship values using string keys. Supports direct fields, parent fields (dot-notation), parent SObjects, and child SObject lists.
    - Example:
        ```apex
        Map<String, Object> contactValues = new Map<String, Object>{
            'LastName' => 'Smith',
            'Account.Name' => 'Parent Co.',
            'Owner' => new sfab_FabricatedSObject(User.class).set('Username', 'testuser@example.com')
        };
        Contact con = (Contact) new sfab_FabricatedSObject(Contact.class, contactValues).toSObject();
        ```

##### Finalizer Method

- `toSObject()`: Converts the fabricated definition into an actual SObject instance. This is typically the last call in the chain.

##### General Setters (`set`)

1.  `set(String fieldNameOrPath, Object value)`

    - Sets a field value by its string name or a dot-notation path for parent fields. Can also set parent SObjects or lists of child SObjects.
    - Examples:
        ```apex
        // Direct field
        fabAccount.set('Name', 'Test Account');
        // System field
        fabAccount.set('Id', '001xx0000000001AAA');
        fabAccount.set('CreatedDate', System.now().addDays(-5));
        // Parent field (dot-notation)
        fabContact.set('Account.Name', 'Global Corp');
        fabContact.set('Account.Owner.Profile.Name', 'System Administrator');
        // Parent SObject
        fabContact.set('Account', new sfab_FabricatedSObject(Account.class).set('Name', 'Direct Parent'));
        // Child SObjects (replaces existing for this relationship)
        List<sfab_FabricatedSObject> opps = new List<sfab_FabricatedSObject>{
            new sfab_FabricatedSObject(Opportunity.class).set('Name', 'Opp 1')
        };
        fabAccount.set('Opportunities', opps);
        ```

2.  `set(SObjectField fieldToken, Object value)`

    - Sets a field value using an `SObjectField` token.
    - _Limitation: Not reliable for setting fields on parent objects via tokens like `Contact.Account.Id`. Use string paths for parent fields._
    - Example: `fabAccount.set(Account.AnnualRevenue, 100000);`

3.  `set(Map<SObjectField, Object> fieldValues)`

    - Sets multiple field values from a map of `SObjectField` tokens to objects.
    - Example: `fabAccount.set(new Map<SObjectField, Object>{Account.BillingCity => 'SF'});`

4.  `set(Map<String, Object> allValues)`
    - Sets multiple fields and relationships from a map of string keys to objects (similar to the constructor).
    - Example: `fabContact.set(new Map<String, Object>{'Email' => 'test@example.com', 'Account.Industry' => 'Retail'});`

##### Explicit Field Setters

1.  `setField(String fieldName, Object value)`

    - Explicitly sets a direct field value by its string name.
    - Example: `fabOpportunity.setField('StageName', 'Prospecting');`

2.  `setField(SObjectField fieldToken, Object value)`
    - Explicitly sets a direct field value using an `SObjectField` token.
    - Example: `fabOpportunity.setField(Opportunity.CloseDate, Date.today().addMonths(1));`

##### Explicit Parent Relationship Setter

1.  `setParent(String parentRelationshipName, sfab_FabricatedSObject parentSObject)`
    - Sets a direct parent relationship. `parentRelationshipName` is the API name of the lookup or master-detail field on the SObject being fabricated.
    - Example:
        ```apex
        sfab_FabricatedSObject fabOwner = new sfab_FabricatedSObject(User.class).set('Username', 'owner@example.com');
        fabAccount.setParent('OwnerId', fabOwner); // Or simply 'Owner'
        ```

##### Child Relationship Management

1.  `add(String childRelationshipName, sfab_FabricatedSObject childSObject)`

    - Adds a single fabricated SObject to the specified child relationship list.
    - Example:
        ```apex
        sfab_FabricatedSObject fabContact = new sfab_FabricatedSObject(Contact.class).set('LastName', 'Doe');
        fabAccount.add('Contacts', fabContact);
        ```

2.  `setChildren(String childRelationshipName, List<sfab_FabricatedSObject> childSObjects)`

    - Sets (replaces) all children for a given relationship with the provided list.
    - Example:
        ```apex
        List<sfab_FabricatedSObject> newContacts = new List<sfab_FabricatedSObject>{...};
        fabAccount.setChildren('Contacts', newContacts);
        ```

3.  `addChild(String childRelationshipName, sfab_FabricatedSObject childSObject)`
    - (More explicit version of `add`) Adds a single fabricated SObject to the specified child relationship list.
    - Example: `fabAccount.addChild('Opportunities', new sfab_FabricatedSObject(Opportunity.class).set('Name', 'Opp X'));`

#### Handling Special Field Types

- **BASE64 Encoded Fields (Blob):** Fields like `ContentVersion.VersionData` or `Attachment.Body` can be set using either a `String` or a `Blob` value. SObjectFabricator handles the conversion.
    ```apex
    // Using String
    fabAttachment.set('Body', 'Attachment content');
    // Using Blob
    fabContentVersion.set('VersionData', Blob.valueOf('Document content'));
    ```
- **Rich Text Fields:** Set with HTML content as a String.
    ```apex
    fabCase.set('Description', '<h1>Issue</h1><p>Details here.</p>');
    ```

#### Key Limitations

- **`SObjectField` for Parent Fields:** Using `SObjectField` tokens like `Contact.Account.Id` (e.g., `fabContact.set(Contact.Account.Id, 'accountId')`) to set parent fields is unreliable and may set the field on the child itself (e.g., `Contact.Id`). **Always use string dot-notation for parent fields** (e.g., `fabContact.set('Account.Id', 'accountId')`).
- **`EmailCapture.RawMessage`:** This field cannot be set due to Salesforce platform restrictions.

#### Error Handling

SObjectFabricator throws specific Apex exceptions for issues like non-existent fields/relationships or type mismatches (e.g., `FieldDoesNotExistException`, `ParentRelationshipDoesNotExistException`, `FieldIsNotParentRelationshipException`, `FieldIsADifferentTypeException`), aiding in debugging test data setup.

#### Best Practices

- **Focused Data:** Create only necessary fields and relationships for the specific test scenario.
- **Test Data Factories:** Encapsulate common SObjectFabricator patterns in factory classes for reuse.
    ```apex
    // public class TestDataFactory {
    //  public static Account createStandardAccount() {
    //      return (Account) new sfab_FabricatedSObject(Account.class)
    //          .set('Name', 'Standard Test Account')
    //          .set('Industry', 'Technology')
    //          .toSObject();
    //  }
    // }
    ```
- **String Names for Relationships:** Prefer string names for parent and child relationships for clarity and to avoid `SObjectField` limitations with parent paths.

#### Common Testing Patterns

- **Testing Rollup Summaries:** Directly set rollup summary field values.

    ```apex
    Account acc = (Account) new sfab_FabricatedSObject(Account.class)
        .set('Name', 'Test Account')
        .set('TotalOpportunityAmount__c', 50000) // Assuming a custom rollup field
        .add('Opportunities', new sfab_FabricatedSObject(Opportunity.class).set('Amount', 50000)) // Optional: add children for context
        .toSObject();
    ```

- **Testing Time-Based Logic:** Set date/datetime fields like `CreatedDate`, `LastModifiedDate` to specific values.

    ```apex
    Case oldCase = (Case) new sfab_FabricatedSObject(Case.class)
        .set('CreatedDate', DateTime.now().addYears(-1))
        .set('Status', 'Closed')
        .toSObject();
    ```

- **Testing Multi-Level Parent Relationships:** Use dot-notation for setting fields on nested parents.
    ```apex
    Contact contact = (Contact) new sfab_FabricatedSObject(Contact.class)
        .set('LastName', 'TestContact')
        .set('Account.Name', 'TestAccount')
        .set('Account.Owner.Manager.Email', 'manager@example.com')
        .toSObject();
    ```
