---
layout: post
title: Optimizing Spring Boot Startup Speed with Class Data Sharing (CDS)
date: 2024-12-05
categories: [ Java, Spring Boot ]
tags: [ java, spring boot, startup time optimization, cds ]     # TAG names should always be lowercase
description: My journey optimizing Spring Boot application startup times using Class Data Sharing (CDS)
toc: true
---

I recently set out to optimize the startup time of our Spring Boot applications running on AWS ECS. While startup speed
might seem trivial for long-running applications, it can make autoscaling more effective and help us unlock cost savings
without sacrificing system resilience. This post shares my journey of using Class Data Sharing (CDS) to
reduce the startup time
of the application.

# What is CDS?

[Class Data Sharing (CDS)](https://docs.oracle.com/en/java/javase/21/vm/class-data-sharing.html) is a JVM feature that
reduces startup time and memory usage by creating a shared archive of loaded classes. Enabled by default for JDK classes
since JDK 12, it can also be extended to application classes.

## Why CDS?

I chose CDS for its simplicity and impact among various optimization techniques:

* First-class support in Spring Framework 6.1 and Spring Boot 3.3
* Fewer constraints compared to Spring AOT, GraalVM and Project CRaC
* Easy integration into existing deployment models.

> [Sébastien Deleuze's talk on Efficient containers with Spring Boot 3, Java 21, and
> CDS](https://youtu.be/H2tM7EClyx8?si=NEI6CP-qDiArSoRJ) is a great resource on startup optimization techniques
{: .prompt-tip }

# Experiment Details

## Deployment

For the experiment, the application is deployed on AWS ECS with the following resources:

* 2 vCpus, 6 GB Memory (5.5 GB allocated to the Java heap)
* Open JDK 21
* Startup time measured over four runs using both G1GC and EpsilonGC to assess garbage collection's impact.

## Application

The test application is a Spring Boot 3.4.0 application (~12k lines of Java and Kotlin) that uses Spring Cloud 2024 and
spring-cloud-config-client for configuration.

**Note**: The application’s source code is proprietary and cannot be shared.

# Initial Setup

> While I would love to use buildpacks to generate our images, some environment constraints require me to use custom
> base images. The learnings around troubleshooting and problematic dependencies are applicable regardless of how the
> image is packaged.
{: .prompt-tip }

My initial Dockerfile was straightforward:

1. Copy the jar into the image.
2. Run it using `java -jar`.

```dockerfile
ARG JAVA_BASE_IMAGE
FROM $JAVA_BASE_IMAGE
ARG JAR_FILE
COPY $JAR_FILE /opt/application/

ENTRYPOINT ["java", "-jar", "/opt/application/$JAR_FILE"]
```

Startup times were:

* With EpsilonGC: 36-40&nbsp;s
* With G1GC: 37-45&nbsp;s

# Exploded Jars

To make the classpath predictable for CDS, the executable JAR was exploded using a multistage Docker build:

```dockerfile
ARG JAVA_BASE_IMAGE
FROM $JAVA_BASE_IMAGE as builder
ARG JAR_FILE
COPY $JAR_FILE /tmp/
RUN java -Djarmode=tools -jar /tmp/$JAR_FILE extract --destination /opt/application


ARG JAVA_BASE_IMAGE
FROM $JAVA_BASE_IMAGE
ARG JARFILE

COPY --link --from=builder /opt/application/lib /opt/application/lib
COPY --link --from=builder /opt/application/$JARFILE /opt/application/

ENTRYPOINT ["java", "-jar", "/opt/application/$JAR_FILE"]
```

This reduced startup time by 7–11&nbsp;s:

* With Epsilon GC: 27–32&nbsp;s
* With G1GC: 30–34&nbsp;s

# First Attempt at CDS

> The classpath must be the same between the CDS training run and real execution. This includes the location of the jars
> on the filesystem!
{: .prompt-tip }

Next, I enabled CDS by adding a training run in the builder stage to create the shared archive:

```dockerfile
ARG JAVA_BASE_IMAGE
FROM $JAVA_BASE_IMAGE as builder
ARG JAR_FILE
COPY $JAR_FILE /tmp/
RUN java -Djarmode=tools -jar /tmp/$JAR_FILE extract --destination /opt/application
RUN java -XX:ArchiveClassesAtExit=/opt/application/application.jsa -Dspring.context.exit=onRefresh -jar /opt/application/$JAR_FILE

ARG JAVA_BASE_IMAGE
FROM $JAVA_BASE_IMAGE
ARG JARFILE

COPY --link --from=builder /opt/application/lib /opt/application/lib
COPY --link --from=builder /opt/application/$JARFILE /opt/application/
COPY --link --from=builder /opt/application/application.jsa /opt/application/

ENTRYPOINT ["java", "-jar", "/opt/application/$JAR_FILE"]
```

Enabling CDS showed no improvement in startup time. It worked in simpler apps, indicating an issue with my setup.

# Troubleshooting CDS

## First Observations

I don't give up after one failure, so it's time to dig deeper. In my docker build log there are no logs from
the Spring application. The log levels are set correctly, so the application might not be starting up properly.

## Load Class List

To understand what occurred during the CDS training run, I added the `-XX:DumpLoadedClassList` argument to generate a
list of loaded classes.

Key observations:

* The only application class included was the Spring Boot application class itself.
* Many Maven dependencies were notably absent.

This indicated issues with class loading during the training phase, limiting CDS effectiveness.

## Classloader Logs

To verify if classes are loaded from the shared archive, we can enable classloading logs during a normal application
run:

```shell
java -XX:SharedArchiveFile=application.jsa  -Xlog:class+load=info:file=class-load.log  -jar extracted/my-application-0.0.1.jar
```

```text
[0.037s][info][class,load] java.lang.Object source: shared objects file
[0.037s][info][class,load] java.io.Serializable source: shared objects file
[0.037s][info][class,load] java.lang.Comparable source: shared objects file

[0.131s][info][class,load] org.springframework.boot.SpringApplication source: file:/dev/my-application/target/extracted/lib/spring-boot-3.4.0.jar
[0.132s][info][class,load] org.springframework.boot.BootstrapRegistry source: file:/dev/my-application/target/extracted/lib/spring-boot-3.4.0.jar
[0.133s][info][class,load] org.springframework.boot.BootstrapContext source: file:/dev/my-application/target/extracted/lib/spring-boot-3.4.0.jar
[0.133s][info][class,load] org.springframework.boot.ConfigurableBootstrapContext source: file:/dev/my-application/target/extracted/lib/spring-boot-3.4.0.jar
```

_Extracts from the classloading log_

Analyzing the classloading logs confirmed only 19% of classes were loaded from the shared archive, with most being
loaded from the filesystem.

## Debugger

Some creative searching led me to the `org.springframework.context.support.DefaultLifecycleProcessor` class that stops
the application when `-Dspring.context.exit=onRefresh` is present.

![DefaultLifecycleProcessor onRefresh method](/assets/img/2024-12-05_spring-boot-cds/DefaultLifecycleProcessor.png)
_DefaultLifecycleProcessor onRefresh method_

Debugging my application, putting a breakpoint in this class, and inspecting the thread stacks leading pointed to two
problematic dependencies.

* Spring Cloud Bootstrap: Bootstrapping caused the application context to refresh before most classes have been loaded.
* Spring Cloud OpenFeign: Each Feign client added a new application context, some of these were refreshed (causing an
  application exit) before most classes had been loaded.

### Spring Cloud Bootstrap

The first time the `onRefresh` method was called, there were several Spring Cloud bootstrap classes present in the
thread stack:

![Debugger Thread Stack](/assets/img/2024-12-05_spring-boot-cds/SpringCloudBootstrapStack.png)
_Spring Bootstrap classes in the thread stack_

**Fix**: I removed the `spring-cloud-starter-bootstrap` dependency and replaced the bootstrap config with the modern
`config:import` style.

This change yielded some results — 72% of the classes are now loaded from the shared archive.

### Spring Cloud openfeign

The second culprit identified was `spring-cloud-openfeign`. It creates a new named application context
for every Feign client. These separate application contexts triggered the early application exit.

Fix: Replaced Feign clients
with [Spring Interface Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface).

After removing Spring Cloud Openfeign, 87% of classes were loaded from the shared archive and tne startup time was
24–28&nbsp;s

# Summary & Conclusion

Our optimizations reduced startup time by 12–17&nbsp;s:

* Before any optimizations: 36–45&nbsp;s
* Exploded Jar without CDS: 27–34&nbsp;s
* Exploded Jar with CDS (and problematic dependencies removed): 24–28&nbsp;s

Impact will vary per application, but given their simplicity, using exploding jars and enabling CDS are likely a
worthwhile addition to your Spring Boot deployment pipeline.

# Next Steps

Future optimization avenues include:

* [Spring AOT](https://docs.spring.io/spring-framework/reference/core/aot.html) on the JVM
* [Project CraC](https://docs.spring.io/spring-boot/reference/packaging/checkpoint-restore.html#packaging.checkpoint-restore)
* [GraalVM Native Images](https://spring.io/blog/2023/12/04/cds-with-spring-framework-6-1)


