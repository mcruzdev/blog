---
draft: true 
date: 2024-01-02
categories:
  - Java
  - Quarkus
---
# How does the Quarkus build process work? 

## TL;DR

This post explores the Quarkus build process for your Quarkus application.

## Quarkus concepts

**Quarkus** is both a framework and a build-time **augmentation** toolkit. Applications built with Quarkus boast exceptionally fast startup times and a minimal footprint, thanks of its augmentation process.

## Augmentation

### What is augmentation?


In Quarkus, augmentation refers to a process where the framework optimizes and enhances your application during the build time:

![augmentation](augmentation.png)

### Maven Plugin

If you see in a Quarkus Application (`pom.xml` file) you will see the following configuration:

```xml linenums="1" hl_lines="9"
      <plugin>
        <groupId>${quarkus.platform.group-id}</groupId>
        <artifactId>quarkus-maven-plugin</artifactId> <!-- (1) -->
        <version>${quarkus.platform.version}</version>
        <extensions>true</extensions>
        <executions>
          <execution>
            <goals>
              <goal>build</goal> <!-- (2) -->
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```

1.  Quarkus Maven Plugin
2.  Goal that calls the [`BuildMojo.java`](https://github.com/quarkusio/quarkus/blob/e87a492ecbd83a20a23c8779b166f297136e686a/devtools/maven/src/main/java/io/quarkus/maven/BuildMojo.java#L35) class.

The line `9` is the line that configures the Maven plugin to call the class [`BuildMojo.java`](https://github.com/quarkusio/quarkus/blob/e87a492ecbd83a20a23c8779b166f297136e686a/devtools/maven/src/main/java/io/quarkus/maven/BuildMojo.java#L35):

```java
@Mojo(name = "build", defaultPhase = LifecyclePhase.PACKAGE, requiresDependencyResolution = ResolutionScope.COMPILE_PLUS_RUNTIME, threadSafe = true)
```

!!! info "Note"
    The value for the `defaultPhase` element is `LifecyclePhase.PACKAGE`, signifying that it will run during the package phase – which makes sense. Additionally, the `requiresDependencyResolution` element has a value of `ResolutionScope.COMPILE_PLUS_RUNTIME`, indicating that the plugin requires resolution of both compile and runtime dependencies.

## BuildMojo.java

This class extends `QuarkusBootstrapMojo.java`. The abstract `QuarkusBootstrapMojo.java` class aims to facilitate code reuse and enable the use of the [Template Method Design Pattern](https://refactoring.guru/design-patterns/template-method) through [`beforeExecute()`](https://github.com/quarkusio/quarkus/blob/e87a492ecbd83a20a23c8779b166f297136e686a/devtools/maven/src/main/java/io/quarkus/maven/QuarkusBootstrapMojo.java#L204) method.

But the principal method of `BuildMojo.java` is the `doExecute()`, when does:

1. `BuildMojo.java`'s principal method is `doExecute()`, which goes through the following steps:
    - Calls `super#bootstrapApplication()` to generate a `CuratedApplication` object.

2. The `CuratedApplication` object triggers:
    - `CuratedApplication#createAugmentor()`, creating an instance named `action` of type `AugmentAction`.

3. Within `action`, it performs:
    - Invokes augmentation via `AugmentResult`, storing the result of the augmentation process.

4. Renaming the JAR file occurs under the condition:
    - If `result.getJar() != null` evaluates to `true`.

