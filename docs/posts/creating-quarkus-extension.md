---
draft: true 
date: 2024-01-02
authors:
  - mcruzdev
categories:
  - Java
  - Quarkus
  - Gizmo
---

# Developing a Quarkus Extension

## TL;DR

This guide demonstrates how to create a Quarkus extension that provides three features that:

- Notify an API regarding the application's running status
- Offer a "straightforward" implementation with Gizmo
- Provide support for the [Resend library](https://resend.com/docs/send-with-java#1-install) within Quarkus

## Quarkus CLI

The Quarkus CLI is an incredibly useful tool for working with Quarkus.

There are several ways to install the Quarkus CLI. [Refer to the official documentation for installation instructions](https://quarkus.io/guides/cli-tooling#installing-the-cli).

With Quarkus CLI, we can `create`, `build`, `deploy`, and perform essential tasks in a developer's day-to-day workflow. We will use Quarkus CLI, be sure that is installed.

## What Will Our Extension Do?

- Notify an API regarding the application's running status;
- Offer a "straightforward" implementation with Gizmo;
- Provide a simple support for the [Resend library](https://resend.com/docs/send-with-java#1-install) within Quarkus;

!!! info "Note"
  
    All features are simple compared to real-world applications, but they serve as a great starting point to grasp and initiate your journey into the world of Quarkus extensions!

## Creating the Extension

Once the Quarkus CLI is installed, we will use it to create our useful extension.

```bash
quarkus create extension dev.matheuscruz:quarkus-useful:0.0.1-SNAPSHOT
```

The desired output should be similar to:

```bash
Detected layout type is 'standalone' 
Generated runtime artifactId is 'quarkus-useful'


applying codestarts...
📚 java
🔨 maven
📦 quarkus-extension
🚀 devmode-test
🚀 extension-base
🚀 integration-tests
🚀 unit-test

-----------
 👍  extension has been successfully generated in:
--> /home/cruz/github.com/mcruzdev/quarkus-useful
-----------
Navigate into this directory and get started: quarkus build
```

## Quarkus Extension structure

A Quarkus extension is divided in two modules, `deployment` and `runtime`.

- The `deployment` module contains `@BuildSteps`, `*BuildItem`s and `*Processor`s components that are used at build time.


- The `runtime` module contains runtime code that are used by `deployment` modul at build time. 

**It might seem odd: why is runtime code executed and utilized by build-time code?**

> The reason is that runtime code is designed to be recorded for execution later. When you write this code, it's intended to be executed or used at a later stage.

!!! warning "Note"

    A `deployment` module must depend on the `runtime` module.

## [Feature #1] Implementing the Notifier

**What do we need here, and what is the expected behavior?**

I want that, when my application start I want to send a HTTP request to a specific endpoint, notifying that I am starting.

!!! warning "Use Quarkus Lifecycle instead"

    This is merely a sample, demonstrating the capabilities of a Quarkus extension. Avoid using it in production scenarios. For a limited number of applications, consider utilizing the [Quarkus Lifecycle](https://quarkus.io/guides/lifecycle) feature instead.


### Creating our configuration

To call a REST service, you'll require the resource's URI. How can you retrieve this information or determine what the URI is?

We can do it:

```java
private static final String REST_ENDPOINT = "https://api.quarkus.com/apps"
```

But, it is not a good practice. Quarkus aims to facilitate the creation of Cloud Native applications, and a good Cloud Native application uses (when necessary) [12 Factors - Configuration](https://12factor.net/config) approach.

We need to make this configurable. I don't want to change the code and go through the entire release process (pull requests, approvals, pipelines, deployment, freezing, etc.) just to upload my extension with a new endpoint. To achieve this, Quarkus allows us to accomplish it using the [Configuration](https://quarkus.io/guides/config-reference) feature. You simply need to create a POJO class and annotate it with `@io.quarkus.runtime.annotations.ConfigRoot`. Let's proceed:


```java
package dev.matheuscruz.quarkus.useful.runtime;

import java.util.Optional;

import io.quarkus.runtime.annotations.ConfigItem;
import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;

@ConfigRoot(name = "useful", phase = ConfigPhase.BUILD_AND_RUN_TIME_FIXED)
public class UsefulConfiguration {

    /**
     * The listener URL to be notified about the 'starting' event of the
     * application.
     */
    @ConfigItem(name = "listenerUrl")
    public Optional<String> listenerUrl;
}
```

!!! tip "Defining a proprety as Optional"

    To define a configuration property as optional, you need to define the property as `Optional<T>`.

- The element `name` means the configuration property name. At the final the properties inside the `UsefulConfiguration` will be the prefix `quarkus.useful`.

- The element `phase` indicates when this configurable will be visible. We have possible choices:


??? info "ConfigPhase enum"

    - `BUILD_TIME` - Values are read and available for usage at build time.
    - `BUILD_AND_RUNTIME_FIXED` - Values are read and available for usage at build time, and available on a read-only basis at run time.
    - `RUN_TIME` - Values are read and available for usage at run time and are re-read on each program execution.

Perfect, now that we have the configuration available, let's use it.

### Calling the HTTP Service

As mentioned previously, all code that executes at runtime should be in the `runtime` module.

Let's create the call to notify the 'starting' event.

```java linenums="1"
package dev.matheuscruz.quarkus.useful.runtime;

import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpRequest.BodyPublishers;
import java.net.http.HttpResponse.BodyHandlers;

import org.eclipse.microprofile.config.ConfigProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import io.quarkus.runtime.annotations.Recorder;

@Recorder
public class NotifyStartingEventRecorder {

    private static final Logger LOGGER = LoggerFactory.getLogger(NotifyStartingEventRecorder.class);

    public void notify(UsefulConfiguration config) {
        if (config.listenerUrl.isEmpty()) {
            LOGGER.warn(
                    "You are using the 'quarkus-useful' extension but the configuration property quarkus.useful.listenerUrl not defined");
            return;
        }

        try {

            String applicationName = ConfigProvider.getConfig()
                    .getConfigValue("quarkus.application.name")
                    .getValue();

            String body = String.format("{ \"applicationName\": \"%s\" }", applicationName);
            HttpRequest httpRequest = HttpRequest.newBuilder(URI.create(config.listenerUrl.get()))
                    .POST(BodyPublishers.ofString(body))
                    .build();

            HttpClient httpClient = HttpClient.newHttpClient();

            HttpResponse<String> httpResponse = httpClient.send(httpRequest, BodyHandlers.ofString());

            LOGGER.info("The quarkus-useful-extension gets the HTTP status code: {}",
                    httpResponse.statusCode());
        } catch (IOException | InterruptedException e) {
            LOGGER.error("It was not possible to notify the listenerUrl {}.", config.listenerUrl.get());
        }   
    }
}
```

!!! info "@Recorder annotation"

    The annotation `@Recorder` indicates that the given type is a recorder that is used to record actions to be executed at runtime.

Perfectly, we have created our code that represents the code to be recorded. We will create now, the build step that will record this piece of code.


### Recording the `notify(UsefulConfiguration config)` method

Now, we will create the class responsible for recording our code.

As mentioned previously all code related to the build process should be in `deployment` module. If you see into `deployment` module, there is a class called `QuarkusUsefulProcessor` (generated by Quarkus CLI):

```java hl_lines="10-13" linenums="1"
package dev.matheuscruz.quarkus.useful.deployment;

import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.FeatureBuildItem;

class QuarkusUsefulProcessor {

    private static final String FEATURE = "quarkus-useful";

    @BuildStep
    FeatureBuildItem feature() {
        return new FeatureBuildItem(FEATURE);
    }
}
```

The Quarkus CLI generated a method with the `@BuildStep` annotation, indicating that this method is a step in the build process. Build step methods can produce or consume build items. In this case, it produces one build item called `FeatureBuildItem`.

The `FeatureBuildItem` is used to include the feature (`quarkus-useful`) to be printed at startup, for example:

```sh 
Installed features: [cdi, quarkus-useful, resteasy, smallrye-context-propagation, vertx]
```

Let's create the method in the `QuarkusUsefulProcessor` class that will handle recording our notifier.


```java linenums="1"
@BuildStep
@Record(value = ExecutionTime.RUNTIME_INIT)
void recordNotifyStartingEventRecorder(UsefulConfiguration config,
        NotifyStartingEventRecorder recorder) {
    // the following call will be recorded
    recorder.notify(config);
}
```

**What does the `@Record` annotation mean, and what is the value `ExecutionTime.RUNTIME_INIT`?**

The `ExecutionTime` enum has two options: `RUNTIME_INIT` and `STATIC_INIT`.

Let's dive into some simple code examples to understand how these choices impact bytecode.

```java
public class GeneratedBytecode {

  static {
    // @Record(value = ExecutionTime.STATIC_INIT)
    // All code recorded with STATIC_INIT will be here.
  }

  public static void main(String args[]) {
    // @Record(value = ExecutionTime.RUNTIME_INIT)
    // All code recorded with RUNTIME_INIT will be here.
  }
}
```

??? tip "See more: How to see the generated bytecode"

    If you want to see the bytecode generated, into extension root dir runs:

    - `mvn clean install`
    - cd `integration-tests`
    - `mvn clean package -DskipTests -Dquarkus.package.vineflower.enabled=true   `
    - `cat target/decompiled/generated-bytecode/io/quarkus/runner/ApplicationImpl.java`

    You will see that in the method `doStart(String[] var1)` there is a piece of code like it: `(new QuarkusUsefulProcessor$recordNotifyStartingEventRecorder1857232657()).deploy(var2);`

    The option `quarkus.package.vineflower.enabled=true` tell to Quarkus decompile generated and transformed bytecode into the 'decompiled' directory. 

Perfect, if you want to test the notifier, execute:

```sh
mvn clean install

cd integration-tests

quarkus dev
```

The output should looks like:

```sh
2024-01-06 19:17:34,449 WARN  [dev.mat.qua.use.run.NotifyStartingEventRecorder] (Quarkus Main Thread) You are using the 'quarkus-useful' extension but the configuration property quarkus.useful.listenerUrl not defined
```

It happened because we forgot to set the `quarkus.useful.listenerUrl` property into `application.properties` file. Add the configuration property to `application.properties` and observe the result.


??? tip "Mocking HTTP request"

        Use https://app.beeceptor.com/ this service to mock the HTTP request.
    
## [Feature #2] Implementing our GreetingService interface

In some scenarios, offering a default implementation to our extension consumers becomes necessary, as seen in great libraries. 

Essentially, in our case the user will inject a `GreetingService` interface, and the extension will provide a default implementation behind the scenes (using Gizmo :smile:).

The interface:

```java linenums="1"
public interface GreetingService {
  String message();
}
```

The user's code:

```java
@Path("/greetings")
public class GreetingResource {

  @Inject
  GreetingService service;

  @GET
  @Produces(MediaType.TEXT_PLAIN)
  public String greeting() {
    return service.message();
  }
}
```

There are several ways to achieve this with Quarkus. One approach is implementing the interface and providing it by using CDI directly. However, let's get hands-on with Gizmo to get our initial experience.

### Gizmo

[Gizmo](https://github.com/quarkusio/gizmo) is a library used by Quarkus to generate bytecode. If you are interested, there is an amazing video by the Quarkus Core team that explains the library in a better way. You can watch it [here](https://www.youtube.com/watch?v=iZ501bG2ZAE).


### Creating the GreetingService

Into the `runtime` module, let's create our interface:

```java
package dev.matheuscruz.quarkus.useful.runtime;

public interface GreetingService {
    String message();
}
```

### Creating the implementation

Into the `deployment` module, let's create a new build step to generate our implementation.

```java linenums="1"
@BuildStep
void generateGreetingService(BuildProducer<GeneratedBeanBuildItem> generatedClasses) {
}
```

!!! info "@Record is not necessary"

    The `@Record` annotation isn't always mandatory. Sometimes, annotating the method solely with `@BuildStep` suffices to include it in the augmentation process. However, the `@Record` annotation becomes necessary when you intend to record a portion of runtime code.



There are two ways to generate a Build Item, the first one is returning the Build Item in method, and the second one is using the class `BuildProducer<T extends BuildItem>`.

Let's continue with the implementation:

```java linenums="1" hl_lines="3 6"  
@BuildStep
void generateGreetingService(BuildProducer<GeneratedBeanBuildItem> generatedClasses) {
      GeneratedBeanGizmoAdaptor gizmoAdapter = new GeneratedBeanGizmoAdaptor(generatedClasses);
    try (ClassCreator classCreator = ClassCreator.builder()
            .className("dev.matheuscruz.quarkus.useful.deployment.UsefulGreetingService")
            .interfaces(GreetingService.class)
            .classOutput(gizmoAdapter)
            .build()) {
                    
        classCreator.addAnnotation(ApplicationScoped.class);

        try (MethodCreator returnHello = classCreator.getMethodCreator("message",
                String.class)) {
            returnHello.setModifiers(Opcodes.ACC_PUBLIC);
            returnHello.returnValue(returnHello.load("Hello from Quarkus Useful extension"));
        }
    }
}
```

What this build step does ind depth? Let's see:

1. The line **"3"**  is an adapter that uses the `BuildProducer<GeneratedBeanBuildItem>` injected into `generateGreetingService` method.

2. The line **"6"** we are indicating that the new class will implements the `GreetingService` interface.

3. In the line "**10"** we are adding the annotation `jakarta.ws.rs.ApplicationPath` to the new class.

4. The `ClassCreator` is the class responsible for generating the bytecode, the instance of `GeneratedBeanGizmoAdaptor` is used as `ClassCreator.classOutput` property.

5. When the method `ClassCreator#close()` is called the `ClassCreator` writes the bytecode inside de `GeneratedBeanGizmoAdaptor` instance. Note that we are using `try-with-resources`.

6. Behind the scenes the `GeneratedBeanGizmoAdaptor` uses the `BuildProducer<GeneratedBeanBuildItem>` instance to produces the a `GeneratedBeanBuildItem` instance.

??? info "See more: ClassCreator#close() implementation"

    ```java
    public void close() {
        final ClassOutput classOutput = this.classOutput;
        if (classOutput != null) {
            writeTo(classOutput);
        }
    }
    ```


??? info "See more: GeneratedBeanGizmoAdaptor#write() implementation"

    ```java
        @Override
    public void write(String className, byte[] bytes) {
        String source = null;
        if (sources != null) {
            StringWriter sw = sources.get(className);
            if (sw != null) {
                source = sw.toString();
            }
        }
        classOutput.produce(new GeneratedBeanBuildItem(className, bytes, source));
    }
    ```

### Using our hidden GreetingService implementation

Now, that we have the default implementation, let's use our `integration-tests` module to test it.

We need now to create a new resource called `GreetingResource`:

```java linenums="1"
package dev.matheuscruz.quarkus.useful.it;

import dev.matheuscruz.quarkus.useful.runtime.GreetingService;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;

@Path("/greetings")
public class GreetingResource {

    @Inject
    GreetingService greetingService;

    @GET
    public String greeting() {
        return greetingService.message();
    }
}
```

Install the extension:

```sh
mvn clean install
```

Go to the `integration-tests` module directory (`cd integration-tests`) and execute:

```sh
quarkus dev
```

Open the browser and access the [resource](http://localhost:8080/greetings).

The output should shows: `Hello from Quarkus Useful extension`.

## [Feature #3] Provide support for the Resend library

The primary purpose of a Quarkus Extension is to seamlessly integrate an external library into the Quarkus ecosystem.

In this section, we will offer support for the [Resend](https://resend.com/overview) library.

We will add the `Resend` library into the `runtime` module.

```xml
<!-- runtime/pom.xml -->
<dependency>
    <groupId>com.resend</groupId>
    <artifactId>resend-java</artifactId>
    <version>2.2.1</version>
</dependency>
```

### Creating the configuration for Resend

The `com.resend.Resend` class contains a constructor that requires the API Key (`String`), which is sensitive data. As mentioned in the previous section, it is considered a best practice to make this configuration adjustable, particularly for its essential runtime usage in this scenario.

```java linenums="1" hl_lines="9"
package dev.matheuscruz.quarkus.useful.runtime;

import java.util.Optional;

import io.quarkus.runtime.annotations.ConfigItem;
import io.quarkus.runtime.annotations.ConfigPhase;
import io.quarkus.runtime.annotations.ConfigRoot;

@ConfigRoot(name = "useful-resend", phase = ConfigPhase.RUN_TIME)
public class ResendConfiguration {

    /**
     * The Resend API Key.
     */
    @ConfigItem(name = "apiKey")
    Optional<String> apiKey;
}
```

On line **"7"**, we set up a new configuration utilizing the `ConfigPhase.RUNTIME` phase, indicating that this configuration is visible during runtime.

### Creating the default Bean in `runtime`

This step is very simple, we just need to use the [CDI Specification](https://www.cdi-spec.org/) and produce a new configurable `com.resend.Resend` bean.


```java linenums="1"
package dev.matheuscruz.quarkus.useful.runtime;

import com.resend.Resend;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Singleton;

@ApplicationScoped
public class ResendProducer {

    @Produces
    @Singleton
    public Resend producesResend(ResendConfiguration config) {
        if (config.apiKey.isEmpty()) {
            throw new IllegalArgumentException("Resend apiKey cannot be empty");
        }
        return new Resend(config.apiKey.get());
    }
}
```

Now that we've produced a bean through our extension, let's use it into our `integration-tests` application.


Add the new property into `integration-tests/src/main/resources/application.properties` file. It is required because we have a `throw new IllegalArgumentException("Resend apiKey cannot be empty");` in the line **"16"**.

```properties
quarkus.useful-resend.apiKey=api-key
```

Install the application:

```bash
mvn clean install
```

Inject and add the new `com.resend.Resend` bean into the controller and use it:


