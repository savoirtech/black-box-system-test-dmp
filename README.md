# SIT - Getting Started with the Docker Maven Plugin

## Overview

Below:

- TL;DR = Quick Start with major components and boilerplate snippets

- After that is more depth.

- Also, see the source code under the project folder for a minimal,
  fully-functional example.

**NOTE** this project does not include any containers with System
dependencies (e.g. DB, JMS, etc) for the application.

## TL;DR

<figure>
<img src="./assets/3rdparty/pexels-kelly-2519374.jpg"
alt="pexels kelly 2519374" />
</figure>

Below are templates with most of the boilerplate to get started:

1.  Try the Example

2.  Create Multi-Module Project

3.  Write the Application

4.  Configure the Containers with the Docker-Maven Plugin

5.  Write the Integration Tests

6.  Configure the Failsafe Plugin

7.  Profit

### Try the Example

``` bash
$ cd project
$ mvn clean install
```

### Create Multi-Module Project

- main - application code

- docker-it - tests and maven build instructions to run the docker
  containers

### Write the Application

**ProjectMain.java**

``` java
@SpringBootApplication
public class ProjectMain {

    public static void main(String[] args) {
        SpringApplication.run(ProjectMain.class, args);
    }
}
```

**ProjectRestResource.java**

``` java
@Component
@Path("/api")
public class ProjectRestResource {

    @GET
    @Path("/hi")
    public String helloEndpoint() {
        return "Hello";
    }
}
```

**JerseyWiring.java**

``` java
@Configuration
public class JerseyWiring extends ResourceConfig {
    public JerseyWiring(@Autowired ProjectRestResource projectRestResource) {
        this.registerInstances(projectRestResource);
    }
}
```

### Configure the Containers with the Docker-Maven Plugin

