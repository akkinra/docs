### Amoss: Apex Mock Objects, Spies, and Stubs

Amoss is a mocking framework for Apex (Salesforce) designed for simplicity and readability in unit testing. It facilitates the creation of Test Doubles (Stubs, Spies, Mocks).

#### Core Concepts: Test Doubles

- **Stub**: Returns predefined values for method calls.
- **Spy**: Records interactions (method calls, parameters) for later verification.
- **Mock**: Expects specific interactions in a defined order and verifies them.

#### Constructing and Using Test Doubles

1.  **Create a Controller**:
    ```apex
    Amoss_Instance controller = new Amoss_Instance(MyClass.class);
    ```
2.  **Get the Test Double**:
    ```apex
    MyClass myDouble = (MyClass)controller.getDouble();
    // Or, for multiple distinct instances tied to the same controller:
    // MyClass myDouble2 = (MyClass)controller.generateDouble();
    ```
    By default, methods on the double do nothing and return `null`.

#### Configuring Behavior (`when`, `allows`, `expects`)

- **`when(methodName)`**: Defines behavior for a stub/spy. Calls are allowed, not enforced.
    ```apex
    controller.when('myMethod').willReturn('someValue');
    ```
- **`allows(methodName)`**: Stricter stub/spy. Only explicitly allowed methods/parameter combinations can be called.
    ```apex
    controller.allows('myMethod').withParameter('specificParam').returning(true);
    ```
- **`expects(methodName)`**: Defines an expectation for a mock. The call must occur, in the specified order if chained with `then()`.
    ```apex
    controller.expects('firstMethod').returning(1)
                .then().expects('secondMethod').withParameter('arg').returning(2);
    ```
- **Chaining configurations**: Use `also()` (typically with `when`/`allows`) or `then()` (typically with `expects`).
- **Longer format**: `when().method('methodName')` (also for `allows()` and `expects()`).

#### Specifying Return Values

- **`willReturn(value)` / `returning(value)` / `returns(value)`**: Specifies the return value.
    ```apex
    controller.when('getName').willReturn('Test Name');
    ```
- **`returnsItself()` / `returningItself()` / `willReturnItself()`**: For fluent interfaces, returns the double instance.
    ```apex
    controller.when('fluentMethod').returnsItself();
    ```
- **`isFluent()`**: Makes all unspecified methods on the controller return the double instance by default.
    ```apex
    controller.isFluent();
    ```
- **`byDefaultMethodsReturn(value)`**: Sets a default return value for all methods not otherwise configured.
    ```apex
    controller.byDefaultMethodsReturn(false);
    ```
- **`throws(exception)` / `willThrow(exception)` / `throwing(exception)`**: Makes the method throw a specified exception.
    ```apex
    controller.when('criticalOperation').throws(new MyException('Failed!'));
    ```

#### Specifying Parameters

Used with `when()`, `allows()`, or `expects()`.

- **Positional Parameters**:
    - `withParameter(value)`: Specifies the first parameter.
    - `thenParameter(value)`: Specifies subsequent parameters.
    - `withAnyParameter()`: Allows any value for the first parameter.
    - `thenAnyParameter()`: Allows any value for subsequent parameters.
    ```apex
    controller.when('calculate').withParameter(10).thenParameter(20).willReturn(30);
    controller.when('process').withParameter('anyId').thenAnyParameter().willReturn(true);
    ```
- **Named Parameters**:
    - `withParameterNamed('paramName').setTo(value)`: Specifies a parameter by name.
    - `andParameterNamed('paramName').setTo(value)`: Specifies subsequent named parameters.
    ```apex
    controller.when('updateUser')
              .withParameterNamed('userId').setTo(id)
              .andParameterNamed('status').setTo('Active')
              .willReturn(true);
    ```
- **Parameter Value Matchers** (chain after `withParameter()`, `withParameterNamed('name')`, etc.):
    - **`setTo(value)`**: Checks for the same instance (for SObjects/Collections) or equality (primitives, objects with `equals` override). _Note: For collections created within the code under test, use `setToTheSameValueAs`._
    - **`setToTheSameValueAs(value)`**: Compares JSON serializations. Useful for objects/collections where instances differ but content should be the same.
    - **`set()`**: Checks if the parameter is not null.
    - **`containing(substring)`**: For String parameters, checks if it contains the substring (case-sensitive).
    - **`matching(regex)`**: For String parameters, checks if it matches the regex.
- **sObject Specific Comparisons**:
    - `withFieldsSetLike(sObjectInstance)`: Checks if the passed sObject has fields matching those set on `sObjectInstance`.
    - `withFieldsSetTo(Map<String,Object> fieldMap)`: Checks if the passed sObject has fields matching the map.
- **List Specific Comparisons**:
    - `aListOfLength(length)`: Checks if the list has the specified number of elements.
    - `withAnyElement().<matcher>(value)`: Checks if any element in the list matches the condition (e.g., `withAnyElement().setTo('A')`).
    - `withAllElements().<matcher>(value)`: Checks if all elements in the list match the condition.
    - `withElementAt(index).<matcher>(value)`: Checks if the element at a specific index matches the condition.
