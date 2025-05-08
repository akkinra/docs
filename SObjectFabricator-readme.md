# SObjectFabricator

[![Deploy to Salesforce](https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/deploy.png)](https://githubsfdeploy.herokuapp.com?owner=mattaddy&repo=SObjectFabricator)

SObjectFabricator is an Apex library designed to help you create SObject test data quickly and easily, right in your unit tests. It's all about reducing database interactions (like SOQL queries and DML operations) and avoiding dependencies on trigger logic when setting up your test scenarios. With SObjectFabricator, you can easily define SObjects with specific field values (including normally read-only system fields, formula fields, and rollup summaries) and complex parent-child relationships. This library is strongly inspired by [Stephen Willcock](https://github.com/stephenwillcock)'s [Dreamforce presentation](https://www.youtube.com/watch?v=dWertK6Legc) on Tests and Testability in Apex.

## Table of Contents

- [Who is this for?](#who-is-this-for)
- [Key Benefits](#key-benefits)
- [Motivation](#motivation)
- [Getting Started](#getting-started)
- [Fabricate Your SObjects](#fabricate-your-sobjects)
  - [How it Works (The Magic Behind the Scenes)](#how-it-works-the-magic-behind-the-scenes)
  - [Simple Example](#simple-example)
    - [Setting fields](#setting-fields)
    - [Setting Parent Relationships](#setting-parent-relationships)
    - [Setting Child Relationships](#setting-child-relationships)
  - [Detailed Examples](#detailed-examples)
  - [Error Handling and Validation](#error-handling-and-validation)
  - [Fluent API](#fluent-api)
  - [Set non-relationship field values in bulk via field references](#set-non-relationship-field-values-in-bulk-via-field-references)
  - [Set all field values in bulk via field name references](#set-all-field-values-in-bulk-via-field-name-references)
  - [More explicit API](#more-explicit-api)
  - [Setting BASE64 encoded fields](#setting-base64-encoded-fields)
  - [Limitations](#limitations)
    - [SObjectField](#sobjectfield)
    - [EmailCapture.RawMessage](#emailcapturerawmessage)
- [Best Practices](#best-practices)
- [Common Testing Patterns](#common-testing-patterns)
  - [Testing Aggregate Functions (e.g., Rollup Summaries)](#testing-aggregate-functions-eg-rollup-summaries)
  - [Testing Time-Based Logic](#testing-time-based-logic)
  - [Testing Multi-Level Parent Relationships](#testing-multi-level-parent-relationships)
- [Working with Special Field Types](#working-with-special-field-types)
  - [Handling Blob/BASE64 Fields](#handling-blobbase64-fields)
  - [Handling Rich Text Fields](#handling-rich-text-fields)
- [Troubleshooting](#troubleshooting)
  - [Common Error Messages](#common-error-messages)
  - [Field Not Being Set](#field-not-being-set)
- [Contributing to SObjectFabricator](#contributing-to-sobjectfabricator)

## Who is this for?

This library is for Apex developers who write unit tests and want to:

*   Make their tests faster and more reliable by avoiding unnecessary database calls (DML and SOQL).
*   Easily create SObject test data in specific states, especially when dealing with read-only fields, formula fields, or complex relationship setups.
*   Reduce dependencies on trigger execution logic within their unit tests for data setup.

A basic understanding of Apex, SObjects, and Salesforce unit testing is recommended.

## Key Benefits

*   **Faster Tests:** Reduces reliance on slow database operations.
*   **More Reliable Tests:** Isolates test logic from database state and trigger side-effects.
*   **Full Control:** Set any field, including system fields (like `Id`, `CreatedDate`), formula fields, and rollup summaries, to the exact values needed for your test scenario.
*   **Easy Relationship Setup:** Simplifies creating records with parent and child relationships.
*   **Fluent and Flexible API:** Provides multiple ways to define your SObjects, suiting different preferences.

## Motivation

> How many times do you have to execute a query to know that it works? - Robert C. Martin (Uncle Bob)

Databases and SOQL are slow, so we should mock them out for the majority of our tests. We should be able to drive our SObjects into the state in which the system can be tested.

In addition to queries, how many times do we need to test our triggers to know that they work?

Rather than relying on triggers to populate fields in our unit tests, or the results of a SOQL query to populate relationships, we should manually force our SObjects into the state in which the system can be tested.

## Getting Started

The easiest way to add SObjectFabricator to your Salesforce org or project is to use the "Deploy to Salesforce" button at the top of this README. This will guide you through the deployment process. Once deployed, you can start using the `sfab_FabricatedSObject` class in your Apex test classes by instantiating it, for example:

```apex
Account myTestAccount = (Account) new sfab_FabricatedSObject(Account.class).toSObject();
Contact myTestContact = (Contact) new sfab_FabricatedSObject(Contact.class).set('LastName', 'Test').toSObject();
```

Note that `sfab_FabricatedSObject` accepts a `Type` parameter (e.g., `Account.class`), not an `SObjectType` token.

## Fabricate Your SObjects

SObjectFabricator provides the ability to set any field value, including system, formula, rollup summaries, and relationships. This is achieved by constructing the SObject in memory with the desired state, bypassing standard DML operations for field population.

### How it Works (The Magic Behind the Scenes)

At its core, SObjectFabricator relies on two key mechanisms:

1.  **Dynamic Schema Introspection**: The library includes a sophisticated component (`sfab_ObjectDescriber`) that dynamically inspects your Salesforce organization's SObject metadata using `Schema.getGlobalDescribe()`. This allows it to understand field types, relationship names (parent and child), and valid target objects for relationships on the fly. This means SObjectFabricator works seamlessly with standard objects, custom objects, and even managed package objects without requiring you to pre-configure or hardcode object structures. It's how the fabricator can intelligently handle dot-notation for parent fields (e.g., `Account.Owner.Profile.Name`) and correctly identify relationship types.

2.  **In-Memory SObject Construction**: When you use methods like `.set()`, `.add()`, or their more specific counterparts (`.setField()`, `.setParent()`, `.setChildren()`), SObjectFabricator builds up an internal representation of your SObject and its related data. The final `.toSObject()` call then takes this internal structure and materializes it into a true SObject instance. This process often involves `JSON.serialize()` and `JSON.deserialize()`, a common Apex technique that allows for the population of fields that are normally read-only after record creation (like `Id` or `CreatedDate`) or are system-managed (like some formula fields or rollup summaries in a test context).

This combination allows you to precisely define the state of an SObject for your tests, including fields and relationships that would be difficult or impossible to set up through conventional DML.

### Simple Example

#### Setting fields

Creating an SObject and setting properties on it can be as simple as:

```java
Account accountSobject = (Account)new sfab_FabricatedSObject( Account.class )
                            .set( 'Name'            , 'Account Name' )
                            .set( 'LastModifiedDate', Date.newInstance( 2017, 1, 1 ) )
                            .toSObject();
```

#### Setting Parent Relationships

Creating an SObject with a Lookup or Master / Detail relationship set can be as simple as:

```java
Contact contactSobject = (Contact)new sfab_FabricatedSObject( Contact.class )
                            .set( 'LastName'    , 'PersonName' )
                            .set( 'Account.Name', 'Account Name' )
                            .toSObject();
```

Or you can explicitly set the object by using another fabricated SObject and setting the relationship:

```java
sfab_FabricatedSObject fabricatedAccount = new sfab_FabricatedSObject( Account.class )
													.set( 'Name', 'Account Name' );

Contact contactSobject = (Contact)new sfab_FabricatedSObject( Contact.class )
                            .set( 'Account', fabricatedAccount )
                            .toSObject();
```

#### Setting Child Relationships

Creating an SObject with a child relationship set can be as simple as:

```java
Account accountSobject = (Account)new sfab_FabricatedSObject( Account.class )
                            .set( 'Name'            , 'Account Name' )
                            .add( 'Contacts', new sfab_FabricatedSObject( Contact.class )
                                                    .set( 'LastName', 'PersonName' ) )
                            .add( 'Contacts', new sfab_FabricatedSObject( Contact.class )
                                                    .set( 'LastName', 'OtherPersonName' ) )
                            .toSObject();
```

Note on `set` vs `add` for Child Relationships:
*   `set('ChildRelationshipName', listOfChildren)` (or `setChildren(...)`) will replace any existing fabricated children for that relationship with the new list.
*   `add('ChildRelationshipName', childObject)` (or `addChild(...)`) will append the `childObject` to the existing list of fabricated children for that relationship. If no children have been added yet for that relationship, it will initialize the list with this child.

### Detailed Examples

There are lots of other options.  With SObjectFabricator, you can:

```java
sfab_FabricatedSObject fabricatedAccount = new sfab_FabricatedSObject( Account.class );

// Set fields using SObjectField references, including those not normally settable:
fabricatedAccount.set( Account.Id, 'Id-1' );
fabricatedAccount.set( Account.LastModifiedDate, Date.newInstance( 2017, 1, 1 ) );

// Set fields using the names of the fields:
fabricatedAccount.set( 'Name', 'The Account Name' );

// Set lookup / master detail relationships explicitly
fabricatedAccount.set( 'Owner', new sfab_FabricatedSObject( User.class ).set( 'Username', 'TheOwner' ) );

// Set lookup / master detail relationships implicitly
fabricatedAccount.set( 'Owner.Alias', 'alias' );

// Set multi-leveled lookup / master detail relationships implicitly
fabricatedAccount.set( 'Owner.Profile.Name', 'System Administrator' );

// Set child relationships in one go
fabricatedAccount.set( 'Opportunities', new List<sfab_FabricatedSObject> {
        new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-1' ),
        new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-2' )
});

// Set child relationships one-by-one
fabricatedAccount.add( 'Opportunities', new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-3' ) );
fabricatedAccount.add( 'Opportunities', new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-4' ) );

// Generate an SObject from that configuration
Account sObjectAccount = (Account)fabricatedAccount.toSObject();

// Account:{LastModifiedDate=2017-01-01 00:00:00, Id=Id-1, Name=The Account Name}
System.debug( sObjectAccount );

// User:{Username=TheOwner, Alias=alias}
System.debug( sObjectAccount.Owner );

// Profile:{Name=System Administrator}
System.debug( sObjectAccount.Owner.Profile );

// (Opportunity:{Id=OppId-1}, Opportunity:{Id=OppId-2}, Opportunity:{Id=OppId-3}, Opportunity:{Id=OppId-4})
System.debug( sObjectAccount.Opportunities );
```

Each of the mechanisms also allow you to navigate through parent structures:

```java
sfab_FabricatedSObject fabricatedContact = new sfab_FabricatedSObject( Contact.class );

fabricatedContact.set( 'Account.Name', 'The Account Name' );

fabricatedContact.set( 'Account.Owner', new sfab_FabricatedSObject( User.class ) );

fabricatedContact.set( 'Account.Opportunities', new List<sfab_FabricatedSObject> {
        new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-1' ),
        new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-2' )
});

fabricatedContact.add( 'Account.Opportunities', new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-1' ) );

Contact sObjectContact = (Contact)fabricatedContact.toSObject();
```

### Error Handling and Validation

SObjectFabricator performs various validations during the fabrication process. For example, it checks:
*   If specified fields or relationship names actually exist on the SObject type (or its parents in dot-notation).
*   If the SObject type being assigned to a parent or child relationship is compatible with what the schema defines for that relationship.
*   If you attempt to use a simple field name where a relationship is expected, or vice-versa.

If validation fails, the library will throw specific Apex exceptions (e.g., `FieldDoesNotExistException`, `ParentRelationshipDoesNotExistException`, `FieldIsNotParentRelationshipException`, `FieldIsADifferentTypeException`) to help you pinpoint issues in your test data setup.

### Fluent API

The set methods form a fluent API, meaning you can condense the configuration along the lines of

```java
Account sObjectAccount = (Account)new sfab_FabricatedSObject( Account.class )
    .set( Account.Id, 'Id-1' )
    .set( Account.LastModifiedDate, Date.newInstance( 2017, 1, 1 ) )
    .set( 'Name', 'The Account Name' )
    .set( 'Owner', new sfab_FabricatedSObject( User.class ).set( 'Username', 'TheOwner' ) )
    .set( 'Opportunities', new List<sfab_FabricatedSObject> {
        new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-1' ),
        new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-2' )
    })
    .add( 'Opportunities', new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-3' ) ) // though, it would be odd to combine set
    .add( 'Opportunities', new sfab_FabricatedSObject( Opportunity.class ).set( 'Id', 'OppId-4' ) ) // and add on a single child relationship
    .toSObject();
```

### Set non-relationship field values in bulk via field references

Field values can be set in bulk, either at construction time, or later (using `set`), by creating a `Map<SObjectField, Object>` of the fields that you wish to set and using that.

```java
Map<SObjectField, Object> accountValues = new Map<SObjectField, Object> {
        Account.Id => 'Id-1',
        Account.LastModifiedDate => Date.newInstance(2017, 1, 1)
};

Account fabricatedViaConstructor = (Account)new sfab_FabricatedSObject(Account.class, accountValues)
                                                    .toSObject();


Account fabricatedViaSet = (Account)new sfab_FabricatedSObject(Account.class)
                                            .set(accountValues)
                                            .toSObject();

```

### Set all field values in bulk via field name references

Field, Parent and Child Relationship values can be set in bulk, either at construction time, or later, by passing a `Map<String,Object>`.

```java
Map<String,Object> contactValues = new Map<String,Object> {
        'Id'                     => 'Id-1',

        'LastModifiedDate'       => Date.newInstance(2017, 1, 1),

        'Account.Name'           => 'The Account Name',

        'Owner'                  => new sfab_FabricatedSObject( User.class )
                                        .set( 'Username', 'The Contact Owner' ),

        'Account.Owner'          => new sfab_FabricatedSObject( User.class )
                                        .set( 'Username', 'The Account Owner' ),

        'Opportunities'          => new List<sfab_FabricatedSObject>{
                                        new sfab_FabricatedSObject( Opportunity.class )
                                            .set( 'Name', 'Contact Opportunity Name' )
                                    },
        'Account.Opportunities'  => new List<sfab_FabricatedSObject>{
                                        new sfab_FabricatedSObject( Opportunity.class )
                                            .set( 'Name', 'Account Opportunity Name' )
                                    }
};

Contact fabricatedViaConstructor = (Contact)new sfab_FabricatedSObject( Contact.class, contactValues )
                                                    .toSObject();

Contact fabricatedViaSet = (Contact)new sfab_FabricatedSObject(Contact.class)
                                            .set( contactValues )
                                            .toSObject();
```

### More explicit API

If you prefer the set methods to be a little more explicit in their intention, you can use more specific versions of the `set` method.

```java
Account sObjectAccount = (Account)new sfab_FabricatedSObject( Account.class )
    .setField( Account.Id, 'Id-1' )
    .setField( 'LastModifiedDate', Date.newInstance( 2017, 1, 1 ) )
    .setParent( 'Owner', new sfab_FabricatedSObject( User.class ).setField( 'Username', 'TheAccountOwner' ) )
    .setParent( 'Contact.Owner', new sfab_FabricatedSObject( User.class ).setField( 'Username', 'TheContactOwner' ) )
    .setChildren( 'Opportunities', new List<sfab_FabricatedSObject> {
        new sfab_FabricatedSObject( Opportunity.class ).setField( Opportunity.Id, 'OppId-1' ),
        new sfab_FabricatedSObject( Opportunity.class ).setField( Opportunity.Id, 'OppId-2' ) } )
    .addChild( 'Opportunities', new sfab_FabricatedSObject( Opportunity.class ).setField( 'Id', 'OppId-3') ) // though, it would be odd to combine setChildren
    .addChild( 'Opportunities', new sfab_FabricatedSObject( Opportunity.class ).setField( 'Id', 'OppId-4') ) // and addChild on a single child relationship
    .toSObject();
```

### Setting BASE64 encoded fields

Fields such as `ContentVersion.VersionData` and `Attachment.Body` are BASE64 encoded, and as such can normally be set by specifying with a Blob value.

When setting these fields using `sfab_FabricatedSObject`, you may specify the value either:
* As a Blob.  E.g. `Blob.valueOf( 'abc' )`
* As a String. E.g. `'abc'`

`sfab_FabricatedSObject` will handle the conversion to the correct data type for you. This is done via the `postBuildProcess` mechanism in the `sfab_FieldValuePairNode` class, which properly handles Blob fields after the JSON deserialization process.

### Limitations

#### SObjectField

When using SObjectFields in order to set field values, it is not possible for SObjectFabricator to ensure that fields from the correct object are being used.  This is due to a limitation in the SObjectField class as supplied by Salesforce.  I.E. SObjectField does not (at time of writing) include a reference to the object that the field belongs to, nor the method of construction.

Therefore, the following may not do quite as expected:

```java
Contact con = (Contact) new sfab_FabricatedSobject( Contact.class )
                    .set( Contact.Account.Id, 'The Account Id?' )
                    .toSObject();

System.debug( '### Contact.Id: ' + con.Id );
// Contact.Id: The Account Id?

System.debug( '### Contact.Account.Id: ' + con.Account.Id );
// Contact.Account.Id: null
```

Similarly, the following yields the same misleading result:

```java
Contact con = (Contact) new sfab_FabricatedSobject( Contact.class )
                    .set( Account.Id, 'The Account Id?' )
                    .toSObject();

System.debug( '### Contact.Id: ' + con.Id );
// Contact.Id: The Account Id?

System.debug( '### Contact.Account.Id: ' + con.Account.Id );
// Contact.Account.Id: null
```

In this case, the correct mechanism to use is a String representation of the field name.  I.E.


```java
Contact con = (Contact) new sfab_FabricatedSobject( Contact.class )
                    .set( 'Account.Id', 'The Account Id!' )
                    .toSObject();

System.debug( '### Contact.Id: ' + con.Id );
// Contact.Id: null

System.debug( '### Contact.Account.Id: ' + con.Account.Id );
// Contact.Account.Id: The Account Id!
```

#### EmailCapture.RawMessage

It is not possible to set the field `RawMessage` on the object `EmailCapture` this is because of two issues combined:

1. Salesforce does not allow `EmailCapture.RawMessage` to be set directly - it will throw `System.SObjectException: Field RawMessage is not editable`.
2. Salesforce cannot deserialize JSON that includes a reference to a BASE64 encoded field.

Because of this combination, there is no way to set `EmailCapture.RawMessage` for an in-memory object.

## Best Practices

To get the most out of SObjectFabricator, consider these best practices:

*   **Keep Tests Focused:** Create only the fields and relationships that your test actually needs. This keeps your tests cleaner and more maintainable.

*   **Create Test Factory Classes:** Consider building wrapper classes around SObjectFabricator that produce common test data patterns for your org. For example:

    ```apex
    public class TestDataFactory {
        public static Account createStandardAccount() {
            return (Account)new sfab_FabricatedSObject(Account.class)
                .set('Name', 'Standard Test Account')
                .set('Industry', 'Technology')
                .set('BillingCity', 'San Francisco')
                .toSObject();
        }
        
        public static Account createAccountWithContacts(Integer numContacts) {
            sfab_FabricatedSObject fabricatedAccount = new sfab_FabricatedSObject(Account.class)
                .set('Name', 'Account With Contacts');
                
            for(Integer i = 0; i < numContacts; i++) {
                fabricatedAccount.add('Contacts', new sfab_FabricatedSObject(Contact.class)
                    .set('LastName', 'Contact ' + i));
            }
            
            return (Account)fabricatedAccount.toSObject();
        }
    }
    ```

*   **Be Mindful of Field Types:** When setting field values, it's best to use values that closely match the field's type. While SObjectFabricator will try to handle type conversions, it's cleaner to provide the correct type upfront.

*   **Use String Field Names for Relationships:** For parent-child relationships, use string field names (e.g., `'Account.Name'`) rather than SObjectField tokens to avoid confusion.

*   **Organize Complex Fabrications:** For complex object hierarchies, consider building up your objects in steps to improve readability:

    ```apex
    // Instead of one giant chained call:
    sfab_FabricatedSObject fabricatedAccount = new sfab_FabricatedSObject(Account.class)
        .set('Name', 'Parent Account');
        
    sfab_FabricatedSObject fabricatedContact = new sfab_FabricatedSObject(Contact.class)
        .set('LastName', 'Smith')
        .set('Account', fabricatedAccount);
    
    sfab_FabricatedSObject fabricatedOpportunity = new sfab_FabricatedSObject(Opportunity.class)
        .set('Name', 'New Opportunity')
        .set('StageName', 'Prospecting')
        .set('CloseDate', Date.today().addDays(30))
        .set('Account', fabricatedAccount);
    ```

## Common Testing Patterns

Here are some common test scenarios and how to handle them with SObjectFabricator:

### Testing Aggregate Functions (e.g., Rollup Summaries)

```apex
// Create an account with opportunities that sum to a specific amount
Account acct = (Account)new sfab_FabricatedSObject(Account.class)
    .set('Name', 'Test Account')
    // Set a rollup summary field directly rather than calculating it from child records
    .set('AnnualRevenue', 500000)
    // Add child opportunities whose amounts would normally roll up
    .add('Opportunities', new sfab_FabricatedSObject(Opportunity.class)
        .set('Name', 'Opp 1')
        .set('Amount', 300000)
        .set('StageName', 'Closed Won')
        .set('CloseDate', Date.today()))
    .add('Opportunities', new sfab_FabricatedSObject(Opportunity.class)
        .set('Name', 'Opp 2')
        .set('Amount', 200000)
        .set('StageName', 'Closed Won')
        .set('CloseDate', Date.today()))
    .toSObject();

// Now you can test code that depends on the AnnualRevenue value
// without having to involve actual DML or triggers to calculate it
```

### Testing Time-Based Logic

```apex
// Create records with specific timestamps for time-based business logic
Case testCase = (Case)new sfab_FabricatedSObject(Case.class)
    .set('Subject', 'Test Case')
    .set('Status', 'New')
    .set('CreatedDate', DateTime.now().addDays(-5))
    .set('LastModifiedDate', DateTime.now().addHours(-2))
    .toSObject();

// Now you can test business logic that depends on when the case was created
// or last modified without waiting or manipulating system timestamps
```

### Testing Multi-Level Parent Relationships

```apex
// Testing a utility that needs access to a contact's account owner's manager
Contact contact = (Contact)new sfab_FabricatedSObject(Contact.class)
    .set('LastName', 'Test Contact')
    .set('Account.Name', 'Test Account')
    .set('Account.Owner.Name', 'Account Owner')
    .set('Account.Owner.Manager.Name', 'Manager')
    .set('Account.Owner.Manager.Email', 'manager@example.com')
    .toSObject();

// Now you can test code that traverses this path without creating all these records in the database
```

## Working with Special Field Types

### Handling Blob/BASE64 Fields

Fields like `ContentVersion.VersionData` and `Attachment.Body` are BASE64 encoded. SObjectFabricator handles these fields specially through the `postBuildProcess` method in the `sfab_FieldValuePairNode` class:

```apex
// Using a String value - will be automatically converted to Blob
Attachment attachment = (Attachment)new sfab_FabricatedSObject(Attachment.class)
    .set('Name', 'Test Attachment')
    .set('Body', 'This is the content of the attachment')
    .toSObject();

// Using a Blob value directly
ContentVersion cv = (ContentVersion)new sfab_FabricatedSObject(ContentVersion.class)
    .set('Title', 'Test Document')
    .set('PathOnClient', 'test.txt')
    .set('VersionData', Blob.valueOf('This is the content of the document'))
    .toSObject();
```

### Handling Rich Text Fields

Rich text fields can contain HTML:

```apex
Case caseWithRichText = (Case)new sfab_FabricatedSObject(Case.class)
    .set('Subject', 'Rich Text Test')
    // Assuming 'Description' is a rich text field
    .set('Description', '<h1>Important Issue</h1><p>This needs <b>immediate</b> attention</p>')
    .toSObject();
```

## Troubleshooting

### Common Error Messages

* **"The field X.Y does not exist"**: Check if the field name or relationship name is spelled correctly. Also verify that the field exists on the SObject you're working with.

* **"The field X.Y is not a parent relationship"**: This error occurs when you attempt to use `setParent` on a field that isn't a lookup or master-detail relationship. Double-check the field type.

* **"The field X.Y is Z, not W"**: This error indicates you're trying to set a field of one type to a value of an incompatible type. Verify that the SObject type you're using for a relationship is correct.

* **"Could not auto-assign an object for the field X.Y"**: This typically happens with polymorphic lookup fields (fields that can point to different SObject types). For these fields, you need to explicitly create the parent SObject rather than relying on automatic creation.

### Field Not Being Set

If a field doesn't appear to be set correctly after calling `toSObject()`:

1. Check if you're using an SObjectField token for a parent field (e.g., `Contact.Account.Name`). As mentioned in the limitations section, this doesn't work as expected. Use a string field name instead.

2. For BASE64/Blob fields, ensure the type of value being set is either a Blob or a String.

3. Remember that system-protected fields like `SystemModstamp` can't be set, even with SObjectFabricator.

## Contributing to SObjectFabricator

If you'd like to contribute to SObjectFabricator, please:

1. Fork the repository
2. Create a feature branch
3. Add appropriate test coverage for your changes 
4. Submit a pull request with a clear description of your changes

Bug reports, feature requests, and questions are also welcome via the Issues tracker on GitHub.

