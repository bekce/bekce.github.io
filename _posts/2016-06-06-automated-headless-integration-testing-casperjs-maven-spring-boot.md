---
layout:     post
title:      Automated Headless UI and Integration Testing with CasperJS, Maven and Spring Boot
date:       2017-03-06 18:22:00
categories: testing
redirect_from: "/unit-testing-vs-integration-testing"
---

Automated Headless UI testing for a Java application is often painful to setup and perform correctly. It needs to be integrated in the build cycle, there may be multiple runtime dependencies and platform specific configuration, etc. In this post, I explain how we implemented platform-independent automated UI testing with CasperJS, Maven and Spring Boot.

[CasperJS](http://casperjs.org/) is a library for test automation that works with [PhantomJS](http://phantomjs.org/), which is a headless WebKit browser. They are pretty popular in JS world for automated testing and we aim to use them in our build. CasperJS is an alternative to Selenium, which is a Java based UI testing framework.

[Maven](https://maven.apache.org/) is a widely used build tool mainly for Java. Our method will integrate with specific phases to execute automatic tests.

[Spring Boot](http://projects.spring.io/spring-boot/) is a prominent framework for developing web applications and it includes its own embedded application server so you don't have to deal with them.

## Maven discussion

Despite seeming easy, doing full UI testing in `test` phase is not a good practice as your app probably needs some db connectivity and some other specific methods to run properly. Which means your unit tests will fail before you even package your app. This practice often leads to skipping tests in every Maven build so you're left with _effectively useless_ test code base.

Maven has a `verify` goal designed to perform integration testing. It also has `pre-integration-test` and `post-integration-test` phases to do some useful stuff (such as starting and stopping your server).

We aimed UI testing must be configured in platform-independent way so that one must install absolutely nothing on a Linux, Windows or OS X machine other than Maven to run full UI integration testing just with `mvn verify`.

## Unit Testing vs Integration Testing

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

## Maven Configuration

Both PhantomJS and CasperJS are native executables so they come in different versions for each platform. Following maven plugins download the platform specific executables automatically. If you have a multi-module maven project, put these plugins in the project that you want to run automated tests on (the one with the main application context - `@SpringBootApplication`).

To run the CasperJS from Java we will need `casperjs-junit` library which is basically a bridge between JUnit and CasperJS. However, it assumes CasperJS is already installed on the machine, so it is not completely useful for automating platform-agnostic tests.

```xml
<dependency>
  <groupId>org.webjars.bower</groupId>
  <artifactId>casperjs</artifactId>
  <version>1.1.1</version>
  <scope>test</scope>
</dependency>

<!-- Clone and install https://github.com/bekce/junit-casperjs for this dependency -->
<dependency>
  <groupId>com.github.raonifn</groupId>
  <artifactId>casperjs-junit</artifactId>
  <version>0.4.1-SNAPSHOT</version>
  <scope>test</scope>
</dependency>
```

Now we will configure the main plugins to download and install PhantomJS, download and unpack CasperJS, set necessary flags and pass the executable path to JUnit side.

```xml
<build>
  <plugins>

    <!-- Downloads PhantomJS to appropriate location on this computer -->
    <plugin>
      <groupId>com.github.klieber</groupId>
      <artifactId>phantomjs-maven-plugin</artifactId>
      <version>0.7</version>
      <executions>
        <execution>
          <goals>
            <goal>install</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <version>1.9.7</version>
      </configuration>
    </plugin>

    <!-- Unpacks CasperJS under build directory -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-dependency-plugin</artifactId>
      <version>2.10</version>
      <executions>
        <execution>
          <id>unpack-casperjs</id>
          <phase>pre-integration-test</phase>
          <goals>
            <goal>unpack</goal>
          </goals>
          <configuration>
            <artifactItems>
              <artifactItem>
                <groupId>org.webjars.bower</groupId>
                <artifactId>casperjs</artifactId>
                <version>1.1.1</version>
                <type>jar</type>
                <overWrite>true</overWrite>
                <outputDirectory>${project.build.directory}/casperjs</outputDirectory>
              </artifactItem>
            </artifactItems>
            <outputDirectory>${project.build.directory}/wars</outputDirectory>
            <overWriteReleases>false</overWriteReleases>
            <overWriteSnapshots>true</overWriteSnapshots>
          </configuration>
        </execution>
      </executions>
    </plugin>

    <!-- Passes executable paths to JUnit (CasperRunner) -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <version>2.15</version>
      <configuration>
        <!-- Sets the VM argument line used when integration tests are run. -->
        <argLine>${failsafeArgLine}</argLine>
        <systemPropertyVariables>
          <phantomjs.binary>${phantomjs.binary}</phantomjs.binary>
          <phantomjs.executable>${phantomjs.binary}</phantomjs.executable>
          <casperjs.executable>${project.build.directory}/casperjs/META-INF/resources/webjars/casperjs/1.1.1/bin/casperjs${casperjs.extension}</casperjs.executable>
        </systemPropertyVariables>
      </configuration>
    </plugin>

    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>2.19</version>
      <configuration>
        <argLine>${surefireArgLine}</argLine>
      </configuration>
    </plugin>

    <!-- ... -->
  </plugins>
</build>
```

For each platform, CasperJS runnable comes with different extensions. Therefore we need one more step to correctly resolve the `casperjs.executable` with `casperjs.extension` property, which is platform-dependent. To fix this issue we add a new profile to our `pom.xml`.

```xml
<!-- checks OS and sets casperJS executable extension -->
<profiles>
  <profile>
    <id>Windows</id>
    <activation>
      <os>
        <family>Windows</family>
      </os>
    </activation>
    <properties>
      <casperjs.extension>.exe</casperjs.extension>
    </properties>
  </profile>
  <profile>
    <id>unix</id>
    <activation>
      <os>
        <family>unix</family>
      </os>
    </activation>
    <properties>
      <casperjs.extension></casperjs.extension>
    </properties>
    <build>
      <plugins>
        <plugin>
          <groupId>org.codehaus.mojo</groupId>
          <artifactId>exec-maven-plugin</artifactId>
          <version>1.2</version>
          <executions>
            <execution>
              <phase>pre-integration-test</phase>
              <goals>
                <goal>exec</goal>
              </goals>
            </execution>
          </executions>
          <configuration>
            <executable>chmod</executable>
            <arguments>
              <argument>+x</argument>
              <argument>${project.build.directory}/casperjs/META-INF/resources/webjars/casperjs/1.1.1/bin/casperjs</argument>
            </arguments>
          </configuration>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

Note that for a `Linux` system, we also need to mark the runnable as executable with `chmod +x` command.

## JaCoCo plugin for coverage

After your tests are run, you will probably want to measure test coverage including coverage from the CasperJS tests and report it to a SonarQube server or something similar. For that we need to configure the `jacoco-maven-plugin` with proper configuration. Note that there will be two report files: `jacoco-ut.exec` for unit tests and `jacoco-it.exec` for integration tests. SonarQube can understand those reports as seperate and show Unit Coverage, Integration Coverage and Combined Coverage metrics, which is cool.

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.7.7.201606060606</version>
  <executions>
    <execution>
      <id>pre-unit-test</id>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
      <configuration>
        <destFile>${project.build.directory}/coverage-reports/jacoco-ut.exec</destFile>
        <propertyName>surefireArgLine</propertyName>
      </configuration>
    </execution>
    <execution>
      <id>post-unit-test</id>
      <phase>test</phase>
      <goals>
        <goal>report</goal>
      </goals>
      <configuration>
        <dataFile>${project.build.directory}/coverage-reports/jacoco-ut.exec</dataFile>
        <outputDirectory>${project.reporting.outputDirectory}/jacoco-ut</outputDirectory>
      </configuration>
    </execution>
    <execution>
      <id>pre-integration-test</id>
      <phase>pre-integration-test</phase>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
      <configuration>
        <destFile>${project.build.directory}/coverage-reports/jacoco-it.exec</destFile>
        <propertyName>failsafeArgLine</propertyName>
      </configuration>
    </execution>
    <execution>
      <id>post-integration-test</id>
      <phase>post-integration-test</phase>
      <goals>
        <goal>report</goal>
      </goals>
      <configuration>
        <dataFile>${project.build.directory}/coverage-reports/jacoco-it.exec</dataFile>
        <outputDirectory>${project.reporting.outputDirectory}/jacoco-it</outputDirectory>
      </configuration>
    </execution>
  </executions>
</plugin>
```

## CasperJS script

Put this simple script in `src/test/resources/casperjs/demo.casper.js` file. It automatically runs all classpath files matching `/casperjs/*.casper.js`. You can learn more about how to write scripts [here](https://github.com/casperjs/casperjs) and mostly [here](http://docs.casperjs.org/en/latest/testing.html).

```js
var sys = require('system');
var url = sys.env.START_URL;
var siteName = 'Demo';
var failures = [];

if (!url) {
  this.casper.die('START_URL not found');
}

casper.test.begin('Demo CasperIT test', function suite(test) {
  casper.start();

  //basic tests to see if we can reach the site etc.
  casper.thenOpen(url + "/", function(response) {
    console.log("Trying to open url: " + url);
  }).then( function() {
    test.assertHttpStatus(200, siteName + ' is up');
    test.assertTitle('Automated CasperJS Demo', siteName + ' has the correct title');
  });

  casper.run(function() {
    test.done();
  });

});
```

## Java Classes

For integration testing, we need classes named as `*IT`. In `DemoIT` class below, we're instructing Spring to run on a random port via `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` annotation. After initialization is complete, `@LocalServerPort` retrieves the random port number so we pass it down as an argument to the CasperJS script (`demo.casper.js`). `CasperIT` class waits the application context to start via the `CountDownLatch`. When the initialization is complete, `@Test` method releases the latch and instructs JUnit to run `CasperIT` class.

```java
@RunWith(SpringRunner.class)
@ActiveProfiles("test")
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class DemoIT {

  @LocalServerPort
  private int serverPort;

  @Test
  public void casperJS() throws Exception {
    CasperIT.serverPort = this.serverPort;
    CasperIT.countDownLatch();
    // Below part makes sure the errors in child runner is propagated correctly to this runner
    JUnitCore jUnitCore = new JUnitCore();
    jUnitCore.addListener(new RunListener(){
      @Override
      public void testFailure(Failure failure) throws Exception {
        Assert.fail(failure.getMessage());
      }
    });
    jUnitCore.run(CasperIT.class);
  }

  @RunWith(CasperRunner.class)
  public static class CasperIT {

    private static final CountDownLatch latch = new CountDownLatch(1);
    private static int serverPort;

    /**
    * The latch must be counted down after server init.
    */
    public static void countDownLatch() {
      latch.countDown();
    }

    /**
    * Ensures spring context is initialized before this class.
    */
    @BeforeClass
    public static void beforeClass() throws Exception {
      latch.await(5, TimeUnit.MINUTES);
    }

    @CasperEnvironment
    public Map<String, String> env() {
      Map<String, String> map = new HashMap<>();
      map.put("START_URL", "http://localhost:"+serverPort);
      return map;
    }
  }
}

```

## Using a seperate test database

We should not do integration testing with production database. In this example I use `spring.data.mongodb.uri=mongodb://localhost/demo-test` property to target a different database.

For testing, we will possibly need some demo data, we can easily insert some data during testing, using a class like below. Note that we put this class in test classpath (`src/test/java`) so that it will not be available in regular application runs.

```java
@Component
public class TestLifecycleBean {

  private static final Logger logger = LoggerFactory.getLogger(TestLifecycleBean.class);

  @Autowired
  private MongoTemplate mongoTemplate;
  @Autowired
  private UserRepository userRepository;

  @EventListener
  public void handleContextRefresh(ContextRefreshedEvent event) {
    logger.info("Importing test data...");
    userRepository.save(new User("John", "Doe", "+901112223344"));
    userRepository.save(new User("Jane", "Doe", "+909116667788"));
  }

  @EventListener
  public void handleContextClose(ContextClosedEvent event) {
    logger.info("Dropping test database...");
    if ("demo-test".equals(mongoTemplate.getDb().getName())) {
      mongoTemplate.getDb().dropDatabase();
      logger.info("Dropped database: demo-test");
    }
  }
}
```

## Running

It should be all set! Just run `mvn verify`!

Note: `verify` stage is typically used to run integration tests. It also runs before `install`, so you can also use that command.

## Complete example

I've also prepared a [complete ready-to-run example project on Github](https://github.com/bekce/automated-casperjs-it-demo). Please give it a star if you like it!

Good luck!
