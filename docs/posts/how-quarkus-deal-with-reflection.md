---
comments: true
title: How Quarkus deal with Reflection
draft: false 
date: 2024-02-14
authors:
  - mcruzdev
categories:
  - Java
  - GraalVM
  - Quarkus
---

# How Quarkus deal with reflection

When running a native application, reflection cannot be used in the same way as in applications running on top of the JVM. All code that will be executed during runtime needs to be known at build time. So, how does Quarkus deal with reflection to generate a native image using Graal SDK?

In this post, you'll see how Quarkus deals with reflection using Ahead-Of-Time (AOT) compilation with GraalVM.

<!-- more -->

> If you want to do the same steps with me, you need to install Graal VM. I used [SDKMAN!](https://sdkman.io/) to install GraalVM. I am using GraalVM Native image tool in the version `17.0.10 2024-01-16` 

## The Closed World assumption

GraalVM uses static analysis to determine which code is used by the application. These elements are referred to as **reachable code**, meaning all code reachable in the static analysis. Only **reachable** code is included in the final image. Once the final image is built, there is no way to add more elements to it. This is known as the **Closed World Assumption**. 

This is a summary from the GraalVM documentation, if you want to learn more about it, [see the official documentation](https://www.graalvm.org/latest/reference-manual/native-image/basics/#static-analysis).

This implies that there is no reflection in a native image (like in JVM); you either add all code that will be executed and known, or you do not add it.

## Generatig a Native Image

If you're familiar with Java, you know that the following class should run perfectly on the JVM.

```java linenums="1"
public class Main {
    public static void main(String ...args) {
        System.out.println("Hello World");
    }
}
```

We need to compile it using the `javac` Java compiler.


```shell
javac Main.java
```

And execute it using `java`.

```shell
java Main
Hello World
```

### Generating a Native Image with native-image tool

If we want to generate a native image, we can use the Graal VM native-image tool.

```shell
native-image Main
```

And finally, execute the native image.

```shell
./main
Hello World
```

It works because all the code is known at build time.

## Adding seasoning 

Let's create a new class:

```java linenums="1"
public class Hero {

    private String name;
  
    public Hero(String name) {
      this.name = name;
    }
  
    public String getName() {
      return name;
    }

    @Override
    public String toString() {
        return "Hero: " + name;
    }
  }
```

And use it in the `main` method, using reflection.

```java linenums="1"
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class Main {
    public static void main(String... args) throws NoSuchMethodException, SecurityException, ClassNotFoundException,
            InstantiationException, IllegalAccessException, IllegalArgumentException, InvocationTargetException {
        System.out.println("Hello World");

        Class<?> forName = Class.forName(Hero.class.getCanonicalName());

        Constructor<?> constructor = forName.getConstructor(String.class);

        Object newInstance = constructor.newInstance("Batman");

        System.out.println(newInstance.toString());
    }
}
```

When we compile using `javac` and try to generate a native image using GraalVM, we may encounter the following message:

```shell
Warning: Reflection method java.lang.Class.getConstructor invoked at Main.main(Main.java:11)
Warning: Aborting stand-alone image build due to reflection use without configuration.
Warning: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
...
Finished generating 'main' in 11.9s.
Generating fallback image...
Warning: Image 'main' is a fallback image that requires a JDK for execution (use --no-fallback to suppress fallback image generation and to print more detailed information why a fallback image was necessary).
```


As we can see, this is not a stand-alone application; a JDK is necessary for execution. However, when we want to generate a native image, we typically don't need a JDK. Let's try using the `--no-fallback` option.

```shell
native-image Main --no-fallback
```

When executing this native image:

```shell
./main
Hello World
Exception in thread "main" java.lang.ClassNotFoundException: Hero
        at org.graalvm.nativeimage.builder/com.oracle.svm.core.hub.ClassForNameSupport.forName(ClassForNameSupport.java:123)
        at org.graalvm.nativeimage.builder/com.oracle.svm.core.hub.ClassForNameSupport.forName(ClassForNameSupport.java:87)
        at java.base@17.0.10/java.lang.Class.forName(DynamicHub.java:1322)
        at java.base@17.0.10/java.lang.Class.forName(DynamicHub.java:1285)
        at java.base@17.0.10/java.lang.Class.forName(DynamicHub.java:1278)
        at Main.main(Main.java:9)
```

We encountered a `ClassNotFoundException` exception. How can we solve this?

## GraalVM

GraalVM employs automatic detection to intercept and analyze specific reflection calls, such as `Class.forName()` and `Class.getDeclaredMethod()`, during build time. For more details see [here](https://www.graalvm.org/latest/reference-manual/native-image/dynamic-features/Reflection/#automatic-detection). But in some cases it is not possible, and we need to add some manual configurations.

### Register class for reflection manually

The Native Image Builder offers a way to configure a class for reflection using the following property.

```shell
-H:ReflectionConfigurationFiles=/path/to/reflectconfig
```

And the configuration for `Hero` class can be:

```json linenums="1"
[
  {
    "name" : "java.lang.String",
    "fields" : [
      { "name" : "name" }
    ],
    "methods" : [
      { "name" : "<init>", "parameterTypes" : ["java.lang.String"] }
    ]
  }
]
```

Let's rebuild our image, this time adding the necessary configuration.


```shell
native-image Main --no-fallback -H:ReflectionConfigurationFiles=hero-reflection-config.json
```

And see the result:

```shell
./main
Hello World
Hero: Batman
```

### Using GraalVM Tracing Agent

There is another way to generate those configuration files and metadata, is using the [GraalVM Tracing Agent](https://www.graalvm.org/latest/reference-manual/native-image/metadata/AutomaticMetadataCollection/), but it implies to execute your application. The agent **will look only the executed code** and generate all necessary configuration files and metadata into the specific directory.

Below, you can see an example of the use of `native-image-agent`.

```shell
java -agentlib:native-image-agent=config-output-dir=/path/to/config-dir/,config-write-period-secs=300,config-write-initial-delay-secs=5 -jar my-app.jar
```

!!! Note

    This agent will only look at executed code. It means that if the unknown code is not executed, the reflection configuration may not be generated.

## The Quarkus power!

As we can see, adding a configuration file works well. It is just a simple class utilizing reflection. However, when using frameworks like Hibernate or libraries that heavily rely on reflection, manual configuration becomes necessary to ensure compatibility with Native Image generation. This involves creating Reflection Configuration Files to explicitly define the reflection usage and enable it within the Native Image, ensuring seamless integration with such libraries in native runtime environments. It is worth noting that this task can be quite challenging and time-consuming.

### How Quarkus deal with reflection?

**The answer is using Build Items and something else...**

Quarkus has some build items to register a class for reflection:

- [ReflectiveClassBuildItem](https://github.com/quarkusio/quarkus/blob/main/core/deployment/src/main/java/io/quarkus/deployment/builditem/nativeimage/ReflectiveClassBuildItem.java)
- [ReflectiveBeanClassBuildItem](https://github.com/quarkusio/quarkus/blob/main/extensions/arc/deployment/src/main/java/io/quarkus/arc/deployment/ReflectiveBeanClassBuildItem.java)
- [ReflectiveMethodBuildItem](https://github.com/quarkusio/quarkus/blob/main/core/deployment/src/main/java/io/quarkus/deployment/builditem/nativeimage/ReflectiveMethodBuildItem.java)
- [ReflectiveFieldBuildItem](https://github.com/quarkusio/quarkus/blob/main/core/deployment/src/main/java/io/quarkus/deployment/builditem/nativeimage/ReflectiveFieldBuildItem.java)
- [ReflectiveClassConditionBuildItem](https://github.com/quarkusio/quarkus/blob/main/core/deployment/src/main/java/io/quarkus/deployment/builditem/nativeimage/ReflectiveClassConditionBuildItem.java)

There are additional build items to handle reflective operations in Quarkus. You can find more details about them [here](link-to-more-details).

### A real-world example

Let's take the Flyway extension for Quarkus as an example. Flyway library [finds](https://github.com/flyway/flyway/blob/bf907e6c8806134acd325927196c8a5e1fa2f9d4/flyway-core/src/main/java/org/flywaydb/core/internal/resolver/java/ScanningJavaMigrationResolver.java#L41) `JavaMigration` implementors to execute migrations using those implementors using reflection.

To register the reflection for Flyway and to generate a native image without to worry about the reflection. [The Quarkus Flyway extension does](https://github.com/quarkusio/quarkus/blob/b70ea4565f75efe0ea903b9f790af0436df5478a/extensions/flyway/deployment/src/main/java/io/quarkus/flyway/deployment/FlywayProcessor.java#L145):

1. Uses `CombinedIndexBuildItem` (Jandex) to find all `JavaMigration` implementors.
2. Produce a `BuildProducer<ReflectiveClassBuildItem>` to produce a `ReflectiveClassBuildItem` containing all necessary `JavaMigration` implementors.


After all, Quarkus will consume each `ReflectiveClassBuildItem` into the class [NativeImageReflectConfigStep](https://github.com/quarkusio/quarkus/blob/b70ea4565f75efe0ea903b9f790af0436df5478a/core/deployment/src/main/java/io/quarkus/deployment/steps/NativeImageReflectConfigStep.java#L26).

And after, for each `ReflectionInfo` will mount the necessary JSON configuration file for Native Image Builder, using another producer called [`GeneratedResourceBuildItem`](https://github.com/quarkusio/quarkus/blob/b70ea4565f75efe0ea903b9f790af0436df5478a/core/deployment/src/main/java/io/quarkus/deployment/steps/NativeImageReflectConfigStep.java#L127) into the file `META-INF/native-image/reflect-config.json`. 

## Thank you!

I hope you learned a lot from this blog post. Thank you so much!

If you have any suggestions or comments, please let me know below!


