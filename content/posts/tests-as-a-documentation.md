+++
author = "Rikito Taniguchi"
description = ""
draft = true
publishDate = ""
tags = []
title = "Tests as a Documentation"

+++
While the design of production code is often discussed, "how unit tests should be written" is less discussed. I think it is often treated lightly as if it is good if the coverage is high anyway.

As a result, tests are added/extended in a "bad" way when writing tests and **especially when adding tests**, making them hard to maintain, fragile**,** and (even by future selves) difficult to read, hard to understand what is being verified.

For more information, see books ([XUnit Test Patterns](http://xunitpatterns.com/), etc.? ) or discuss it with your team members, but I hope this article will raise awareness of how to write tests.

## Introduction

One of the most important roles of unit testing is to verify that the code (unit) you wrote or modified behaves "as expected". On top of that, I personally believe that one of the most important roles is "Tests as Documentation", which in the xUnit Test Patterns is called [Communicate Intent](http://xunitpatterns.com/Principles%20of%20Test%20Automation.html#Communicate%20Intent)

Ideally, unit tests are written to verify a program unit (function or class) behaves as expected under a specific condition, with an input

 "good" unit tests (tests that capture the intent) are those that ) is

* Gives a quick and accurate picture of the overall expected behavior of the SUT (system under test).
  *  (compared to reading the program line by line or the specification, which is not updated (or even exists)).
* Easy to maintain
  *  Just as simple production code that is easy to understand is easier to maintain than complex code.

What are the do's and don'ts to write such unit tests?

***

## To write a test that conveys the intention of the test.

[Tests should be easy to write and maintain - Goals of Test Automation](http://xunitpatterns.com/Goals%20of%20Test%20Automation.html#Tests%20should%20be%20easy%20to%20write%20and%20maintain)

Tests should be small and satisfy the principle of [Verify One Condition](http://xunitpatterns.com/Principles%20of%20Test%20Automation.html#Verify%20One%20Condition%20per%20Test) per Test. This makes it easier to know what the test is verifying and facilitates [Defect Localisation](http://xunitpatterns.com/Goals%20of%20Test%20Automation.html#Defect%20Localization) when the test fails.

* Doesn't mean that there should always be one assertion per test, but that one test should verify one "behavior".
  * If multiple assertions are needed to verify behavior, there is no need to split the test.
* If the correspondence between the conditions and the expected results is clear, then a Table Driven Test, for example, can be used.