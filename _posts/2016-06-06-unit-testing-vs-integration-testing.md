---
layout:     post
title:      Unit Testing vs Integration Testing
date:       2016-06-08 11:21:29
categories: testing
---

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

There are many articles to read on testing but they are often very long and hard to track. I hope this information is useful to some people. Please comment below if you think this can be improved!
