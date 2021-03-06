# Enhancing Boot Application with Metrics

## Set up the Actuator

Spring Boot includes a number of additional features to help you monitor and manage your application when it’s pushed to production. These features are added by adding **spring-boot-starter-actuator** to the classpath.  Our initial project setup already included it as a dependency.

* Verify the Spring Boot Actuator dependency in the following file: **cloud-native-spring/pom.xml**. you will need to add this.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
By default Spring Boot does not expose these management endpoints (which is a good thing!).  Though you wouldn't want to expose all of them in production, we'll do so in this sample app to make demonstration a bit easier and simpler.  

* Add the following properties to **cloud-native-spring/src/main/resources/application.yml**.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

* Run the updated application

```bash
mvn clean spring-boot:run
```

* Try out the following endpoints with [Postman](https://www.getpostman.com). The output is omitted here because it can be quite large:

`http :8080/actuator/health`

-> Displays Application and Datasource health information.  This can be customized based on application functionality, which we'll do later.

`http :8080/actuator/beans`

-> Dumps all of the beans in the Spring context.

`http :8080/actuator/autoconfig`

-> Dumps all of the auto-configuration performed as part of application bootstrapping.

`http :8080/actuator/configprops`

-> Displays a collated list of all @ConfigurationProperties.

`http :8080/actuator/env`

-> Dumps the application’s shell environment as well as all Java system properties.

`http :8080/actuator/mappings`

-> Dumps all URI request mappings and the controller methods to which they are mapped.

`http :8080/actuator/threaddump`

-> Performs a thread dump.

`http :8080/actuator/httptrace`

-> Displays trace information (by default the last few HTTP requests).

`http :8080/actuator/flyway`

-> Shows any Flyway database migrations that have been applied.

* Stop the `cloud-native-spring` application with **CTRL-c**

## Include Version Control Info

Spring Boot provides an endpoint (`http://localhost:8080/actuator/info`) that allows the exposure of arbitrary metadata. By default, it is empty.

One thing that _actuator_ does well is expose information about the specific build and version control coordinates for a given deployment.

* Edit the following file: **cloud-native-spring/pom.xml**

```xml
<pluginRepositories>
	<pluginRepository>
	    <id>sonatype-snapshots</id>
		<name>Sonatype Snapshots</name>
        <url>"https://oss.sonatype.org/content/repositories/snapshots/"</url>
	</pluginRepository>
</pluginRepositories>
```

* Then, you must edit the file and add the plugin dependency within <build> block.

```xml
    <build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<executions>
					<execution>
						<goals>
							<goal>build-info</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
			<plugin>
				<groupId>pl.project13.maven</groupId>
				<artifactId>git-commit-id-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
```
* Run the cloud-native-spring application:

```#!/usr/bin/env bash
mvn spring-boot:run
```

* Let's use httpie to verify that Git commit information is now included

```bash
http :8080/actuator/info
```

```json
{
    "git": {
        "commit": {
            "time": "2017-09-07T13:52+0000",
            "id": "3393f74"
        },
        "branch": "master"
    }
}
```

* Stop the **cloud-native-spring_ application**

*What Just Happened?*

By including the _git-commit-id-plugin_ dependency, details about git commit information will be included in the */actuator/info* endpoint. Git information is captured in a _git.properties_ file that is generated with the build. Review the following file: */cloud-native-spring/build/resources/main/git.properties*

## Include Build Info

* Add the following properties to *cloud-native-spring/src/main/resources/application.yml*.

```yaml
info: # add this section
  build:
    artifact: @project.artifactId@
    name: @project.name@
    description: @project.description@
    version: @project.version@
```
Note we're defining token delimited value-placeholders for each property.  In order to have these properties replaced, we'll need to add some further instructions to the _build.gradle_ file.

-> if STS/Eclipse [reports a problem](https://jira.spring.io/browse/STS-4201) with the application.yml due to @ character, the problem can safely be ignored.

* Again we'll use httpie to verify that the Build information is now included

```bash
http :8080/actuator/info
```

```json
{
    "git": {
        "commit": {
            "time": "2018-03-15T08:01:31Z",
            "id": "4d1fe8b"
        },
        "branch": "master"
    },
    "build": {
        "version": "0.0.1-SNAPSHOT",
        "artifact": "cloud-native-spring",
        "name": "cloud-native-spring",
        "group": "io.pivotal",
        "time": "2018-03-15T08:22:48.543Z"
    }
}
```

* Stop the cloud-native-spring application.

*What Just Happened?*

We have mapped build properties into the /actuator/info endpoint.

Read more about exposing data in the /actuator/info endpoint [here](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready)

## Health Indicators

Spring Boot provides an endpoint `http://localhost:8080/actuator/health` that exposes various health indicators that describe the health of the given application.

Normally, the /actuator/health endpoint will only expose an UP or DOWN value.

```json
{
  "status": "UP"
}
```
We want to expose more detail about the health and well-being of the application, so we're going to need a bit more configuration to **cloud-native-spring/src/main/resources/application.yml**, underneath the _management_ prefix, add

```yaml
  endpoint:
    health:
      show-details: always
```
* Run the cloud-native-spring application:

```bash
mvn spring-boot:run
```

* Use httpie to verify the output of the health endpoint

```bash
http :8080/actuator/health
```

Out of the box is a _DiskSpaceHealthIndicator_ that monitors health in terms of available disk space. Would your Ops team like to know if the app is close to running out of disk space? DiskSpaceHealthIndicator can be customized via _DiskSpaceHealthIndicatorProperties_. For instance, setting a different threshold for when to report the status as DOWN.

```json
{
    "status": "UP",
    "details": {
        "diskSpace": {
            "status": "UP",
            "details": {
                "total": 499963170816,
                "free": 375287070720,
                "threshold": 10485760
            }
        },
        "db": {
            "status": "UP",
            "details": {
                "database": "H2",
                "hello": 1
            }
        }
    }
}
```

* Stop the cloud-native-spring application.

* Create the class _io.pivotal.FlappingHealthIndicator_ (**/cloud-native-spring/src/main/java/io/pivotal/FlappingHealthIndicator.java**) and into it paste the following code:

```java
package io.pivotal;

import java.util.Random;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class FlappingHealthIndicator implements HealthIndicator {

    private Random random = new Random(System.currentTimeMillis());

    @Override
    public Health health() {
        int result = random.nextInt(100);
        if (result < 50) {
            return Health.down().withDetail("flapper", "failure").withDetail("random", result).build();
        } else {
            return Health.up().withDetail("flapper", "ok").withDetail("random", result).build();
        }
    }
}
```
This demo health indicator will randomize the health check.

* Build and run the _cloud-native-spring_ application:

```bash
$ mvn clean spring-boot:run
```

* Browse to `http://localhost:8080/actuator/health` and verify that the output is similar to the following (and changes randomly!).

```json
{
    "status": "UP",
    "details": {
        "flapping": {
            "status": "UP",
            "details": {
                "flapper": "ok",
                "random": 63
            }
        },
        "diskSpace": {
            "status": "UP",
            "details": {
                "total": 499963170816,
                "free": 375287070720,
                "threshold": 10485760
            }
        },
        "db": {
            "status": "UP",
            "details": {
                "database": "H2",
                "hello": 1
            }
        }
    }
}
```

## Metrics

Spring Boot provides an endpoint http://localhost:8080/actuator/metrics that exposes several automatically collected metrics for your application. It also allows for the creation of custom metrics.

* Browse to `http://localhost:8080/actuator/metrics`. Review the metrics exposed.

```json
{
    "names": [
        "jvm.memory.max",
        "http.server.requests",
        "jdbc.connections.active",
        "process.files.max",
        "jvm.gc.memory.promoted",
        "tomcat.cache.hit",
        "system.load.average.1m",
        "tomcat.cache.access",
        "jvm.memory.used",
        "jvm.gc.max.data.size",
        "jdbc.connections.max",
        "jdbc.connections.min",
        "jvm.gc.pause",
        "jvm.memory.committed",
        "system.cpu.count",
        "logback.events",
        "tomcat.global.sent",
        "jvm.buffer.memory.used",
        "tomcat.sessions.created",
        "jvm.threads.daemon",
        "system.cpu.usage",
        "jvm.gc.memory.allocated",
        "tomcat.global.request.max",
        "hikaricp.connections.idle",
        "hikaricp.connections.pending",
        "tomcat.global.request",
        "tomcat.sessions.expired",
        "hikaricp.connections",
        "jvm.threads.live",
        "jvm.threads.peak",
        "tomcat.global.received",
        "hikaricp.connections.active",
        "hikaricp.connections.creation",
        "process.uptime",
        "tomcat.sessions.rejected",
        "process.cpu.usage",
        "tomcat.threads.config.max",
        "jvm.classes.loaded",
        "hikaricp.connections.max",
        "hikaricp.connections.min",
        "jvm.classes.unloaded",
        "tomcat.global.error",
        "tomcat.sessions.active.current",
        "tomcat.sessions.alive.max",
        "jvm.gc.live.data.size",
        "tomcat.servlet.request.max",
        "hikaricp.connections.usage",
        "tomcat.threads.current",
        "tomcat.servlet.request",
        "hikaricp.connections.timeout",
        "process.files.open",
        "jvm.buffer.count",
        "jvm.buffer.total.capacity",
        "tomcat.sessions.active.max",
        "hikaricp.connections.acquire",
        "tomcat.threads.busy",
        "process.start.time",
        "tomcat.servlet.error"
    ]
}
```

* Stop the cloud-native-spring application.

## Deploy _cloud-native-spring_ to Pivotal Cloud Foundry

* When running a Spring Boot application on Pivotal Cloud Foundry with the actuator endpoints enabled, you can visualize actuator management information on the Applications Manager app dashboard.  To enable this there are a few properties we need to add.  Add the following to **/cloud-native-spring/src/main/resources/application.yml**:

```yaml
---
spring:
  profiles: cloud

management:
  cloudfoundry:
    enabled: true
    skip-ssl-validation: true
```

* Push application into Cloud Foundry

```bash
  mvn package
  cf push
```

* Find the URL created for your app in the health status report. Browse to your app.  Also view your application details in the Apps Manager UI:

![Appsman](appsman.jpg)

* From this UI you can also dynamically change logging levels:

![Logging](logging.jpg)

**Congratulations!** You’ve just learned how to add health and metrics to any Spring Boot application.
