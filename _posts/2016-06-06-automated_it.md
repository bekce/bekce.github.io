---
layout:     post
title:      Automated Headless UI Testing with CasperJS, Maven and Spring Boot
date:       2016-06-08 11:21:29
categories: testing
---

Automated UI testing is often painful to setup and perform correctly. There are different libraries and the application may not be designed in a way to test easily. Moreover, it may be hard to setup fully automated configuration regardless of the build machine.

We aimed UI testing must be configured in platform-independent way so that one must install absolutely nothing on a Linux, Windows or OS X machine other than Maven to run full UI integration testing just with `mvn verify`.

[CasperJS](http://casperjs.org/) is a library for test automation that works with [PhantomJS](http://phantomjs.org/), which is a headless WebKit browser. They are pretty popular in JS world. As in Java, there is Selenium. Our build style covers both CasperJS style tests and Selenium style tests, as we have both.

[Spring Boot](http://projects.spring.io/spring-boot/) is a prominent framework for developing web applications and it includes its own embedded server runner so you don't have to deal with application servers, but it is also not that hard to automate too.

### Maven discussion

Despite seeming easy, doing full UI testing in `test` phase is not a good practice as your app probably needs some db connectivity and some other specific methods to run properly so your unit tests will fail before you even package your app. This practice often leads to skipping tests in every Maven build so you're left with a useless test code.

Maven has a `verify` goal designed to perform integration testing. It also has `pre-integration-test` and `post-integration-test` phases to do some useful stuff (such as starting and stopping your server) which are (AFAIK) otherwise not available in `test` goal.

### Unit Testing vs Integration Testing

In the table below, I have summarized the properties of and differences between Unit Testing and Integration Testing with regards to Maven inferred during my expertises.

| | Unit Testing | Integration Testing |
|-|:-|:-|
| Tests | easily reproducible headless cases: utility functions, partially independent business logic, isolated functionalities with mocking | reproducible realistic cases (including -but not limited to- UI) often depending on other systems with little to no mocking but still isolated from production instances |
| Applicable Project Types | any project, particularly important for library or utility projects | shippable projects, often the last module(s) in multi-module project chains |
| Coverage Goals | increase as much as possible | complete it |
| Maintained by | developer team | test team |
| Runtime Duration | instant to short | medium to long |
| Typical Run Interval | often: after every push to main branch (must be automated with CI) or fixed intervals like 15mins to 1hr (depending on the project) | not-so-often: before releases/after sprints or fixed intervals (like once a day or a week), automated or manual runs |
| Runtime Dependencies | none, must be self-sufficient | database, other services in the network (though it is best if those get automatically configured at runtime) |
| Maven Goal | `test` | `verify` |
| Maven Plugin | [maven-surefire-plugin](http://maven.apache.org/surefire/maven-surefire-plugin/) | [maven-failsafe-plugin](http://maven.apache.org/surefire/maven-failsafe-plugin/) |
| Java Class Naming Convention | Ending with `Test` | Ending with `IT` |

There are many articles to read on testing but they are often very long and hard to track. I hope this information is useful to some people.

### PhantomJS installer

Both PhantomJS and CasperJS are native executables in multiple forms for each platform so we must use a

In this guide I will explain the setup we performed

 (`pre-integration-test` -> `verify` -> `post-integration-test` )