**NOTE** the image is exluded here due to its size and its inclusion
later in the article; see the project sources, or the detail section
[Creation of a Test Container for the
Application](#_creation_of_a_test_container_for_the_application) below,
for the example image boilerplate.

``` xml
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.43.4</version>

    <showLogs>true</showLogs>
    <autoCreateCustomNetworks>true</autoCreateCustomNetworks>

    <executions>
        <execution>
            <id>start-before-integration-test</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>build</goal>
                <goal>start</goal>
            </goals>
        </execution>
        <execution>
            <id>stop-after-integration-test</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>stop</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <images>
            ...
        </images>
    </configuration>
</plugin>
```

### Write the Integration Tests

``` java
public class HelloIT {

    @Test
    public void testHelloEndpoint() {
        ...
    }
}
```

### Configure the Failsafe Plugin

``` xml
<!--                -->
<!-- TEST EXECUTION -->
<!--                -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.0.0-M5</version>
    <executions>
        <execution>
            <id>projname-integration-test</id>
            <goals>
                <goal>integration-test</goal>
            </goals>
            <phase>integration-test</phase>
            <configuration>
                <excludes>
                    <exclude>none</exclude>
                </excludes>
                <includes>
                    <include>**/*IT.java</include>
                </includes>
            </configuration>
        </execution>

        <!-- Fail the build on IT Failures.  Executed as a separate step so that post-integration-test -->
        <!--  phase executes even after an IT failure.                                                 -->
        <execution>
            <id>projname-verify-it</id>
            <goals>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!--suppress MavenModelInspection -->
        <skipITs>${skipITs}</skipITs>
        <reuseForks>true</reuseForks>
        <useSystemClassLoader>false</useSystemClassLoader>
        <systemProperties>
            <property>
                <name>application.base-url</name>
                <!--suppress MavenModelInspection -->
                <value>http://localhost:${application-http-port}</value>
            </property>
        </systemProperties>
    </configuration>
</plugin>
```

### Profit

``` bash
$ mvn clean install
```

## What’s the Opposite of TL;DR?

<figure>
<img src="./assets/3rdparty/pexels-andrea-piacquadio-3790797.jpg"
alt="pexels andrea piacquadio 3790797" />
</figure>

There is a good amount of boiler-plate to get started with the Docker
Maven Plugin, and some gotchas.

Here we cover the basics, provide an example, and discuss the basics of
the example which can be used as a template for your own projects.

Note that this article specifically covers the use of the
`docker-maven-plugin` and not the use of the alternative with
`Test Containers`. The main advantages of the plugin over test
containers include:

- Maven lifecycle support:

  - the plugin creates and starts up the test containers during maven’s
    `pre-integration-test`
    [phase](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

  - it shuts down and removes the containers during maven’s
    `post-integration-test`
    [phase](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

  - all integration tests execute between those two phases, during the
    `integration-test`
    [phase](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

- Fairly direct translation from the plugin config to Dockerfile /
  Docker Compose

**NOTE** a notable disadvantage of the plugin is a lack of cleanup if
the Maven build itself is interrupted. See the section below on "Cleanup
after Interrupted Tests" for more information.

## Multi-Module Project Structure

While it is possible to use the `docker-maven-plugin` in a single-module
project, doing so makes the POM harder to read, and can complicate the
flow. This author is a huge fan of modularity, within reason, and
strongly advocates for the use of a multi-module project here.

The structure of the example project:

- Parent

  - Main

  - Docker-IT

### Parent POM

Links together the entire project through Maven modules:

``` xml
<modules>
    <module>main</module>
    <module>docker-it</module>
</modules>
```

The parent is also a great place for the following, although it is not
demonstrated in this example project:

- Defining common versions of dependencies to use across the entire
  project

- Defining common versions, and configuration, for plugins used across
  the entire project

- Defining common profiles and/or build properties to support build
  parameters (e.g. skipping tests)

### Main JAR

The entire demo application is contained in this one JAR file using the
spring-boot-maven-plugin to generate an "executable jar" file.

Included in this example project is a web service with a simple endpoint
that returns a fixed response with the text `Hello` served at the path
`/api/hi`.

To view the code, please see the TL;DR section above for a quick
overview, or view the full code itself under the project folder. Here
are links to the full code of the Main module:

- [pom.xml](https://github.com/savoirtech/black-box-system-test-dmp/blob/main/project/pom.xml)

- [ProjectMain.java](https://github.com/savoirtech/black-box-system-test-dmp/blob/main/project/main/src/main/java/com/savoirtech/blog/systest/ProjectMain.java)

- [ProjectRestResource.java](https://github.com/savoirtech/black-box-system-test-dmp/blob/main/project/main/src/main/java/com/savoirtech/blog/systest/rest/ProjectRestResource.java)

- [JerseyWiring.java](https://github.com/savoirtech/black-box-system-test-dmp/blob/main/project/main/src/main/java/com/savoirtech/blog/systest/rest/JerseyWiring.java)

#### Generating the Executable JAR

Below is the section of the `pom.xml` in the Main module that packages
the JAR file into an "executable jar".

``` xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>${spring-boot.version}</version>
    <configuration>
        <layout>ZIP</layout>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Note the use of the ZIP layout - this enables deployments to easily add
to the executions class path through environment variables or system
properties. Adding to the classpath enhances support for externalized
configuration. Here’s an example using an environment variable to add to
the class path:

``` bash
$ LOADER_PATH="/opt/application/config"
$ export LOADER_PATH
$ java -jar ...
```

### Docker IT

Here are the key parts for the System Tests:

- creation of a test container for the application

- instructions to spin up and shut down test containers

- test code itself

- handling errors reported by the tests

Please see the TL;DR section above for a quick overview of the code, or
view the full code itself under the project folder. Here are links to
the full code of the Main module:

- [pom.xml](https://github.com/savoirtech/black-box-system-test-dmp/blob/main/project/docker-it/pom.xml)

- [HelloIT.java](https://github.com/savoirtech/black-box-system-test-dmp/blob/main/project/docker-it/src/test/java/com/savoirtech/blog/systest/it/HelloIT.java)

### Creation of a Test Container for the Application

Applications may or may not require a docker container to be created as
a product of the build. This example excludes a separate image as a
product of the build, and instead creates a test-specific image only.

Here is the **from** definition for the container. For this example, a
simple Java-based container is all we need.

``` xml
<build>
    <from>openjdk:17-alpine</from>
```

A subtle, but valuable part of the image creation here uses the
`maven-dependency-plugin` to copy the main JAR into the `docker-it`
project’s target folder. While it is feasible to directly link to the
JAR file in the main sub-module’s folder, there are cases where that is
not the right JAR file to use, or where the JAR file may not be there.
Note that we’re talking about an edge-case here, but it can happen and
when it does, it can create significant confusion due to the wrong
version of the project main JAR running in the test.

Using the maven dependency plugin, we ensure that Maven gives us the
correct version of the JAR file based on the way the developer runs the
build. For example, when running a full build from the parent folder,
maven builds the main JAR, attaches it to the build, and then the
dependency plugin picks up that version of the JAR. On the other hand,
if the developer runs a build from the `docker-it` module folder itself
(not using `-pl` arguments), then maven will use the version of the main
JAR file from the developer’s `~/.m2/repository` cache.

There are more combinations and possible outcomes here. There’s even the
case of testing against a released version of the main JAR file,
downloaded from Maven Central (or other release repository). The safe
advice for new developers is to always run the full build from the
parent folder while developing and testing your local code. Please feel
free to discuss further on the Github discussion pages for this project.

``` xml
<!-- MAVEN DEPENDENCY PLUGIN -->
<!-- Be sure to pick up the correct version of the MAIN jar for this build, whether is comes -->
<!--  from the same build/reactor, maven cache, or other.                                    -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>3.1.2</version>
    <executions>
        <execution>
            <id>copy-main-artifact</id>
            <phase>package</phase>
            <goals>
                <goal>copy</goal>
            </goals>
            <configuration>
                <skip>${skipITs}</skip> <!-- not needed if we are skipping the docker build -->
                <artifactItems>
                    <artifactItem>
                        <groupId>com.savoirtech</groupId>
                        <artifactId>black-box-systest-dmp-main</artifactId>
                        <version>${project.version}</version>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
```

And here we tell the Docker Maven Plugin to add our main project JAR
file into the docker image under the directory `/app`, using the
dependency extracted by the `maven-dependency-plugin`.

``` xml
<assembly>
    <targetDir>/</targetDir>

    <inline xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
            xsi:schemaLocation="
                 http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2
                 http://maven.apache.org/xsd/assembly-1.1.2.xsd
             ">

        <!-- Copy the dependency (main jar) to /app from the target/dependency folder. -->
        <!--  It was placed there by the maven dependency plugin above                 -->
        <fileSets>
            <fileSet>
                <directory>${project.build.directory}/dependency</directory>
                <outputDirectory>/app</outputDirectory>
            </fileSet>
        </fileSets>
    </inline>
</assembly>
```

The `<entryPoint>` tag contains the instructions for starting the
application at container startup time. Note the command is

``` bash
java -Dloader.path=/app/config -jar /app/black-box-systest-dmp-main-${project.version}.jar
```

Note that `${project.version}` will be replaced with the actual project
version by maven at build time.

On a side note - with the entrypoint defined this way in the docker
image, arguments given to a container, when created with this image,
will be passed as arguments to the project’s Main JAR file, and hence
accessible as command-line arguments in `ProjectMain.java`.

``` xml
<entryPoint>
    <exec>
        <arg>java</arg>
        <arg>-Dloader.path=/app/config</arg>
        <arg>-jar</arg>
        <arg>
            /app/black-box-systest-dmp-main-${project.version}.jar
        </arg>
    </exec>
</entryPoint>
```

#### Application Test Image - Putting it All Together

``` xml
<!--                    -->
<!-- APPLICATION IMAGE  -->
<!--                    -->
<image>
    <name>projname-application-it-image</name>
    <alias>application</alias>
    <build>
        <from>openjdk:17-alpine</from>
        <assembly>
            <targetDir>/</targetDir>

            <inline xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                    xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
                    xsi:schemaLocation="
                         http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2
                         http://maven.apache.org/xsd/assembly-1.1.2.xsd
                     ">

                <!-- Copy the dependency (main jar) to /app from the target/dependency folder. -->
                <!--  It was placed there by the maven dependency plugin above                 -->
                <fileSets>
                    <fileSet>
                        <directory>${project.build.directory}/dependency</directory>
                        <outputDirectory>/app</outputDirectory>
                    </fileSet>
                </fileSets>
            </inline>
        </assembly>
        <entryPoint>
            <exec>
                <arg>java</arg>
                <arg>-Dloader.path=/app/config</arg>
                <arg>-jar</arg>
                <arg>
                    /app/black-box-systest-dmp-main-${project.version}.jar
                </arg>
            </exec>
        </entryPoint>
    </build>
    <run>
        <hostname>test-application</hostname>
        <ports>
            <port>application-http-port:8080</port>
            <!--<port>5005:5005</port>-->
        </ports>
        <env>
            <!-- Need to make sure address=* is in the DEBUG OPTS otherwise it listens on the container's localhost only -->
            <JAVA_TOOL_OPTIONS>-Djava.security.egd=file:/dev/./urandom
                -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
            </JAVA_TOOL_OPTIONS>
        </env>
        <wait>
            <log>Started ProjectMain in .* seconds</log>
            <time>60000</time>
        </wait>
        <network>
            <mode>custom</mode>
            <name>projname-docker-it-network</name>
            <alias>minion</alias>
        </network>
    </run>
</image>
```

#### HelloIT Test

Placing the test file, `HelloIT`, under the project’s `src/main/test`
folder, the Failsafe plugin matches the file based on the naming pattern
`**/*IT.java` and executes the test code within.

All methods marked with the `@Test` annotation are executed by Failsafe.

Note the use of RestAssured for a fairly easy way to build the client
call to our service.

``` java
@Test
public void testHelloEndpoint() {
    String baseUrl = System.getProperty("application.base-url");
    String url = baseUrl + "/api/hi";

    RestAssuredConfig config =
            RestAssuredConfig.config()
                    .httpClient(
                            HttpClientConfig.httpClientConfig()
                                    .setParam("http.connection.timeout", DEFAULT_SOCKET_TIMEOUT)
                                    .setParam("http.socket.timeout", DEFAULT_SOCKET_TIMEOUT)
                    );

    Response response =
            RestAssured.given()
                    .config(config)
                    .get(url)
                    .thenReturn();

    assertEquals(200, response.getStatusCode());
    assertEquals("Hello", response.getBody().asString());
}
```

Note the use of `System.getProperty("application.base-url");` in the
test method. That property is injected by the Failsafe plugin using the
following section under the plugin’s `<configuration>` tag:

``` xml
<systemProperties>
    <property>
        <name>application.base-url</name>
        <!--suppress MavenModelInspection -->
        <value>http://localhost:${application-http-port}</value>
    </property>
</systemProperties>
```

Notice the following line:

``` xml
<value>http://localhost:${application-http-port}</value>
```

That ties in the dynamically-allocated, ephemeral, port from docker,
which is significant for minimizing environment-specific failures, such
as attempting to build on a machine where the developer has other
applications listening on the same ports.

``` xml
<ports>
    <port>application-http-port:8080</port>
    <!--<port>5005:5005</port>-->
</ports>
```

Notice the left side of the `:` in the port declaration is
`application-http-port`, which informs the docker-maven-plugin to
dynamically allocate a host port and then populate a Maven property of
the same name with that port number for its value. The number on the
right side of the `:` is the container’s internal port number - 8080 in
this case, the port number on which our application is listening for web
requests.

The commented-out `5005:5005` port mapping is a convenience to simplify
the effort needed to debug the application while it is running inside
the container. Uncommenting this line grants access to the applications
debug port inside the container from the host port 5005. Note that the
image is also created with `JAVA_TOOL_OPTIONS` configured with debugging
enabled for the application.

#### Handling Errors in the Test

Once integration tests have completed, it is important to inform Maven
of failures. This is not done automatically, so we need to configure the
plugin to do so. Here is the boilerplate that goes in the `executions`
section of the `maven-failsafe-plugin`.

``` xml
<!-- Fail the build on IT Failures.  Executed as a separate step so that post-integration-test -->
<!--  phase executes even after an IT failure.                                                 -->
<execution>
    <id>projname-verify-it</id>
    <goals>
        <goal>verify</goal>
    </goals>
</execution>
```

Without this, after tests fail, Maven will continue the build and ignore
the failures. Adding this section, Maven fails the build when there are
test failures (ignoring possible maven options to explicitly ignore
failures).

## Cleanup after Interrupted Tests

Imagine this - you’re running the integration tests and notice a
failure, or remember that you forgot to finish that one thing that will
make it all fail. So you decide not to wait for the tests to complete.

So you hit Ctrl-C and stop the process during the integration-test
phase. Here’s what you are left with:

- All of the containers successfully started by the
  `docker-maven-plugin` during the `pre-integration-test` phase continue
  to run

- The custom network used for the test remains

- Temporary volumes created by the plugin remain

Here are docker instructions that you can use to help with cleanup:

``` bash
$ docker ps
$ docker stop a0b1c2d3
$ docker container rm a0b1c2d3
$ docker network ls
$ docker network rm projname-docker-it-network
$ docker volume ls
$ docker volume rm aa00bb11cc22...
```

## Try it Out

``` bash
$ git clone https://github.com/savoirtech/black-box-system-test-dmp.git
$ cd black-box-system-test-dmp
$ mvn -f project clean install
```

## Author’s Notes

- Initially, the plan was to include 1 external system dependency - such
  as a DB or JMS system - but due to the length of the article, it was
  removed.

- JAX-RS was used in this example

  - The main motivation is to avoid "vendor lock-in", and use the
    wider-reaching standard

  - Spring’s `@RestController` could just as easily have been used.

  - There is additional boilerplate wiring for JAX-RS

- Always using `clean` in `mvn clean install`

  - Maven will attempt incremental builds to save time

  - There are many cases Maven does not catch, which leads to the
    incremental build being incorrect

  - The amount of confusion and time to resolve these cases just isn’t
    worth the savings **most** of the time

  - If there are very slow parts of the build, such as the integration
    tests described in this article, skipping them while iterating over
    the code, and then making sure to run them later - before commit and
    push - is effective.

  - The `install` target is ideal in most cases because it makes sure
    that working with a subset of the build - such as running maven out
    of a sub-module’s folder - gives the developer the expected, latest
    version of built in-project dependencies.

  - Feel free to use different targets as they suit your needs - just be
    aware of the potential pitfalls.

# About the Authors

[Arthur
Naseef](https://github.com/savoirtech/blogs/blob/main/authors/ArthurNaseef.md)

## Reaching Out

Please do not hesitate to reach out with questions and comments, here on
the Blog, or through the Savoir Technologies website at
<https://www.savoirtech.com>.

Also note the discussion section on Github for the project is probably
the best place to discuss this article since it is shared with everyone
and makes the entire conversation easy to find and follow.

# With Thanks

Thank you to the following individuals for the stock photos!

- Kelly for
  <https://www.pexels.com/photo/motor-bike-running-close-up-photography-2519374>

- Andrea Piacquadio for
  <https://www.pexels.com/photo/woman-in-white-dress-shirt-using-white-laptop-computer-3790797>

\(c\) 2023 Savoir Technologies
