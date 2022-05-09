+++
author = "Rikito Taniguchi"
description = ""
draft = true
publishDate = ""
tags = []
title = "Tests as a Documentation"

+++
Developers often talk about the software architecture, and how to write clean and readable code. On the other hand, "how unit tests should be written" is less discussed, and many developers don't care about the readability of the test programs.

As a result, tests are added/extended in a "bad" manner (that is usually avoided if it is in non-test programs), making tests hard to maintain, fragile, and (even by future selves) difficult to read, hard to understand what is being verified.

This small article introduces one aspect of a "good" test program. For more information, see books ([XUnit Test Patterns](http://xunitpatterns.com/), etc.? ) or discuss your team members, but I hope this article will raise awareness of how to write good tests.

## Introduction (definition of "good" unit tests)

One of the most important roles of unit testing is to verify that the code (unit) you wrote or modified behaves "as expected". On top of that, I personally believe that one of the most important roles is "Tests as Documentation". In the xUnit Test Patterns, it is called [Communicate Intent](http://xunitpatterns.com/Principles%20of%20Test%20Automation.html#Communicate%20Intent)

Ideally, unit tests are written to verify a program unit (function or class) behaves as expected under a specific condition, given an input, and "good" unit tests (tests that capture the intent) are those that )

* Gives a quick and accurate big picture of the expected behavior of the SUT (system under test).
  * (compared to reading the program line by line or the specification)
    * (IMHO, specifications are usually abandoned (or even do not exists)).
* Easy to maintain
  * As you understand simple, easy-to-understand production code is easy to maintain.

What are the dos and don'ts for writing such good unit tests?

***

## To write a test that tells the intention.

[Tests should be easy to write and maintain - Goals of Test Automation](http://xunitpatterns.com/Goals%20of%20Test%20Automation.html#Tests%20should%20be%20easy%20to%20write%20and%20maintain)

### Simple Test

Tests should be small and satisfy the principle of [Verify One Condition](http://xunitpatterns.com/Principles%20of%20Test%20Automation.html#Verify%20One%20Condition%20per%20Test) per Test. This makes it easier to know what the test is verifying and facilitates [Defect Localisation](http://xunitpatterns.com/Goals%20of%20Test%20Automation.html#Defect%20Localization) when the test fails.

* **It doesn't mean that there should always be one assertion per test, but that one test should verify one "behavior".**
  * If multiple assertions are needed to verify behavior, there is no need to split the test.
* If the correspondence between the conditions and the expected results is clear, then a [Table Driven Test](https://github.com/golang/go/wiki/TableDrivenTests) may help.

### Expressive Tests (TODO)

Use the Test Utility Method to write 

The Test Utility Method, for example, is used to make it easy to understand what is happening and what you are trying to verify by reading the tests. This is a common practice for production code.

One drawback of the Test Utility Method is that the number of APIs that the test reader needs to know increases. Method, it is possible to write tests that convey the meaning easily.