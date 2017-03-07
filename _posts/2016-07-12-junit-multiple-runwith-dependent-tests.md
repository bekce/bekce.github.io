---
layout:     post
title:      "JUnit: Multiple Runners and Dependent Tests"
date:       2016-07-12 15:12:00
categories: testing
---

There are times that you may want to run a test case with multiple `@RunWith` annotations or you may have a bootstrap test class that you must run before every other test, such as bootstrapping the application during integration testing. JUnit by its nature does not have test class chaining but there is an easy solution.

In this example I will mention `SpringJUnit4ClassRunner` which is used to bootstrap a Spring context, which is handy for integration testing, but this method applies to every case for JUnit, not just Spring. We also use [`CasperRunner`](https://github.com/bekce/junit-casperjs/blob/master/src/main/java/com/github/raonifn/casperjs/junit/CasperRunner.java) to run [CasperJS](https://jbt.github.io/markdown-editor/casperjs.github.io) tests. However, CasperJS tests must run **after** the context has been initialized, and this order is not deterministic in JUnit.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = MyApplication.class)
@WebIntegrationTest
public class MainTest {

  @Test
  public void subRunner() throws Exception {
    System.out.println("MainTest subRunner()");
    JUnitCore.runClasses(SubTestWithRunner.class);
  }

  @RunWith(CasperRunner.class)
  public static class SubTestWithRunner {
    @BeforeClass
    public static void init() throws Exception {
      System.out.println("SubTestWithRunner init()");
    }
  }
}
```

`MainTest` will be runned by JUnit. `SubTestWithRunner` depends on it (by logic) and it will be runned _inside_ of the main in a `@Test` method, during the call `JUnitCore.runClasses(SubTestWithRunner.class)`. This method ensures the main runner is started correctly before sub runner runs, effectively implementing multiple nested `@RunWith` annotations with dependent test classes.
