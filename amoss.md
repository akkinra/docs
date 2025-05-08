# Amoss

Apex Mock Objects, Spies and Stubs - A Simple Mocking framework for Apex (Salesforce)

## Costs, and a note from the author...

Use, replication and extension of Amoss is entirely free, and provided without support (see the associated license, which will take precendence in any ambiguity).

If you wish to donate to the original author and major contributor, you can do so via the original source repository (https://github.com/bobalicious), and using the Sponsorship button on that site.

Please note, there is no obligation to donate, and doing so does not imply any form of contract or support arrangement.  I'll always do my best to help though... just reach out to me.

## Table of Contents

- [Why use Amoss?](#why-use-amoss)
- [Understanding Testing Terminology](#understanding-testing-terminology)
  - [What is a Test Double?](#what-is-a-test-double)
  - [Types of Test Doubles](#types-of-test-doubles)
  - [Choosing the Right Test Double](#choosing-the-right-test-double)
- [Installation](#installating-it)
  - [What do I get?](#what-do-i-get)
- [How do I use it?](#how-do-i-use-it)
  - [Constructing and using a Test Double](#constructing-and-using-a-test-double)
  - [Configuring return values](#configuring-return-values)
  - [Using the Test Double as a Spy](#using-the-test-double-as-a-spy)
  - [Configuring a stricter Test Double](#configuring-a-stricter-test-double-that-isnt-a-mock-object)
- [Specifying return values in different ways](#specifying-return-values-in-different-ways)
  - [returnsItself](#returnsitself--returningitself--willreturnitselfF)
  - [isFluent](#isfluent)
  - [byDefaultMethodsReturn](#bydefaultmethodsreturn)
- [Specifying parameters in different ways](#specifying-parameters-in-different-ways)
  - [setTo](#setto)
  - [setToTheSameValueAs](#settothesamevalueas)
  - [set](#set)
  - [containing](#containing)
  - [matching](#matching)
  - [sObject Specific Comparisons](#sobject-specific-comparisons)
  - [List Specific Comparisons](#list-specific-comparisons)
  - [Custom Verifiers](#custom-verifiers)
- [Other Behaviours](#other-behaviours)
  - [Using Method Handlers](#using-method-handlers)
  - [Throwing exceptions](#throwing-exceptions)
  - [Don't allow any calls](#dont-allow-any-calls)
  - [getDouble / generateDouble / createClone](#getdouble--generatedouble--createclone)
  - [Longer format .method definition](#longer-format-method-definition)
- [What? When?](#what-when)
  - [Test Stub](#test-stub)
  - [Strict Test Stub](#strict-test-stub)
  - [Test Spy](#test-spy)
  - [Strict Test Spy](#strict-test-spy)
  - [Mock Object](#mock-object)
- [Synonyms](#synonyms)
- [Limitations](#limitations)
- [HttpCalloutMock Support](#httpcalloutmock-support)
  - [Building a simple HttpCalloutMock](#building-a-simple-httpcalloutmock)
  - [Defining a default return](#defining-a-default-return)
  - [Defining a conditional return](#defining-a-conditional-return)
  - [Verification Methods](#verification-methods)
  - [Using expects, allows and Test Spy behaviours](#using-expects-allows-and-test-spy-behaviours-with-httpcalloutmocks)
  - [Custom verifiers and handlers](#custom-verifiers-and-handlers)
  - [expectsNoCalls for HTTP mocks](#expectsnocalls-for-http-mocks)
- [Troubleshooting](#troubleshooting)
  - [Common Issues and Solutions](#common-issues-and-solutions)
- [Release Notes](#release-notes)
- [Acknowledgements / References](#acknowledgements--references)

## Why use Amoss?

Amoss provides a simple interface for implementing Mock, Test Spy and Stub objects (Test Doubles) for use in Unit Testing.

It's intended to be very straightforward to use, and to result in code that's even more straightforward to read.

As a simple example, the following example:

* Creates a Test Double of the class `DeliveryProvider`
* Configures the methods `canDeliver` and `scheduleDelivery` to always return `true`

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .willReturn( true )
    .also().when( 'scheduleDelivery' )
        .willReturn( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();
```

It also provides an interface for creating and registering an `HttpCalloutMock` in a simple and easy to read format.

For example:

```java

Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .when()
        .method( 'GET' )
        .endpoint().containing( 'account/' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
            .body( new Map<String,Object>{ 'Name' => 'The account name' } )
    .also().when()
        .method( 'POST' )
        .respondsWith()
            .status( 'Not Found' )
            .statusCode( 404 );
```

This documetation page covers the fundamentals of building Test Doubles.  [For documentation on creating `HttpCalloutMocks`, see here.](HTTPCALLOUTMOCKS.md).

## Understanding Testing Terminology

If you're new to unit testing or test-driven development, some of the terminology can be confusing. Here's a beginner-friendly explanation of key concepts used throughout this documentation:

### What is a Test Double?

A **Test Double** is like a stunt double in movies - it stands in for a real object during testing. Just as a stunt double looks like the actor but is specialized for specific scenes, a Test Double looks like the real class but is specialized for testing.

Test Doubles allow you to:
- Test code in isolation (without dependencies)
- Control the behavior of dependencies
- Verify how your code interacts with its dependencies

In Salesforce, Test Doubles are especially useful when your code depends on:
- External web services
- Complex database operations
- Classes that are difficult to set up in a test environment

### Types of Test Doubles

#### Stub

A **Stub** is the simplest type of Test Double. It's like a cardboard prop on a movie set - it looks real enough for the scene, but has minimal functionality.

**Example in real life**: A store mannequin - it looks like a person but only does one thing: display clothes.

**When to use**: When you just need your test to return predefined values and don't care about how the object is used.

```java
// Creating a stub
deliveryProviderController
    .when('canDeliver')
    .willReturn(true);
```

#### Spy

A **Spy** is like a recording device attached to a real object. It remembers what happened so you can check it later.

**Example in real life**: A delivery tracking system that records when packages are picked up, where they go, and when they're delivered.

**When to use**: When you want to verify that certain methods were called with specific values.

```java
// Creating a spy and later checking what happened
deliveryProviderController.when('scheduleDelivery').willReturn(true);
// ... test runs ...
Assert.areEqual(expectedPostcode, deliveryProviderController.latestCallOf('scheduleDelivery').parameter('postcode'));
```

#### Mock

A **Mock** is the most sophisticated Test Double. It's like an actor with a script who will complain if the scene doesn't go according to plan.

**Example in real life**: A rehearsed dance partner who expects certain steps in a specific order.

**When to use**: When the exact sequence and parameters of method calls are critical to your test.

```java
// Creating a mock with expectations
deliveryProviderController
    .expects('canDeliver')
    .withParameter(deliveryPostcode)
    .returning(true)
    .then().expects('scheduleDelivery')
    .withParameter(deliveryPostcode)
    .returning(true);
// ... test runs ...
deliveryProviderController.verify(); // Will fail if the expected calls didn't happen in order
```

### Choosing the Right Test Double

As a junior developer, here's a simple guide:

- Use a **Stub** when you just need something to return predefined values
- Use a **Spy** when you want to check how your code used an object after the test is complete
- Use a **Mock** when the sequence of interactions is important and you want automatic verification

Amoss makes it easy to create all these types of Test Doubles with a consistent, readable syntax.

### Installating it

#### git clone / copy / deploy

If you are familar with using SFDX, the ant migration tool or using a local IDE, It is recommended that you either clone this repository or download a release from the release section, copy the Amoss files you require, and install them using your normal mechanism.

#### Unlocked Package - SFDX

Alternatively, Amoss is available as an Unlocked Package, and the 'currently published' version based *this* branch can be installed (after setting the default org), using:

`sfdx force:package:install --package "amoss@1.2.0-0"`

You should note that this *may not* be the most recent version that exists on this branch.  There are times when the most recent version has not been published as an Unlocked Package Version.  In addition, the Unlocked Package contains the `amoss_main` and `amoss_test` files, though does not include `amoss_examples`.

Links to the release notes, and any changes pending release can be found at the end of this file.

##### Note
If you are not familiar with the SFDX commands, then it is recommended that you read the documentation here: https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_force_package.htm


#### Unlocked Package - Installation link

For Dev Instances or Production, the Unlocked Package can be installed via:

* https://login.salesforce.com/packaging/installPackage.apexp?p0=04t4K000002SC9NQAW

For all other instances:

* https://test.salesforce.com/packaging/installPackage.apexp?p0=04t4K000002SC9NQAW

#### Install Button

As a final option, you can install directly from this page, using the following 'Deploy to Salesforce' button.

##### Note

If running from the branch 'main', you should enter 'main' into the 'Branch/Tag/Commit:' field of 'Salesforce Deploy'.

This is because of a bug in that application that incorrectly selects the default branch as 'master' (https://github.com/afawcett/githubsfdeploy/issues/43)

<a href="https://githubsfdeploy.herokuapp.com">
  <img alt="Deploy to Salesforce"
       src="https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/src/main/webapp/resources/img/deploy.png">
</a>

Be aware that this will grant a third pary Heroku app access to your org.  I have no reason to believe that 'Andy in the Cloud'  (https://andyinthecloud.com/about/) will do anything malicious with that access, though you should always be aware that there are risks when you grant access to an org, and using such an app is entirely at your own risk.

#### What do I get?

Amoss consists of 3 sets of classes:
* `Amoss_`
  * The only parts that are necessary - the framework itself.
* `AmossTest_`
  * The tests for the framework, and their supporting files.  Required if you want to enhance Amoss, not so useful otherwise, but worth keeping if you can, just in case.  All `Amoss_` classes are defined as `@isTest`, meaning they do not count towards code coverage or Apex lines of codes.
* `AmossExample_`
  * Example classes and tests, showing simple use cases for Amoss in a runnable format.  There is no reason to keep these in your code-base once you've read and understand them.

## How do I use it?

### Constructing and using a Test Double

Amoss can be used to build simple stub objects - AKA Configurable Test Doubles, by:
* Constructing an `Amoss_Instance`, passing it the type of the class you want to make a double of.
* Asking the resulting 'controller' to generate a Test Double for you.

```java
    Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
    DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();
```

The result is an object that can be used in place of the object being stubbed.

Every non-static public method on the class is available, and when called, will do nothing.

If a method has a return value, then `null` will be returned.

You then use the Test Double as you would any other instance of the object.

In this example, we're testing `scheduleDelivery` on the class `DeliveryOrder`.

The method takes a `DeliveryProvider` as a parameter, calls methods against it and then returns `true` when the delivery is successfully scheduled:

```java

Test.startTest();
    DeliveryOrder order = new DeliveryOrder().setDeliveryDate( deliveryDate ).setDeliveryPostcode( deliveryPostcode );
    Boolean scheduled = order.scheduleDelivery( deliveryProviderDouble );
Test.stopTest();

Assert.isTrue( scheduled, 'scheduleDelivery, when called for a DeliveryOrder that can be delivered, will check the passed provider if it can deliver, and return true if the order can be' );

```

What we need to do, is configure our Test Double version of `DeliveryProvider` so that tells the order that it can deliver, and then allows the order to schedule it.

### Configuring return values

We can configure the Test Double, so that certain methods return certain values by calling `when`, `method` and`willReturn` against the controller.

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .willReturn( true )
    .also().when( 'scheduleDelivery' )
        .willReturn( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

Now, whenever either `canDeliver` or `scheduleDelivery` are called against our double, `true` will be returned.  This is regardless of what parameters have been passed in.

#### Responding based on parameters

If we want our Test Double to be a little more strict about the parameters it recieves, we can also specify that methods should return particular values *only when certain parameters are passed* by using methods like `withParameter`, `thenParameter`:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .withParameter( deliveryPostcode )
        .thenParameter( deliveryDate )
        .willReturn( true )
    .also().when( 'canDeliver' )
        .willReturn( false );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

We can also use a 'named parameter' notation, by using the methods `withParameterNamed` and `setTo`:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .willReturn( true )
    .also().when( 'canDeliver' )
        .willReturn( false );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

**Note on Named Parameters:** Matching parameters by name (e.g., using `withParameterNamed` or retrieving with `parameter('name')`) depends on the Apex runtime providing these names to the framework. While generally reliable, there might be scenarios (e.g., with code compiled on much older API versions) where parameter names are not available at runtime. In such cases, Amoss would not be able to match by name. If you encounter issues, consider using positional parameter matching (e.g., `withParameter(value).thenParameter(anotherValue)`) for greater certainty, as it relies on the order of parameters which is always available.

If we want to be strict about some parameters, but don't care about others we can also be flexible about the contents of particular parameters, using `withAnyParameter`, and `thenAnyParameter`:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .withParameter( deliveryPostcode )
        .thenAnyParameter()
        .willReturn( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

Or, when using 'named notation', we can simply omit them from our list:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode ) // since we don't mention 'deliveryDate', it can be any value
        .willReturn( true )
    .also().when( 'canDeliver' )
        .willReturn( false );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

This is very useful for making sure that our tests only configure the Test Doubles to care about the parameters that are important for the tests we are running, making them less brittle.

There are several ways of specifying the expected values of parameters, and the details are covered later in the documentation.

### Using the Test Double as a Spy

The controller can then be used as a Test Spy, allowing us to find out what values where passed into methods by using `latestCallOf` and `call` and `get().call()`:

```java

Assert.areEqual( deliveryPostcode, deliveryProviderController.latestCallOf( 'canDeliver' ).parameter( 0 )
                    , 'scheduling a delivery, will call canDeliver against the deliveryProvider, passing the postcode required, to find out if it can deliver' );

Assert.areEqual( deliveryDate, deliveryProviderController.call( 0 ).of( 'canDeliver' ).parameter( 1 )
                    , 'scheduling a delivery, will call canDeliver against the deliveryProvider, passing the date required, to find out if it can deliver' );

```

You can also retrieve all calls to a method and iterate through them, or check the total count of calls, using standard Apex `Assert` methods for verification.

For instance, if `scheduleDelivery` was expected to be called multiple times with different parameters:

```java
// ... (Amoss_Instance setup as before) ...
// deliveryProviderDouble is used by the system under test, which calls scheduleDelivery twice

// Verify the total number of calls
Assert.areEqual(2, deliveryProviderController.countOf('scheduleDelivery'), 'scheduleDelivery should have been called twice.');

// Verify parameters of the first call (index 0)
Amoss_Instance.Call firstCall = deliveryProviderController.call(0).of('scheduleDelivery');
Assert.areEqual(expectedPostcode1, firstCall.parameter('postcode'), 'First call to scheduleDelivery had incorrect postcode.');
Assert.areEqual(expectedDate1, firstCall.parameter('deliveryDate'), 'First call to scheduleDelivery had incorrect deliveryDate.');

// Verify parameters of the second call (index 1)
Amoss_Instance.Call secondCall = deliveryProviderController.call(1).of('scheduleDelivery');
Assert.areEqual(expectedPostcode2, secondCall.parameter('postcode'), 'Second call to scheduleDelivery had incorrect postcode.');
Assert.areEqual(expectedDate2, secondCall.parameter('deliveryDate'), 'Second call to scheduleDelivery had incorrect deliveryDate.');
```
This demonstrates how Amoss captures interaction details, which are then asserted using the standard Apex `Assert` class.

This can be very useful if the order and completeness of processing is absolutely vital.

A single object can be used as both a Mock and a Test Spy at the same time:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .expects( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .returning( true )
    .then().expects( 'scheduleDelivery' )
        .withParameter( deliveryPostcode )  // once again, either syntax is fine
        .thenParameter( deliveryDate )
        .returning( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();
```

The `verify()` method ensures that all expected calls on the mock object occurred in the specified order and with the correct parameters. However, you will often also want to use standard Apex `Assert` methods to check the state of your system under test *after* these interactions have taken place.

For example, if the `DeliveryOrder`'s `scheduleDelivery` method, after successfully interacting with the `DeliveryProvider` mock, is supposed to update its own status:
```java
// ... (Amoss_Instance setup and DeliveryOrder.scheduleDelivery call as before)
// DeliveryOrder order = new DeliveryOrder(); // Example instantiation
// order.scheduleDelivery(deliveryProviderDouble); // Example call

// Verify interactions with the mock
deliveryProviderController.verify();

// Verify the outcome on the system under test using standard Apex assertions
// Assume 'order' is the instance of DeliveryOrder that was subject to the test
Assert.isTrue(order.isScheduled(), 'Order status should be scheduled after successful delivery scheduling.');
Assert.areEqual('Scheduled', order.getStatus(), 'Order should have the expected status.');
// Any other relevant state checks on 'order' or related objects
```
In this way, Amoss's mock verification complements standard assertions to provide comprehensive testing.

This can be very useful if the order and completeness of processing is absolutely vital.

A single object can be used as both a Mock and a Test Spy at the same time:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .expects( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .returning( true )
    .also().when( 'scheduleDelivery' )
        .willReturn( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...

deliveryProviderController.verify();

Assert.areEqual( deliveryPostcode, deliveryProviderController.latestCallOf( 'scheduleDelivery' ).parameter( 'postcode' )
                    , 'scheduling a delivery, will call scheduleDelivery against the deliveryProvider, passing the postcode required' );
```

This will then allow `scheduleDelivery` to be called at any time (and any number of times), but `canDeliver` must be called with the stated parameters, and must be called exactly once.

### Configuring a stricter Test Double that isn't a Mock Object

If you like the way Test Spies have clear assertions, but don't want just any method to be allowed to be called on your Test Double, you can use `allows`

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .allows( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .returning( true )
    .also().allows( 'scheduleDelivery' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .returning( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

This means that `canDeliver` and `scheduleDelivery` can be called in any order, and do not *have* to be called, but that *only* these two methods with these precise parameters can be called, and any other method called against the Test Double will result a test failure.

## Specifying return values in different ways

### `returnsItself` / `returningItself` / `willReturnItself`

If you wish to define a method as returning 'this' - for example, if your class implements a flient interface, you can use `returnsItself` or one of its synonyms.

For example, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .when( 'fluentMethod' )
	.returnsItself();

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( classDouble, classDouble.fluentMethod() );
```

### `isFluent`

Defining a controller as `isFluent` will ensure that all otherwise unspecified method calls will return an instance of the generated Test Double.

For example, given that `AmossTest_ClassToDouble` has a method `fluentMethod` that has a return type of `AmossTest_ClassToDouble`, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .isFluent();

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( classDouble, classDouble.fluentMethod() );
```

The specification of any `when`, `allows` or `expects` against a method will override the return value of that method for the specified parameter configuration.

For example, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .isFluent()
    .when( 'fluentMethod' )
        .returns( null );

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( null, classDouble.fluentMethod() );
```

This is the case even if no return is specified for the method.  For example, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .isFluent()
    .when( 'fluentMethod' );

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( null, classDouble.fluentMethod() );
```

If you wish to define a method as being fluent in such a scenario, you can use `returningItself` or one of its synonyms.  For example, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .isFluent()
    .when( 'fluentMethod' )
	.returnsItself();

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( classDouble, classDouble.fluentMethod() );
```

It should be noted that if a method is called that has an incompatible type, then a "System.TypeException: Invalid conversion from runtime type..." exception will be thrown.  Currently, there is no way for Amoss to detect and stop this exception from occurring.  If Salesforce provides the capabiliy to stop this from occurring in the future, then the library will be updated to more helpfully describe the issue, or stop it from occurring.


#### Behaviour with `createClone` and `generateDouble`

If a new controller is cloned from a pre-existing one (I.E. by using `createClone`), or multiple Test Doubles are generated (I.E. by using multiple calls to `generateDouble` against the same controller), each instance will continue to return the appropriate `this` from each fluent method.

For example, the following is true:

```java

Amoss_Instance classToDoubleController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classToDoubleController
    .isFluent();

AmossTest_ClassToDouble classToDouble1 = (AmossTest_ClassToDouble)classToDoubleController.getDouble();
AmossTest_ClassToDouble classToDouble2 = (AmossTest_ClassToDouble)classToDoubleController.generateDouble();

Test.startTest();
    AmossTest_ClassToDouble returnFromDouble1 = classToDouble1.fluentMethod();
    AmossTest_ClassToDouble returnFromDouble2 = classToDouble2.fluentMethod();
Test.stopTest();

Assert.areEqual( classToDouble1, returnFromDouble1 );
Assert.areEqual( classToDouble2, returnFromDouble2 );
Assert.areNotEqual( returnFromDouble1, returnFromDouble2 );

```

And, the following is true:

```java

Amoss_Instance classToDoubleController1 = new Amoss_Instance( AmossTest_ClassToDouble.class );
Amoss_Instance classToDoubleController2 = classToDoubleController1.createClone();

AmossTest_ClassToDouble classToDouble1 = (AmossTest_ClassToDouble)classToDoubleController1.getDouble();
AmossTest_ClassToDouble classToDouble2 = (AmossTest_ClassToDouble)classToDoubleController2.getDouble();

Test.startTest();
    AmossTest_ClassToDouble returnFromDouble1 = classToDouble1.fluentMethod();
    AmossTest_ClassToDouble returnFromDouble2 = classToDouble2.fluentMethod();
Test.stopTest();

Assert.areEqual( classToDouble1, returnFromDouble1 );
Assert.areEqual( classToDouble2, returnFromDouble2 );
Assert.areNotEqual( returnFromDouble1, returnFromDouble2 );

```

### `byDefaultMethodsReturn`

Stating that `byDefaultMethodsReturn` will set the default value of any method calls that are not otherwise specified against the controller.

For example, given that `AmossTest_ClassToDouble` has a method `methodUnderDouble` that has a return type of `String`, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .byDefaultMethodsReturn( 'ThisDefaultValue' );

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( 'ThisDefaultValue', classDouble.methodUnderDouble( '1', 2 ) );
```

As with `isFluent`, the specification of any `when`, `allows` or `expects` against a method will override the return value of that method for the specified parameter configuration.  This is true whether a return is specified for the method or not.

For example, the following is true:

```java

Amoss_Instance classController = new Amoss_Instance( AmossTest_ClassToDouble.class );
classController
    .byDefaultMethodsReturn( 'ThisDefaultValue' );
    .when( 'methodUnderDouble' );

AmossTest_ClassToDouble classDouble = (AmossTest_ClassToDouble)classController.getDouble();

Assert.areEqual( null, classDouble.methodUnderDouble( '1', 2 ) );
```

## Specifying parameters in different ways

All of the below can be used with either `withParameter` or `withParameterNamed`.

### `setTo`

In general, will check that the expected and passed values are the same instance, unless object specific behaviour has been defined.

It is probably the most common method of checking values, particularly when you care that the Sobjects / Objects / collections are the same instance and therefore may be mutated by the methods correctly - for example, when you are testing trigger handlers that aim to mutate the trigger context variables.

In detail, it checks that the passed parameter:

* If an Sobject / List / Set / Map, equals the expected, as per the behaviour of '===', being:
  * That the expected and passed objects are the same instance.

* Otherwise, as per the behaviour of '==', being:
  * If the parameter is a primitive -  that the value is the same.
  * If the parameter is an Object that does not implement `equals`:
    * That the expected and passed objects are the same instance.
  * If the parameter is an Object that does implement `equals`:
    * That the return of `equals` is true.

Note: the specification of `withParmeter( value )` is shorthand for `withParameter().setTo( value )`.

### `setToTheSameValueAs`

Attempts to check that the expected and passed values evaluate to the same value, regardless of whether they are the same instance.

Used when you don't have access to the instances that are likely to be passed, or if it is unimportant that the objects are new instances.  For example, it may be used to check the values of parameters where the objects are constructed within the method under test.

In detail, it checks that the expected and passed values equal each other when serialised as JSON strings.

You should note that this may not be reliable in all situations, but should suffice for the majority of use cases.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().setToTheSameValueAs( anObject )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).setToTheSameValueAs( anObject )
        .willReturn( 'theReturn' );
```

### `set`

Attempts to check that the passed value is set to a not null value.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().set()
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).set()
        .willReturn( 'theReturn' );
```

### `containing`

Checks that the passed value is a String, which contains the given String - matching in a case sensitive way.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().containing( 'AnExpectedString' )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).containing( 'AnExpectedString' )
        .willReturn( 'theReturn' );
```

### `matching`

Checks that the passed value is a String, which fully 'matches' the given regular expression - matching in a case sensitive way.

Note that the whole of the String must match the regular expression, rather than a fragment of string matching, as per the behaviour of `Matcher.matches`.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().matching( 'OPP-[0-9]+' )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).matching( 'OPP-[0-9]+' )
        .willReturn( 'theReturn' );
```

### sObject Specific Comparisons

#### `withFieldsSetLike` / `withFieldsSetTo`

Used to check the field values of sObjects when only some of the fields are important.  For example, you may check that certain fields are populated by the method under test before passing them into the method being doubled.  This allows you to specify the fields that will be set without concerning your test with the other values, which will be incidental.

* `withFieldsSetLike` - Receives an sObject
* `withFieldsSetTo` - Receives a `Map<String,Object>`

For each of the properties set on the 'expected' object, the passed sObject is checked.  Only if all the specified properties match will the passed object 'pass'.

The passed object may have more properties set, and they can have any value.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().withFieldsSetLike( new Contact( FirstName = 'theFirstName', LastName = 'theLastName' ) )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'theFirstName', 'LastName' => 'theLastName' } )
        .willReturn( 'theReturn' );
```

### List Specific Comparisons

#### `aListOfLength`

Used to check that a parameter is a list that consists of the specified number of elements.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().aListOfLength( 1 )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).aListOfLength( 2 )
        .willReturn( 'theReturn' );
```

#### `withAnyElement`

Used to check that a parameter is a list that contains *any* of the elements passing the specified condition.  It can be used with any of the matching methods that you can use directly on the parameter (e.g. `setTo`, `setToTheSameValueAs`, etc), with the exception of the other list comparisons (I.E. you cannot check a list within a list.  Yet).

* `withAnyElement` - Requires a further condition to be defined.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().withAnyElement().setTo( 'expectedString' )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).withAnyElement().withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'theFirstName', 'LastName' => 'theLastName' } )
        .willReturn( 'theReturn' );
```

#### `withAllElements`

Used to check that a parameter is a list where *all* of the elements pass the specified condition.  It can be used with any of the matching methods that you can use directly on the parameter (e.g. `setTo`, `setToTheSameValueAs`, etc), with the exception of the other list comparisons (I.E. you cannot check a list within a list.  Yet).

* `withAllElements` - Requires a further condition to be defined.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter().withAllElements().setTo( 'expectedString' )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' ).withAllElements().withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'theFirstName', 'LastName' => 'theLastName' } )
        .willReturn( 'theReturn' );
```

#### `withElementAt`

Used to check that a parameter is a list where the element at the given position passes the specified condition.  It can be used with any of the matching methods that you can use directly on the parameter (e.g. `setTo`, `setToTheSameValueAs`, etc), with the exception of the other list comparisons.

* `withElementAt` - Requires an element position to be defined, followed by a further condition.

Examples:
```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter()
            .withElementAt( 0 ).setTo( 'expectedString-number1' )
            .withElementAt( 1 ).setTo( 'expectedString-number2' )
        .willReturn( 'theReturn' );

classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterNamed( 'parameterName' )
            .withElementAt( 0 ).withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'Person1' } )
            .withElementAt( 1 ).withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'Person2' } )
        .willReturn( 'theReturn' );
```

#### Combining list specifications

All of the list based specifications can be combined, providing the opportunity to create quite complex parameter checking with simple structures.

For example, you may want to check that every Contact in a list has the 'PersonAccount' flag set, and then also check the particular Names of the Contacts at the same time, maybe to ensure they are passed in a particular order.  You might do that with:

```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameter()
            .withAllElements().withFieldsSetTo( new Map<String,Object>{ 'IsPersonAccount' => true } )
            .withElementAt( 0 ).withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'Person1' } )
            .withElementAt( 1 ).withFieldsSetTo( new Map<String,Object>{ 'FirstName' => 'Person2' } )
        .willReturn( 'theReturn' );
```

### Custom Verifiers

If the standard means of verifying parameters doesn't give you the level of control you need, you might consider writing a custom verifier.

Before you do so, you might consider if you would be better served by making your Test Double's behaviour less specific and using it as a Test Spy (check parameters after the call).  Doing so will likely result in a more readable test that is less brittle.

That said, if you need to, you can implement your own implementations of `Amoss_ValueVerifier` and pass it into the specification using `verifiedBy`.

For example:

```java
classToDoubleController
    .when( 'objectMethodUnderDouble' )
        .withParameterName( 'parameter1' ).verifiedBy( customVerifier )
        .willReturn( 'theReturn' );
```

An implementation must implement two methods:

* `toString` - A string representation that will be used when describing the expected call in a failed verify call against the Test Double's controller.
* `verify` - The method that will check the given value 'matches' the expected.

### Writing the `verify` method

`verify` is used in two ways:
* Checking that a method call matches an expectation, and ultimately issuing a failing assertion if it doesn't.
* Checking if a 'when' is applicable for a particular method call.

In order to ensure that that the method works in both situations, `verify` should check that the passed given value passes verification, by reporting any via throwing an exception of one the following types:
    * `Amoss_Instance.Amoss_AssertionFailureException`
    * `Amoss_Instance.Amoss_EqualsAssertionFailureException`

These exceptions are then either caught and resolved as a 'mis-match', or converted into a failed assertion.

When the Exception is raised, `setAssertionMessage` should be called to clearly define the failure.

E.g.

```java
throw new Amoss_Instance.Amoss_AssertionFailureException()
                                .setAssertionMessage( 'Value should be a Map indexed by Date, containing if each is a bank holiday' )
                                // or some other complex check
```

When using Amoss_EqualsAssertionFailureException, setExpected and setActual should also be set, with the values
being relevant within the context of the stated assertionMessage.

E.g.

```java
throw new Amoss_Instance.Amoss_EqualsAssertionFailureException()
                                .setExpected( 'Map<Date,Boolean>' )
                                .setActual( actualType )
                                .setAssertionMessage( 'Value should be a Map indexed by Date, containing if each is a bank holiday.  Was not the expected Type.' )
```
(Note that `setAssertionMessage` returns a `Amoss_AssertionFailureException`, so it is easiest to order the method calls this way round)

If other verifiers are used within a custom verifier, any `Amoss_AssertionFailureExceptions` can be caught and
have context added to the failure by calling addContextToMessage against the exception before re-throwing.

Care should be taken to ensure that no exceptions other than `Amoss_AssertionFailureExceptions` and its subclasses are
thrown.  This ensures that failures are clearly reported to the user.

In addition, no calls to `System.assert` or its variations should be made directly in this method otherwise unexpected behaviours may result, particularly when using the 'when' and 'allows' syntax.

## Other Behaviours

### Using Method Handlers

In some situations it is not enough to define a response statically in this way.

For example, you may require a response that is based on the values of a passed in parameter in a more dynamic way - like the Id of an Sobject that is passed in.

Also, you want to take advantage of some of the parameter checking and verify mechanisms of the framework for existing tests that currently use the standard Salesforce `StubProvider`.

For those situations, you can use `handledBy` in order to specify an object that will handle the call, perform processing and generate a return.

This method can take one of two types of parameter:
* `StubProvider` - Providing the full capabilities of the StubProvider interface means that you can re-use any pre-existing test code, as well as write new classes that utilise the full set of parameters (`methodName`, `parameterTypes`, etc), if so required.
* `Amoss_MethodHandler` - A much simpler version of a `StubProvider`-like interface means that you can create handler methods that are focused entirely on the parameter values of the called method.

#### Example using Amoss_MethodHandler

For example, you are testing a method that uses a method on another object (`ClassBeingDoubled.getContactId`) to get the Id from a Contact.  This method has a single parameter - the Contact.

You may implement this be defining an implementation of Amoss_MethodHandler and using that in your Test Double's definition:

```java
class ExampleMethodHandler implements Amoss_MethodHandler {
    public Object handleMethodCall( List<Object> parameters ) {
        Contact passedContact = (Contact)parameters[0];
        return passedContact.Id;
    }
}

@isTest
private static void methodBeingTested_whenGivenSomething_doesSomething() {

    Amoss_MethodHandler methodHander = new ExampleMethodHandler();

    Amoss_Instance objectBeingDoubledController = new Amoss_Instance( ClassBeingDoubled.class );

    objectBeingDoubledController
        .when( 'getContactId' )
            .withAnyParameter()
            .handledBy( methodHander );

    ...
```

Notice that you can still use `withParameter` and the related methods in order to specify the situations in which the handler will be called, as well as any of the other `expects`, `allows` and spy capabilities.

#### Example using StubProvider

Alternatively, you may define the handler class using the `StubProvider` interface.

E.g.

```java
class ExampleMethodHandler implements StubProvider {

    public Object handleMethodCall( Object       mockedObject,
                                    String       mockedMethod,
                                    Type         returnType,
                                    List<Type>   parameterTypes,
                                    List<String> parameterNames,
                                    List<Object> parameters ) {

        Contact passedContact = (Contact)parameters[0];
        return passedContact.Id;
    }
}

@isTest
private static void methodBeingTested_whenGivenSomething_doesSomething() {

    StubProvider methodHander = new ExampleMethodHandler();

    Amoss_Instance objectBeingDoubledController = new Amoss_Instance( ClassBeingDoubled.class );

    objectBeingDoubledController
        .when( 'getContactId' )
            .withAnyParameter()
            .handledBy( methodHander );

    ...
```

That is, `handledBy` is overloaded, and can take either definition type.  The behaviours being identical to each other.

### Throwing exceptions

Test Doubles can also be told to throw exceptions, using `throwing`, `throws` or `willThrow`:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .when( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .throws( new DeliveryProvider.DeliveryProviderUnableToDeliverException( 'DeliveryProvider does not have a delivery slot' ) );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

### Don't allow any calls

In some situations you may want to create a Mock Object that ensures that no calls are made against it.

For that, you can use `expectsNoCalls`.

This method cannot be used in conjunction with any other method definition (`expects`, `allows`, `when`). If an attempt is made to do so, an exception is thrown.

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .expectsNoCalls();

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();

...
```

It is valid to call `verify` against the controller at the end of the test, but this will always pass since the expected call stack will always be empty.

### `getDouble` / `generateDouble` / `createClone`

There are two mechanisms for retrieving the Test Double from the controller:
* `getDouble`
* `generateDouble`

In must situations, you will only ever retrieve a single double from a single controller, and in that instance there is no functional difference between `getDouble` and `generateDouble`.

The functional difference only appears on the second and subsequent calls to these methods.

#### `getDouble`

Will return the same Test Double as the previous call to either `getDouble` or `generateDouble`.

That is, the following is true:

```java

Amoss_Instance classToDoubleController = new Amoss_Instance( AmossTest_ClassToDouble.class );

AmossTest_ClassToDouble classToDouble1 = (AmossTest_ClassToDouble)classToDoubleController.getDouble();
AmossTest_ClassToDouble classToDouble2 = (AmossTest_ClassToDouble)classToDoubleController.getDouble();

Assert.areEqual( classToDouble1, classToDouble2 );

```

There is not normally any reason to call `getDouble` multiple times, since you can, of course, store the result of the first `getDouble` call in a variable and use that variable multiple times.

#### `generateDouble`

Will return a new instance of the Test Double, tied to the same `Amoss_Instance` as any previously generated Test Doubles.

That is, the following is true:

```java

Amoss_Instance classToDoubleController = new Amoss_Instance( AmossTest_ClassToDouble.class );

AmossTest_ClassToDouble classToDouble1 = (AmossTest_ClassToDouble)classToDoubleController.getDouble();
AmossTest_ClassToDouble classToDouble2 = (AmossTest_ClassToDouble)classToDoubleController.generateDouble();

Assert.areNotEqual( classToDouble1, classToDouble2 );

classToDouble1.methodUnderDouble( '1', 2 );
classToDouble2.methodUnderDouble( '1', 2 );

Assert.areEqual( 2, classToDoubleController.countOf( 'methodUnderDouble' ) );

```

This is particularly useful when you need multiple instances of the same object with the same behavior configuration, but want to track their invocations separately using the same controller. For example, when testing a method that processes a list of objects, each requiring the same mock dependency.

#### `createClone`

Any `Amoss_Instance` can be cloned in order to create an independent controller that can then generate its own Test Doubles.  The cloned controller will have the same initial configuration as the controller it was cloned from, including any defined `when`, `allows` and `expects` behaviours.

However, once cloned, there is no link between the original and the cloned controller and they do not interact in any way.

That is, the following is true:

```java

Amoss_Instance classToDoubleController1 = new Amoss_Instance( AmossTest_ClassToDouble.class );
Amoss_Instance classToDoubleController2 = classToDoubleController1.createClone();

AmossTest_ClassToDouble classToDouble1 = (AmossTest_ClassToDouble)classToDoubleController1.getDouble();
AmossTest_ClassToDouble classToDouble2 = (AmossTest_ClassToDouble)classToDoubleController2.getDouble();

Assert.areNotEqual( classToDouble1, classToDouble2 );

classToDouble1.methodUnderDouble( '1', 2 );
classToDouble2.methodUnderDouble( '1', 2 );

Assert.areEqual( 1, classToDoubleController1.countOf( 'methodUnderDouble' ) );
Assert.areEqual( 1, classToDoubleController2.countOf( 'methodUnderDouble' ) );

```

Cloning is useful when you need to create multiple instances of an object with the same base configuration but that will evolve differently during the test or need to be verified independently.

### Longer format .method definition

Occassionally you may find that the definition of the method name gets swamped by the other definitions.  In that situation you may want to use the slightly longer form `when().method( methodName )`, to more clearly highlight the methods being defined.

For example:

```java

Amoss_Instance deliveryProviderController = new Amoss_Instance( DeliveryProvider.class );
deliveryProviderController
    .expects()
        .method( 'canDeliver' )
        .withParameterNamed( 'postcode' ).setTo( deliveryPostcode )
        .andParameterNamed( 'deliveryDate' ).setTo( deliveryDate )
        .returning( true )
    .also().when()
        .method( 'scheduleDelivery' )
        .willReturn( true );

DeliveryProvider deliveryProviderDouble = (DeliveryProvider)deliveryProviderController.getDouble();
```
The `method` variation is available for all three method definition scenarios: `when`, `allows` and `expects`.

## What? When?

With such flexibility, it's important that you make good decisions on which behaviours to use, and when.

The decision will be based on a balance of two main factors:
* Ensuring that you produce a meaningful test of the behaviour, whilst
* Limiting the scope of test changes that are required when the implementation of the classes that are being stubbed or tested change.

The following aims to describe when to each of the constructs, roughly referencing the types of "Test Doubles" that are described in Gerard Meszaros's book "xUnit Test Patterns".

### Test Stub

Is the least brittle of the constructs, allowing any method to be called, potentially with any parameters.  In most cases, some return values for some methods will be specified.

Typically used to replace ('stub out') an object that is not the focus of the test, but on which the test relies, often in order to direct the object under test into a specificaly required behaviour.

That is, the object provides some functionality that means that the test can run, but it is not the calling of methods on this object that define the behaviour that is being tested.

However, it is likely that the test will check that the method under tests acts in a particular way when it receives certain values from the methods being stubbed.

It is of particular note that when implemented using 'withAnyParameter' (or without parameters being specified), the method signatures of the object being stubbed can change without impacting the tests.

#### Is characterised by the pattern: when.with.willReturn

```java

testDoubleController
    .when( 'methodName1' )
        .withAnyParameters()    // this is actually redundant, as the default behaviour is 'withAnyParameters'
        .willReturn( true )
    .also().when( 'methodName2' )
        .withAnyParameters()
        .willReturn( true );
```

If return values do not need to be specified, may be as simple as:

```java
    ObjectUnderTestDouble testDouble = (ObjectUnderTestDouble)( new Amoss_Instance( ObjectUnderTestDouble.class ).getDouble() );
```

#### Brittle?

* Additions to the interface of the object being stubbed will not break the test, unless particular return values are required in the test.
* Changes to the interface of existing methods will not break tests when `withAnyParameters` or `withAnyParameter` are used and the parameters do not need to be reflected in the return values.
* Changes to the interface of existing methods may break tests when parameter values are specified in the configuration.
* Generally not affected by changes the implementation of the method under test that affect the order of processing in, or number of method calls made by the method under test.
* Is affected by changes in the implementation of the method under test where the change results in different values being returned by the methods being stubbed.

### Strict Test Stub

Similar to a Test Stub, although is defined in such a way that *only* the methods that are configured are allowed to be called.

Also typically used when the object being stubbed is not the focus of the test, and potentially the parameter values being passed in are not of importance.

However, it is implied that it is important that *other* methods, those not specified, are *not* called.

Is not particularly brittle to changes in the implementation of the class being 'stubbed'.  However, tests may start to break when the implementation under test changes and new methods are called against that object.

#### Is characterised by the pattern: allows.with.returning

```java

testDoubleController
    .allows( 'methodName1' )
        .withAnyParameters()
        .returning( true )
    .also().allows( 'methodName2' )
        .withAnyParameters()
        .returning( true );
```

#### Brittle?

* Additions to the interface of the object being stubbed **will** break the test if those methods are called by the method under test.
* Changes to the interface of existing methods will not break tests when `withAnyParameters` or `withAnyParameter` are used and the parameters do not need to be reflected in the return values.
* Changes to the interface of existing methods may break tests when parameter values are specified in the configuration.
* Generally not affected by changes to the order of processing in, or number of method calls made by the method under test.
* Is affected by changes in the implementation of the method under test where the change results in additional methods being called against the object being stubbed.
* Is affected by changes in the implementation of the method under test where the change results in different values being returned by the methods being stubbed.

### Test Spy

Is specified intially in the same way as a Test Stub, though after the method under test is executed, the controller is then interrogated to determine the value of parameters.

Typically used to create a Test Double of an object that *is* the focus of the test.

That is, the test is checking that the method under test calls particular methods on the given Test Double passing parameters with certain values that are predictable.

Can be used to test that individual methods are called, and the order in which that particular method is called.  However, cannot be used to test that different methods are called in a particular sequence.  In order to do that, a Mock Object (see below) is required.

#### Is characterised by the pattern: when.with.willReturn followed by call.of.parameter

```java

spiedUponObjectController
    .when( 'methodName1' )
        .withAnyParameters()
        .willReturn( true )
    .also().when( 'methodName2' )
        .withAnyParameters()
        .willReturn( true );

// followed by

Assert.areEqual( 'expectedParameterValue1',
                        spiedUponObjectController.latestCallOf( 'method1' ).parameter( 'parameter1' ),
                        'methodUnderTest, when called will pass "expectedParameterValue1" into "method1"' );

Assert.areEqual( 'expectedParameterValue2',
                        spiedUponObjectController.call( 0 ).of( 'method2' ).parameter( 'parameter2' ),
                        'methodUnderTest, when called will pass "expectedParameterValue2" into "method2"' );

```

### Strict Test Spy

Similar to a Test Spy, although is defined in such a way that *only* the methods that are configured are allowed to be called.

As with the Test Spy, is used to create a Test Double an object that *is* the focus of the test.

That is, the test is checking that the method under test calls particular methods on the given Test Double passing parameters with certain values that are predictable.

Can be used to test that individual methods are called, and the order in which that particular method is called.  However, cannot be used to test that different methods are called in a particular sequence.  In order to do that, a Mock Object (see below) is required.

It is implied that it is important that *other* methods, those not specified, are *not* called.

#### Is characterised by the pattern: allows.with.willReturn followed by call.of.parameter

```java

spiedUponObjectController
    .allows( 'methodName1' )
        .withAnyParameters()
        .willReturn( true )
    .also().allows( 'methodName2' )
        .withAnyParameters()
        .willReturn( true );

// followed by

Assert.areEqual( 'expectedParameterValue1',
                        spiedUponObjectController.latestCallOf( 'method1' ).parameter( 'parameter1' ),
                        'methodUnderTest, when called will pass "expectedParameterValue1" into "method1"' );

Assert.areEqual( 'expectedParameterValue2',
                        spiedUponObjectController.call( 0 ).of( 'method2' ).parameter( 'parameter2' ),
                        'methodUnderTest, when called will pass "expectedParameterValue2" into "method2"' );

```

### Mock Object

Similar to a Test Spy, although is defined in such a way that *only* the methods that are configured are allowed to be called, and only in the order that they are specified.

If any specified method is called out of order, or with the wrong parameters, it will fail the test.

Is therefore used to 'mock' an object when the order of execution of different methods is important to the success of the test.

It is also implied that it is important that *other* methods, those not specified, are *not* called.

Because of the strict nature of the specification, this is the most brittle of the constructs, and often results in tests that fail when the implementation of the method under test is altered.

#### Is characterised by the pattern: expects.with.willReturn followed by verify

```java

mockObjectController
    .expects( 'methodName1' )
        .withParameter( 'expectedParameterValue1' )
        .willReturn( true )
    .then().expects( 'methodName2' )
        .withParameter( 'expectedParameterValue2' )
        .willReturn( true );

// followed by
mockObjectController.verify();

```

### Summary

In all cases, 'willReturn' or 'returning' could be replaced with 'throws' or 'handledBy' without changing the categorisation of the Test Double in question.

Type                | Use Cases | Brittle? | Construct Pattern
------------------- | --------- | -------- | ------------------------
Test Stub           | Ancillary objects, parameters passed are not the main focus of the test             | Least brittle | when.with.willReturn
Strict Test Stub    | Ancillary objects, parameters passed are not the main focus of the test             | Brittle to addition of new calls on object being stubbed | allows.with.returning
Test Spy            | Focus of the test, order of execution is not important, prefer the assertion syntax | Is brittle to the interface of the object being stubbed, less brittle to the implementation of the method under test | when.with.willReturn, call.of.parameter
Strict Test Spy     | Focus of the test, order of execution is not important, prefer the assertion syntax | Is brittle to the interface of the object being stubbed and addition of new calls on object being stubbed, a little more brittle to the implementation of the method under test | allows.with.returns, call.of.parameter
Mock Object         | Focus of the test, order of execution is important                                  | Most brittle construct, brittle to the implementation of the method under test | expect.with.returning, verify

## Synonyms

Some of the functions have synonyms, allowing you to choose the phrasing that is most readable for your team.

Purpose                                                            | Synonyms
------------------------------------------------------------------ | ---------------------------------
Specifying individual parameters (positional notation)             | `withParameter`, `thenParameter` (I.E. start with withParameter, otherwise use thenParameter)
Stating that any single parameter is allowed (positional notation) | `withAnyParameter`, `thenAnyParameter` (I.E. start with withAnyParameter, otherwise use thenAnyParameter)
Specifying individual parameters (named notation)                  | `withParameterNamed`, `andParameterNamed` (I.E. start with withParameterNamed, otherwise use andParameterNamed)
Stating the return value of a method                               | `returning`, `returns`, `willReturn`
Stating that a method throws an exception                          | `throwing`, `throws`, `willThrow`
Start the specification of an additional method                    | `then`, `also` (Generally use 'then' with 'expects', 'also' with 'when' and 'allows' )
Retrieving the parameters from a particular call of a method       | `call`, `get().call`
Retrieving the parameters from the latest call of a method         | `latestCallOf`, `call( -1 ).of`

## Limitations

Since the codebase uses that Salesforce provided StubProvider as its underlying mechanism for creating the Test Double, it suffers from the same fundamental limitations.

Primarily these are:
* The following cannot have Test Doubles generated:
  * Sobjects
  * Classes with only private constructors (e.g. Singletons)
  * Inner Classes
  * System Types
  * Batchables
* Static and Private methods may not be stubbed / spied or mocked.
* Member variables, getters and setters may not be stubbed / spied or mocked.
* Iterators cannot be used as return types or parameter types.

For more information, see here: https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_interface_System_StubProvider.htm

## HttpCalloutMock Support

Amoss provides powerful support for creating `HttpCalloutMocks` using a syntax that is very similar to that which is used to create other Test Doubles.

The `HttpCalloutMock` capabilities are built on top of the core functionality of Amoss, so it takes advantage of most of Amoss's core capabilities, including conditional behaviors, mock object behaviors, and verifications.

### Building a simple `HttpCalloutMock`

The simplest `HttpCalloutMock` that can be created is one that will always return the same result - an empty `HttpResponse` object.

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock();
```

Calling `isACalloutMock` will:
* Tell the Amoss_Instance to create a Test Double of the `HttpCalloutMock` interface
* Register the generated Test Double as the Callout Mock (no need to call `Test.setMock`, Amoss does this for you)

### Defining a default return

If we always want our callout to return the same response, we can define a default response:

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .byDefault()
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
            .body( new Map<String,Object>{ 'parameter' => 'value' } )
            .header( 'ResponseHeaderKey' ).setTo( 'value' )
```

The above defines the full shape of the `HttpResponse` object that is returned whenever a callout is made:

* `respondsWith().status( value )` - Sets the status of the response (`HttpResponse.setStatus( value )`)
* `respondsWith().statusCode( value )` - Sets the status code of the response (`HttpResponse.setStatusCode( value )`)
* `respondsWith().body( value )` - Sets the contents of the body
  * If `value` is a `String`, calls `HttpResponse.setBody( value )`
  * If `value` is a `Blob`, calls `HttpResponse.setBodyAsBlob( value )`
  * If `value` is any other type of `Object`, JSON serializes `value` before passing it to `HttpResponse.setBody`
* `respondsWith().header( key ).setTo( value )` - Sets the value of the specified header (`HttpResponse.setHeader( key, value )`)
* `throws( exception )` / `willThrow( exception )` / `throwing( exception )` - Instructs the mock to throw the given exception when called

### Defining a conditional return

For some tests, we don't want the `HttpCalloutMock` to *always* behave in the same way. We can specify the conditions under which a particular response will be returned:

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .when()
        .method( 'GET' )
        .endpoint().contains( '/account/' )
        .header( 'Authorization' ).isSet()
        .compressed()
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
            .body( new Map<String,Object>{ 'parameter' => 'value' } )
            .header( 'ResponseHeaderKey' ).setTo( 'value' );
```

In this situation, the service will return a 200 status code and the body if and only if these conditions are met:
* The HTTP Method is 'GET'
* The Endpoint contains the string `/account/`
* The Header with the key `Authorization` is set to a non-blank value
* The `HttpRequest` is 'compressed'

Multiple conditions can be defined alongside a default:

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .when()
        .method( 'GET' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
            .body( new Map<String,Object>{ 'parameter' => 'value' } )
    .also().when()
        .method( 'POST' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
    .byDefault()
        .respondsWith()
            .status( 'Not Found' )
            .statusCode( 404 )
```

In this example:
* All GET and POST requests will get a 200 - Complete
* GET requests will also have a body set
* Any other method will result in a 404 - Not Found

### Verification Methods

Several methods are available for verifying the properties of the `HttpRequest`:

* `method( httpMethod )` - Verifies that the request has the specified HTTP method
* `endpoint( uri )` - Verifies that the request has the endpoint set to the precise URI provided
* `header( key ).setTo( value )` - Verifies that the request has the specified header set to the value provided
* `body( value )` - Verifies that the request has the body set to the value provided
* `compressed()` / `notCompressed()` - Verifies that the request is, or is not compressed

For string properties (endpoint, body, header), additional verification options are available:

* `setTo( value )` - Verifies that the property is set to the given value
  ```java
  .endpoint().setTo( 'http://example.com/account/12345' )
  ```

* `set()` - Verifies that the property is set to a non-empty String
  ```java
  .header( 'Authorization' ).set()
  ```

* `containing( value )` - Verifies that the property contains the given value
  ```java
  .endpoint().containing( 'account/12345' )
  ```

* `matching( expression )` - Verifies that the property matches the given regular expression
  ```java
  .endpoint().matching( '.*/account/12345' )
  ```

### Using `expects`, `allows` and Test Spy behaviours with HttpCalloutMocks

Like other Amoss Test Doubles, HttpCalloutMocks can be configured with different levels of strictness:

#### Using `expects` for strict mock behavior

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .expects()
        .method( 'GET' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
    .then().expects()
        .method( 'POST' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 );

Test.startTest();
    // Do the stuff that is tested
Test.stopTest();

// Verify that everything that was expected was called
httpCalloutMock.verify();
```

#### Using `allows` for restricted but unordered calls

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .allows()
        .method( 'GET' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 )
    .also().allows()
        .method( 'POST' )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 );
```

#### Test Spy behaviors for inspecting calls

After a test completes, you can inspect the calls made to your mock:

```java
HttpRequest requestZeroCalledWith = (HttpRequest)httpCalloutMock.get().call(0).of('respond').parameter(0);
HttpRequest latestRequestCalledWith = (HttpRequest)httpCalloutMock.get().latestCallOf('respond').parameter(0);
```

### Custom verifiers and handlers

For complex verification needs, you can use custom verifiers:

```java
httpCalloutMock
    .isACalloutMock()
    .when()
        .verifiedBy( customVerifier )
        .respondsWith()
            .status( 'Complete' )
            .statusCode( 200 );
```

You can implement either:
* `Amoss_ValueVerifier` - The general verifier interface
* `Amoss_HttpRequestVerifier` - A specialized interface for HTTP requests

Similarly, you can use custom handlers to generate responses dynamically:

```java
httpCalloutMock
    .isACalloutMock()
    .expects()
        .method( 'GET' )
        .handledBy( customHandler );
```

You can implement one of:
* `HttpCalloutMock` - The standard Salesforce interface
* `Amoss_MethodHandler` - The simplified Amoss interface
* `StubProvider` - The standard Salesforce stubbing interface

### `expectsNoCalls()` for HTTP mocks

You can ensure no HTTP callouts are made with:

```java
Amoss_Instance httpCalloutMock = new Amoss_Instance();

httpCalloutMock
    .isACalloutMock()
    .expectsNoCalls();
```

This is useful for ensuring that services are not called - for example, checking that guard clauses work correctly.

## Release Notes

Release notes for Amoss can be found [here](./RELEASE_NOTES.md), and changes on this branch that are pending release into the Unlocked Package / Release Tags can be found [here](./PENDING_RELEASE_NOTES.md).

## Acknowledgements / References

Thanks to Aidan Harding (https://twitter.com/AidanHarding), for kickstarting the whole process.  If it wasn't for his post on an experiment he did https://twitter.com/AidanHarding/status/1276512814421639168, this project probably wouldn't have started.

You can find the repo with his experimental implementation here: https://github.com/aidan-harding/apex-stub-as-mock

Also to Martin Fowler for the beautifully succinct article referenced in that tweet - https://martinfowler.com/articles/mocksArentStubs.html

And Gerard Meszaros for his book "xUnit Test Patterns", from which many of the ideas of how Mocks, Spies and Test Doubles should work are taken.

## Troubleshooting

### Common Issues and Solutions

#### "Invalid conversion from runtime type X to Y" Error

When using `returnsItself` or `isFluent` with method return types that don't match the doubled class, you may receive a "System.TypeException: Invalid conversion from runtime type..." exception. This occurs because Amoss cannot prevent type mismatches at runtime. To fix:

1. Check that your fluent methods have the correct return type in the original class
2. Ensure the return type matches the class being doubled
3. For specific methods that need different return types, use explicit `returns` instead of `isFluent`

#### Parameter Verification Failures

If parameter verification fails unexpectedly:

1. Try using `setToTheSameValueAs` instead of `setTo` if you're comparing complex objects
2. Use the appropriate list verifier (`withAllElements`, `withAnyElement`, etc.) for lists
3. For sObjects, consider using `withFieldsSetLike` to check specific fields only

#### Missing Parameter Names

If you're using `withParameterNamed` and experiencing issues:

1. Be aware that parameter names might not be available in very old API versions
2. Use positional parameters (`withParameter().thenParameter()`) instead, which are always available
3. Check that your parameter names exactly match those in the original class

#### Mock Verification Failures

When `verify()` fails but you believe your test is correct:

1. Check that your expectations are defined in the exact order they'll be called
2. Consider using Test Spy patterns (`allows`/`when` + assertions) instead of strict Mock patterns
3. Use `allows` with explicit parameter values instead of `expects` if order is not important 
