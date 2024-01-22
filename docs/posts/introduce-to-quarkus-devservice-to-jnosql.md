---
title: Introduction to Quarkus DevService with Quarkus JNoSQL
draft: false 
date: 2024-01-20
authors:
  - mcruzdev
categories:
  - Java
  - Quarkus
  - JNoSQL
  - Docker
  - TestContainers
---

# Introduction to Quarkus DevService with Quarkus JNoSQL

In this post, we will get an introduction to JNoSQL and Quarkus DevServices, two great tools that facilitate our lives as developers.

We will implement a simple DevService feature for `quarkus-jnosql` project. Basically, if you change to a CouchDB database the extension will provide a CouchDB container for you, if you change to a ArangoDB database the extension will provide a ArangoDB container for you!

With this post, you will be able to understand how Quarkus DevService works and get started with Quarkus extension contribution. Why not?

<!-- more -->
## JNoSQL

Recently, in the Java world, the [JakartaOne](https://jakartaone.org/) event took place in Portuguese. In this event, [Max Arruda](https://twitter.com/maxdearruda) talked about JNoSQL. In a simple way, in my humble opinion, I can summarize: JNoSQL is the [Strategy](https://refactoring.guru/design-patterns/strategy) pattern but for switching database implementations. What does it mean? For example, if you are using a key-value database like Redis and you want to change to another key-value database like ArangoDB, you just need to add the implementation (ArangoDB), make a simple configuration change, and voilà, you are now using ArangoDB to persist all the necessary data.

## Quarkus DevServices

If you are using Quarkus, you already know about the developer experience that Quarkus provides. DevServices is just the same thing: it increases the developer experience when you are coding a Quarkus application. Basically, if your application needs to access a database or send a message to Kafka, Quarkus provides it through `application.properties` configurations. It is an amazing experience because you do not have to worry about accessing the Kafka Docker website, obtaining the necessary configuration to start the Kafka container.

## Knowing the quarkus-jnosql project

Let's think that you want to use the Quarkus JNoSQL extension with ArangoDB implementation, to do it, you need to create a Quarkus application and add the following dependency into your `pom.xml`, it is very simple, looks:


```xml
<dependency>
  <groupId>io.quarkiverse.jnosql</groupId>
  <artifactId>quarkus-jnosql-document-arangodb</artifactId>
  <version>3.2.2.2</version>
</dependency>

<dependency>
  <groupId>org.eclipse.jnosql.databases</groupId> 
  <artifactId>jnosql-arangodb</artifactId>
  <version>1.0.4</version>
</dependency>
```

And, to configure the `application.properties` file:

```properties
jnosql.document.database=arangodb
jnosql.arangodb.host=localhost:8529
jnosql.arangodb.password=openSesame
```

And, to execute the container image containing ArangoDB database:

```sh
docker run -p 8529:8529 -e ARANGO_ROOT_PASSWORD=openSesame arangodb/arangodb:3.11.6
```

Ok, it is relatevily simple, no? But, we can dot it more simple with DevServices.

### Code implementation

Our API will have a simple `Developer` entity and a simple resource. Below, you can see the `Developer` entity:

```java linenums="1"
package dev.matheuscruz;

import java.util.Objects;
import com.arangodb.serde.jackson.Id;
import jakarta.nosql.Column;
import jakarta.nosql.Entity;

@Entity
public class Developer {

    @Id
    String id;

    @Column
    String name;

    @Column
    String github;

    public Developer() {
    }

    public Developer(String name, String github) {
        this.name = Objects.requireNonNullElse(name, "anonymous");
        this.github = Objects.requireNonNullElse(github, "https://github.com/anonymous");
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getGithub() {
        return github;
    }
}
```

The Resource:

```java linenums="1"
package dev.matheuscruz;

import java.net.URI;
import java.util.List;
import jakarta.inject.Inject;
import jakarta.nosql.document.DocumentTemplate;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.Response;

@Path("/devs")
public class DeveloperResource {

    @Inject
    DocumentTemplate template;

    @POST
    public Response create(DevRequest request) {

        Developer dev = new Developer(request.name, request.github);
        Developer newDev = template.insert(dev);
        return Response.created(URI.create("/devs/" + newDev.getId())).build();
    }

    @GET
    public Response all() {
        List<Developer> all = template.select(Developer.class).result();
        return Response.ok(all).build();
    }

    record DevRequest(String name, String github) {
    }
}
```

If you access the API endoint through the following `cURL`, you will create a `Developer`.

```sh
curl --request POST \
  --url http://localhost:8080/devs \
  --header 'Content-Type: application/json' \
  --header 'User-Agent: insomnia/8.5.1' \
  --data '{
	"name": "Matheus Cruz",
	"github": "https://github.com/mcruzdev"
}'
```

To get all users, execute the following `cURL` command:

```sh
curl --request GET \
  --url http://localhost:8080/devs \
  --header 'User-Agent: insomnia/8.5.1'
```

You can see something like this:

```json
[
	{
		"id": "Developer/179",
		"name": "Matheus Cruz",
		"github": "https://github.com/mcruzdev"
	}
]
```

Done, we are using Quarkus JNoSQL with ArangoDB, and for now, the idea here is not to change the implementation (classes, etc.). We want to keep the codebase intact. If we decide to change the database, it is necessary only to modify the `pom.xml` (dependencies) and the `application.properties` file (configurations).


## Getting Started with DevServices

If you saw the last section, it was necessary to execute a Docker container with some configurations, right? A Quarkus extension is not just to provide build-time augmentation; it is meant to offer a better developer experience for extension users.

Let's offer for our extension users a way to provide the container when they change the `jnosql.document.database` property.

!!! info

    The goal here is to demonstrate how Quarkus DevServices works in practice. It does not reflect a real-world implementation but can serve as a starting point for you. We will implement DevServices for both ArangoDB Document and CouchDB Document.

We will divide this in 3 steps:

1. Look at the `jnosql.document.database` to determine which database we are using.
2. Read the configuration property (only the necessary to run the container) for each database type. This is necessary because a database configuration differs from another one.
3. Start a container for ArangoDB or CouchDB if necessary.


### 1. Reading the jnosql.document.database property

!!! tip "Developing a Quarkus extension"

    
    If you are not familiar with Quarkus extensions, I recommend checking out [my first post on this subject](https://matheuscruz.dev/#developing-a-quarkus-extension).


Having the `quarkus-jnosql` [fork](https://github.com/quarkiverse/quarkus-jnosql/fork), let's create the class responsible for DevService:

```java linenums="1"
package io.quarkiverse.jnosql.document.deployment;

import io.quarkus.deployment.annotations.BuildStep;
import org.eclipse.microprofile.config.ConfigProvider;

public class DocumentDevServices {
    @BuildStep
    public void build() {
    }
}
```

There are some ways to read the `application.properties`, to do it we will use the `ConfigProvider` class:

```java linenums="8"
@BuildStep
public void build() {

    Set<String> allowedDatabases = Set.of("arangodb", "couchdb"); // (1)
    String database = ConfigProvider.getConfig()
            .getOptionalValue("jnosql.document.database", String.class)
            .orElse(""); // (2)
    if (!allowedDatabases.contains(database)) {
        LOGGER.warn("This extensions does provide support only for [arangodb, couchdb] document databases");
        return;
    }
}
```

1. We are configuring all allowed databases;
2. We are using `ConfigProvider.getConfig()` method to get the `jnosql.document.database` value;

### 2. Reading specific database configuration

The goal here is to obtain only the necessary configuration to run the container with the application. 

For the ArangoDB container, the minimal configuration to execute the container is `docker run -p 8529:8529 -e ARANGO_ROOT_PASSWORD=openSesame arangodb/arangodb:3.11.6`, mapping to jnosql configuration, we need:

```properties
jnosql.document.database=arangodb
jnosql.arangodb.host=localhost:8529
jnosql.arangodb.password=openSesame
```

For the CouchDB container, the minimal configuration to execute the container is `docker run -e COUCHDB_PASSWORD=password -e COUCHDB_USER=admin -p 5984:5984 couchdb`, mapping to jnosql configuration, we need:

```properties
jnosql.document.database=couchdb
jnosql.couchdb.port=5984
jnosql.couchdb.host=localhost
jnosql.couchdb.password=password
jnosql.couchdb.username=admin
```

### 3. Start a container ...

To use [TestContainers](https://java.testcontainers.org/) we need to add the dependency in `pom.xml`:

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
</dependency>
```

The TestContainer library provides a class that allows us to run and control a container. This class is called `GenericContainer<T>`. We need to create two classes that extend `GenericContainer<T>`, where we will include the necessary code to configure the container.


### CouchDBContainer
```java linenums="1"
    // ... ommited
    static class CouchDBContainer extends GenericContainer<CouchDBContainer> {
        public static final Integer COUCHDB_DEFAULT_PORT = 5984;
        public CouchDBContainer() {
            super(DockerImageName.parse("couchdb:latest")); // (1)
        }
    }
```

1. We are defining the image name

### ArangoDBContainer
```java
    // ommited
    static class ArangoDBContainer extends GenericContainer<ArangoDBContainer> {
        public static final Integer ARANGO_DB_DEFAULT_PORT = 8529;
        public ArangoDBContainer() {
            super(DockerImageName.parse("arangodb:latest")); // (1)
        }
    }
```

1. We are defining the image name

Now, let's use those class into our `@BuildStep` method.


```java linenums="1" hl_lines="1 5 17-19 30 34-38"
@BuildStep(onlyIfNot = { IsNormal.class }, onlyIf = { GlobalDevServicesConfig.Enabled.class })
public void build(BuildProducer<DevServicesResultBuildItem> devServicesProducer) {

    Set<String> allowedDatabases = Set.of("arangodb", "couchdb");
    String database = ConfigProvider.getConfig()
            .getOptionalValue("jnosql.document.database", String.class)
            .orElse("");
    if (!allowedDatabases.contains(database)) {
        LOGGER.warn("This extensions does provide support only for [arangodb, couchdb] document databases");
        return;
    }

    LOGGER.info("jnosql.document.database === {}", database);

    if (database.equals("couchdb")) {

        Integer bindPort = ConfigProvider.getConfig().getValue("jnosql.couchdb.port", Integer.class);
        String password = ConfigProvider.getConfig().getValue("jnosql.couchdb.password", String.class);
        String username = ConfigProvider.getConfig().getValue("jnosql.couchdb.username", String.class);

        CouchDBContainer couchDBContainer = new CouchDBContainer()
                .withEnv(
                        Map.of("COUCHDB_PASSWORD", password,
                                "COUCHDB_USER", username))
                .withExposedPorts(CouchDBContainer.COUCHDB_DEFAULT_PORT);

        couchDBContainer.setPortBindings(
                List.of(String.format("0.0.0.0:%d:%d", CouchDBContainer.COUCHDB_DEFAULT_PORT, bindPort)));

        couchDBContainer.start();

        LOGGER.info(couchDBContainer.getLogs());

        devServicesProducer.produce(new DevServicesResultBuildItem.RunningDevService(
                "jnosql-document",
                couchDBContainer.getContainerId(),
                couchDBContainer::close,
                Map.of()).toBuildItem());
    } else {
        String host = ConfigProvider.getConfig().getValue("jnosql.arangodb.host", String.class);
        String[] hostPort = host.split(":");
        Integer bindPort = Integer.valueOf(hostPort[1]);

        String password = ConfigProvider.getConfig().getValue("jnosql.arangodb.password", String.class);

        ArangoDBContainer arangoDBContainer = new ArangoDBContainer()
                .withExposedPorts(ArangoDBContainer.ARANGO_DB_DEFAULT_PORT)
                .withEnv("ARANGO_ROOT_PASSWORD", password);

        arangoDBContainer.setPortBindings(
                List.of(String.format("0.0.0.0:%d:%d", ArangoDBContainer.ARANGO_DB_DEFAULT_PORT, bindPort)));

        arangoDBContainer.start();

        LOGGER.info(arangoDBContainer.getLogs());

        devServicesProducer.produce(new DevServicesResultBuildItem.RunningDevService(
                "jnosql-document",
                arangoDBContainer.getContainerId(),
                arangoDBContainer::close,
                Map.of()).toBuildItem());
    }
}

static class ArangoDBContainer extends GenericContainer<ArangoDBContainer> {
    public static final Integer ARANGO_DB_DEFAULT_PORT = 8529;

    public ArangoDBContainer() {
        super(DockerImageName.parse("arangodb:latest"));
    }
}

static class CouchDBContainer extends GenericContainer<CouchDBContainer> {
    public static final Integer COUCHDB_DEFAULT_PORT = 5984;

    public CouchDBContainer() {
        super(DockerImageName.parse("couchdb:latest"));
    }
}
```

!!! info "Disclaimer"

    All the code provided here is crafted to streamline the utilization of TestContainer. In a real-world scenario, it is essential to consider and adhere to best practices in coding.


- In line **1** we are defining that this `@BuildStep` method will be called if the `BooleanSupplier` `IsNormal.class` resolves to false (`onlyIfNot = { IsNormal.class }`) and if the `quarkus.devservices.enabled` (`onlyIf = { GlobalDevServicesConfig.Enabled.class }`) value is true. This is necessary because DevServices only run in `Test` and `Dev` modes and in our scenario if Quarkus DevServices is enabled.

- At line **5**, we are retrieving the value of the `jnosql.document.database` property to obtain the database name to be used later.
- In lines **17-19** we are reading the specific configuration for CouchDB.
- At line **30** we are starting the container.
- In lines **34-38**, we are using the `BuildProducer<DevServicesResultBuildItem>` instance to produce a `DevServicesResultBuildItem` build item, which will be utilized later by Quarkus. Note that we are using the `CouchDBContainer::close` `Closeable` reference in the `BuildStep`. This is very useful for Quarkus, as it allows us to "close" the container when the `Test` or `Dev` mode is finished.


The same thing is made for ArangoDB database.

### Testing DevServices

After changing the `quarkus-jnosql` project, we will install it locally. Go to the `quarkus-jnosql` directory and execute:

```sh
mvn clean install
```

You need now, to install it in your Quarkus project using a `SNAPSHOT` version. Today, in my case the latest version is `3.2.2.2-SNAPSHOT`, add the following dependency into your Quarkus project. **We will test the CouchDB first**.


```xml
<dependency>
  <groupId>io.quarkiverse.jnosql</groupId>
  <artifactId>quarkus-jnosql-document-couchdb</artifactId>
  <version>3.2.2.2-SNAPSHOT</version>
</dependency>
```

Now, if you change the `jnosql.document.database` property value to `couchdb` you will get a container running in your local development or test.

### Testing CouchDB

Necessay dependencies:

```xml
<dependency>
  <groupId>io.quarkiverse.jnosql</groupId>
  <artifactId>quarkus-jnosql-document-couchdb</artifactId>
  <version>3.2.2.2-SNAPSHOT</version>
</dependency>

<dependency>
  <groupId>org.eclipse.jnosql.databases</groupId>
  <artifactId>jnosql-couchdb</artifactId>
  <version>1.0.4</version>
</dependency>
```

Necessary configurations:

```properties
jnosql.document.database=couchdb
jnosql.couchdb.port=5984
jnosql.couchdb.host=localhost
jnosql.couchdb.password=password
jnosql.couchdb.username=admin

quarkus.devservices.enabled=true
```

Running `quarkus dev` and using the API, you can see that the application works well!

??? tip "Request to create dev"

    ```sh
    curl --request POST \
      --url http://localhost:8080/devs \
      --header 'Content-Type: application/json' \
      --header 'User-Agent: insomnia/8.5.1' \
      --data '{
      "name": "Matheus Cruz",
      "github": "https://github.com/mcruzdev"
    }'
    ```

??? tip "Request to get all devs"

    ```sh
      curl --request GET \
      --url http://localhost:8080/devs \
      --header 'User-Agent: insomnia/8.5.1'
    ```

Result after creating a dev and getting all:

```json
[
	{
		"id": "3ff80981-925b-43f6-bcce-65a86469400a",
		"name": "Matheus Cruz",
		"github": "https://github.com/mcruzdev"
	}
]
```

### Chaning to ArangoDB

Necessary dependencies:

!!! info "Note"

    Remove the ArangoDB dependencies first.


```xml
<dependency>
  <groupId>io.quarkiverse.jnosql</groupId>
  <artifactId>quarkus-jnosql-document-arangodb</artifactId>
  <version>3.2.2.2-SNAPSHOT</version>
</dependency>

<dependency>
  <groupId>org.eclipse.jnosql.databases</groupId>
  <artifactId>jnosql-arangodb</artifactId>
  <version>1.0.4</version>
</dependency>
```

Necessary properties:

```properties
jnosql.document.database=arangodb
jnosql.arangodb.host=localhost:8529
jnosql.arangodb.password=openSesame
```

Result after creating a dev and getting all:

```json
[
	{
		"id": "Developer/170",
		"name": "Matheus Cruz",
		"github": "https://github.com/mcruzdev"
	}
]
```

## Source code 

The source code from this post can be reached [here](https://github.com/mcruzdev/quarkus-jnosql/tree/blog-post-jnosql-devservices) (JNoSQL fork) and [here](https://github.com/mcruzdev/quarkus-jnosql-devservices)(Quarkus Application that uses Quarkus JNoSQL fork)!

## Thank you

That's all; thank you for reading! See you in the next post. Goodbye! :wave: