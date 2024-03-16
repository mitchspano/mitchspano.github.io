---
layout: post
title: "Pure Unit Testing in Apex"
date: 2022-01-17 19:43:24 -0500
---

# Pure Unit Testing in Apex

Unit tests are something that every Salesforce developer is familiar with. We all need to exceed the 75% code coverage requirements, but why does Salesforce impose this requirement upon us, and what makes a high quality unit test? In this post we will:

- Discuss the fundamental concepts of unit testing
- Look at the quality signals of a good unit test
- Discuss issues with the status quo in the Salesforce ecosystem.
- Share tips and tricks to boost unit test performance

---

## Unit Testing Overview

### What are Unit Tests?

Within a software project, units are the most fine-grained or atomic pieces of software that could be possibly tested. We can identity units within our applications by taking a look at our interfaces and public methods.

A _unit test_ is a piece of code that validates that a unit is working properly. Every time that we write a new unit test, we are leaving a breadcrumb on the trail of our thoughts so that the next generation of developers on our project can understand the original intent of our application.

### How Should Test be Used?

Tests are to be executed whenever a new change is proposed to our application. Executing tests in this way gives us as developers confidence that our newly proposed code changes do not impact historically delivered functionality.

Authoring unit tests which clearly describe the intent of the code and executing them with every proposed change allows the unit tests to act as an internal compass and an automatic self-documentation mechanism for any modifications to the application throughout its entire lifecycle.

### The Big Picture

Writing unit tests is about much more than obtaining 75% code coverage. We want to write unit tests which define the system's behavior because _unit tests are the low level system specifications_. We should execute all tests as a prerequisite for performing code review to give our team confidence that the proposed change does not impact any historically delivered functionality.

![Path to Repository](/images/pureUnitTesting/pathToRepository.png)

---

## Quality Signals

Like any type of code, some unit tests are written better than others. Some of the quality signals we look for within a unit test are defined below.

### Descriptive Name

Our unit tests should be descriptive and give the reader clear understanding of the intent of the functionality we are testing.

```java
getCalculatedCategoryShouldReturnNorthIfBillingStateIsMinnesota(){...}

insertAccountShouldFailIfTheBillingStateIsNotPopulated(){...}
```

### Single Source of Failure

A unit test should only ever have one reason it can possibly fail. This helps to guarantee the utility of the test as a valid prerequisite for proposed code modifications, and makes your development and delivery process much easier to maintain.

### Repeatable

A unit test should operate the same way every time it is executed. It should not matter if it is executed in a scratch org, sandbox, or production. A test should operate the same way if the test is executed by a system administrator or by a DevOps system user. A test should not depend upon any existing user records or custom metadata within the application. Writing tests which operate in this way help to reduce false negatives of execution failures.

### Cheap

A unit test should be extremely cheap to construct, execute, and report on. Ideally, we want our unit tests to only be a handful of lines, and they should be able to execute extremely fast. We should try to make unit tests which are no more than 45 lines of code and take no more than 100ms to execute.

### Arrange, Act, Assert

In general, we want to follow the _Arrange, Act, Assert_ protocol for defining the structure of our tests.

**Arrange:** The first part of our test should construct objects in memory to set up our test.

**Act:** Next, we should make the explicit method call we are trying to test.

**Assert:** Finally, we should make assertions about the state of the system after the method call is completed.

### Example

Here is an example of a unit test which passes all of our defined quality signals.

```java
@IsTest
private class UselessStringBuilderTest {

  @IsTest
  private static void buildShouldConcatenateTwoStringsTogether() {
    UselessStringBuilder myStringBuilder = new UselessStringBuilder(); // arrange

    String response = myStringBuilder.build('Hello ', 'World!'); // act

    System.assertEquals(
      'Hello World!',
      response,
      'Failed to properly concatenate two strings.'
    ); // assert
  }
}
```

---

## Issues with the Status Quo

### Types of Tests

There are three types of tests we can define in any software project.

A _unit test_ is meant to test the interfaces and public method signatures within our application. Because each unit test is so small in scope, we want to keep their execution time as low as possible and construct many of them.

An _integration test_ is meant to test the behavior of the application when many parts of the system are interacting with each other, such as the server logic and the database. Integration tests take longer to execute and they can break in more ways than a unit test, so we want to construct and own less of them than unit tests.

An _end-to-end_ test is meant to emulate end user behavior by launching a headless browser and emulating the click paths of users. End-to-end tests take much more time to construct, operate, and maintain than the other kinds of tests so we want to construct even less of them.

![](/images/pureUnitTesting/testPyramid.png)

### Explicit Database Dependency

As Salesforce Developers, we sometimes forget how to classify our interactions with the database when writing tests. Apex treats the Database as a first class citizen within the language; we can execute a query and directly assign the query results to objects in memory. This is _not_ the norm for any other technology stack.

![](/images/pureUnitTesting/apexDatabase.png)

Because of this uniquely low friction when interacting with the database, we as Salesforce developers often accidentally define _integration_ tests instead of pure unit tests. Consider the following code block:

```java
Long before = System.currentTimeMillis();
insert new Case(Origin='Email', Status= 'New');
Long after = System.currentTimeMillis();
System.debug('MS to insert a Case : ' + (after - before));
```

This code takes **197ms** to execute when executed in an empty scratch org - which doesn't sound that long in human terms, but we need to realize that this is _as fast as it ever will be_.

As the application grows and more automation gets introduced to the Case sObject - that insertion time is going to continue to increase, so any tests which insert a Case record will execute slower and slower as time goes on. Eventually, running the whole collection of tests for your application will become a burden for your developers, and it will inhibit your team's ability to deliver code on a timely basis.

---

## Unit Test Performance

### Fake Record Ids

Generally speaking, when writing a pure unit test, we want to avoid performing any DML operations. One trick to help us avoid such database interactions is to use fake record Ids. Consider the following code:

```java
Long before = System.currentTimeMillis();
Case myCase = new Case(
 Id = TestUtility.getFakeId(Case.SObjectType),
 Origin = 'Email',
 Status = 'New'
);
Long after = System.currentTimeMillis();
System.debug('MS to construct a Case : ' + (after - before));
```

This code takes **2**ms to execute - no matter if in an empty
scratch org, or a sandbox, or in a production org that has been
running for 9 years. We construct the fake record Id by using
a `TestUtility` class.

```java
@IsTest
public class TestUtility {
  static Integer myNumber = 1;

  public static Id getFakeId(Schema.SObjectType sObjectType) {
    String index = String.valueOf(myNumber++);
    return (Id) (sObjectType.getDescribe().getKeyPrefix() +
    '0'.repeat(12 - index.length()) +
    index);
  }
}
```

Constructing fake record Ids will help us to decrease the necessity of inserting a record just to prepare it for use within a unit test, but sometimes we have more complicated interactions with the database than just obtaining a record Id.

### Virtual Selectors

When our application logic needs to query for records within the database, we often insert the records as part of the setup for the test. Take the following class diagram as an example:

![AccountController and Test](/images/pureUnitTesting/accountControllerAndTest.png)

We can imagine that the `getAccountFromId` method must execute a query to fetch the account from the given record Id. In order to write a test for this class, we would traditionally write something like this where we insert the account, then pass its Id into the method call.

```java
@IsTest
static void getCalculatedCategoryShouldReturnNorthIfBillingStateIsMinnesota() {
  AccountController controllerUnderTest = new AccountController();
  Account testAccount = new Account(
    Name = 'Acme',
    BillingState = 'Minnesota',
  );
  insert testAccount;

  String category = controllerUnderTest.getCalculatedCategory(testAccount.Id);

  System.assertEquals(
    'North',
    category
    'The Category is supposed to be north if the Account BillingState is Minnesota'
  );
}
```

This would work, and it's a fine quality test, but it is _not a pure unit test_ - this is an _integration test_. When we insert the `testAccount`, we are interacting with the database and introducing multiple places where this test can fail that have nothing to do with the calculation of the category. To remedy this situation, we will introduce a new _private virtual inner_ `Selector` class within the `AccountController` class.

![Virtual Selector and Test](/images/pureUnitTesting/virtualSelectorAndTest.png)

Then we can use a `FakeSelector` class within our test that extends the `AccountController.Selector` class and overrides its `getAccountFromId` method, and inject the query results at construction.

```java
public class AccountController {
  @TestVisible AccountController.Selector selector = new AccountController.Selector();

  private Account getAccountFromId(Id recordId) {
    List<Account> accounts = selector.getAccountFromId(recordId);
    if (accounts.isEmpty()) {
      throw new AuraHandledException(Constants.INVALID_RECORD_ID + recordId);
    }
    return accounts[0];
  }

  @TestVisible
  private virtual with sharing class Selector {
    public virtual List<Account> getAccountFromId(Id accountId) {
      return [
        SELECT Id, BillingState, ParentId, Parent.BillingState
        FROM Account
        WHERE Id = :accountId
      ];
    }
  }
}
```

```java
@IsTest
private class AccountControllerTest {
  @IsTest
  static void getCalculatedCategoryShouldReturnNorthIfBillingStateIsMinnesota() {
    AccountController controllerUnderTest = new AccountController();
    Account mockedAccount = new Account(
      Id = TestUtility.getFakeId(Account.SObjectType),
      Name = 'Acme',
      BillingState = 'Minnesota'
    );
    controllerUnderTest.selector = new FakeSelector(
      new List<Account>{ mockedAccount }
    );

    String category = controllerUnderTest.getCalculatedCategory(
      mockedAccount.Id
    );

    System.assertEquals(
      'North',
      category,
      'The Category is supposed to be north if the Account BillingState is Minnesota'
    );
  }

  private inherited sharing class FakeSelector extends AccountController.Selector {
    private List<Account> theAccounts;
    public FakeSelector(List<Account> theAccounts) {
      this.theAccounts = theAccounts;
    }
    public override List<Account> getAccountFromId(Id accountId) {
      return this.theAccounts;
    }
  }
}
```

This test will operate in a constant amount of time, no matter how much automation is introduced on the `Account` sObject. This test can only fail if the `getCalculatedCategory` method is written incorrectly. This test is also is an example of [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection) - the `AccountController` depends upon the existence of an Account record object. Traditionally, we would fetch that record from the database. Here, we are constructing our class in such a way where we can _inject_ the account query results in at construction of the `FakeSelector`. This technique is _extremely_ useful to help improve the testability of your code. As your team matures and you start discussing Test Driven Development and execution of all tests as a prerequisite for code review, techniques like this will become of greater importance during the application lifecycle development process.

---

## Challenge: Test _Everything_ Without DML

My challenge to you is to try to test all of your logic without executing any DML operations. So far, we have discussed using Fake record Ids, and Virtual Selector classes to help isolate our application functionality from the database and allow us to author more pure unit tests. These are helpful, but they are not a complete set of tools for all scenarios - there are some opportunities for improvement.

Try to write a `TestUtility` method which will allow you to set read only fields on sObjects in memory during test execution. This will help you to test any logic which relies on a formula, or rollup summary field value without interacting with the database.

Next, write a `TestUtility` method which will allow you to set parent-child relationships in memory. This will allow you to test application logic which relies on the execution of a subquery without needing to insert any records in the first place.

Finally, similar to how we used dependency injection to test the `AccountController` using the `Selector`, try to write a `DML` class which will allow you to mock the `Database.SaveResult`, `Database.DeleteResult`, etc. objects which are returned to you at runtime during a test.

![Challenge](/images/pureUnitTesting/challenge.png)

Employing all of these techniques will help you to write pure unit tests which operate in a constant amount of time and can be executed as a prerequisite for code review. This will help your team build a suite of tests that can help improve your confidence that your application is still operating in the way you intend.