- **Custom Verifiers**:
    - `verifiedBy(customVerifierInstance)`: Uses a custom `Amoss_ValueVerifier` implementation.

#### Using as a Test Spy (Inspecting Calls)

After the code under test has executed:

- **`latestCallOf('methodName').parameter(indexOrName)`**: Gets a parameter from the most recent call to the method.
    ```apex
    String name = (String)controller.latestCallOf('setName').parameter(0);
    String status = (String)controller.latestCallOf('updateUser').parameter('status');
    ```
- **`call(callIndex).of('methodName').parameter(indexOrName)`**: Gets a parameter from a specific call (0-indexed).
    ```apex
    String firstArg = (String)controller.call(0).of('processData').parameter(0);
    ```
- **`countOf('methodName')`**: Gets the number of times a method was called.
    ```apex
    Assert.areEqual(3, controller.countOf('processItem'));
    ```

#### Mock Object Verification

- **`verify()`**: Call after test execution to check if all `expects()` conditions were met in the specified order.
    ```apex
    controller.verify(); // Throws an assertion error if expectations are not met.
    ```

#### Other Behaviors

- **`expectsNoCalls()`**: Asserts that no methods are called on the double. Cannot be combined with other method definitions.
    ```apex
    controller.expectsNoCalls();
    ```
- **Method Handlers (`handledBy`)**: For dynamic responses or complex logic.
    - `handledBy(new MyAmossMethodHandler())` where `MyAmossMethodHandler implements Amoss_MethodHandler` (receives `List<Object> parameters`).
    - `handledBy(new MyStubProvider())` where `MyStubProvider implements System.StubProvider`.
    ```apex
    controller.when('complexCalc').withAnyParameter().handledBy(new MyDynamicHandler());
    ```
- **Cloning Controllers (`createClone`)**: Creates an independent copy of the controller with its current configuration.
    ```apex
    Amoss_Instance clonedController = originalController.createClone();
    ```

#### HttpCalloutMock Support

Amoss simplifies creating `HttpCalloutMock`.

1.  **Initialize**:
    ```apex
    Amoss_Instance httpMock = new Amoss_Instance().isACalloutMock();
    // This also calls Test.setMock(HttpCalloutMock.class, theAmossDouble);
    ```
2.  **Define Default Response**:
    ```apex
    httpMock.byDefault()
            .respondsWith()
            .status('OK')
            .statusCode(200)
            .body('{"message":"default"}')
            .header('Content-Type').setTo('application/json');
    ```
3.  **Define Conditional Responses**:

    ```apex
    httpMock.when()
            .method('GET')
            .endpoint().containing('/users/')
            .header('Authorization').set() // Checks header is present and not blank
            .compressed() // Checks request is compressed
            .respondsWith()
            .statusCode(200)
            .body(new Map<String, Object>{'id' => '123'});

    httpMock.also().when()
            .method('POST')
            .respondsWith()
            .statusCode(201);
    ```

4.  **Request Verification Methods** (used within `when()`, `allows()`, `expects()`):
    - `method(httpMethod)`
    - `endpoint(uri)` / `endpoint().setTo(uri)` / `endpoint().containing(partialUri)` / `endpoint().matching(regex)`
    - `header(key).setTo(value)` / `header(key).set()` / `header(key).containing(value)` / `header(key).matching(regex)`
    - `body(value)` / `body().setTo(value)` / `body().containing(value)` / `body().matching(regex)`
    - `compressed()` / `notCompressed()`
5.  **Using `expects`, `allows`, Spy Behaviors**: Same as regular test doubles.

    ```apex
    httpMock.expects().method('GET').endpoint('/api/data').respondsWith().statusCode(200);
    // ... execute callout ...
    httpMock.verify();

    // Spy:
    // HttpRequest request = (HttpRequest)httpMock.latestCallOf('respond').parameter(0);
    ```

6.  **Custom Verifiers/Handlers**:
    - `verifiedBy(customHttpVerifier)` where `customHttpVerifier` implements `Amoss_HttpRequestVerifier` or `Amoss_ValueVerifier`.
    - `handledBy(customHttpHandler)` where `customHttpHandler` implements `HttpCalloutMock`, `Amoss_MethodHandler`, or `StubProvider`.
7.  **`expectsNoCalls()`**: Ensures no HTTP callouts are made.

#### Synonyms

Amoss provides synonyms for many methods to improve readability (e.g., `returning`, `returns`, `willReturn`; `throwing`, `throws`, `willThrow`).

#### Limitations (due to `System.StubProvider`)

- Cannot create doubles for: SObjects, classes with only private constructors, inner classes, System Types, Batchables.
- Static and private methods cannot be stubbed/spied/mocked.
- Member variables, getters, and setters cannot be stubbed/spied/mocked.
- Iterators cannot be used as return or parameter types.
