---
comments: true
title: Integration Tests with Quakus + Wiremock
draft: false 
date: 2024-02-12
authors:
  - mcruzdev
categories:
  - Java
  - Quarkus
  - Wiremock
  - TestContainers
---

# Integration Tests with Quarkus + Wiremock


## What is Integration Tests?

Integration tests are a type of software testing where individual software modules or components are combined and tested collectively. The purpose of integration testing is to verify that different parts of the system work together correctly when integrated. These parts may include a message broker, database, another web service, etc.

These tests serve to uncover bugs. Integration tests help identify bugs that arise from the interaction between components of your software.

If you are a Java developer, you have likely used Mockito to mock a database call, an HTTP request, etc. Mockito is well-suited for unit tests and makes perfect sense in that context. However, relying solely on unit tests may not provide 100% confidence that your software will perform flawlessly in production.

It is very cool, we are testing the behavior of `execute` method using a Unit Tests, is fast, but we can have issues if we did not add more test cases and to stress this use case.

<!-- more -->

### What is the problem when you just have Unit Tests?

![only-unit-tests](assets/only-unit-tests.png){align=right}

In the image, the bench and the grid work fine individually, but together they don't allow someone to sit comfortably.

Today, it's common for applications to interact with other services, store customer details, and send notifications about due dates or promotions. Testing the real communication between these components is challenging with just unit tests.

This blog post demonstrates using Wiremock with Quarkus for integration testing. While Wiremock isn't a silver bullet, it can reveal more issues in your system. Other techniques, like Contract Testing, functional testing, and end-to-end testing, also play crucial roles in ensuring system reliability.


## Steps to get your application tested with Wiremock

1. Set up the Quarkus project
2. Add Wiremock and Rest Client dependencies
3. Implement Resource layer
4. Implement Use Case layer
5. Implement Integration Tests
6. Execute the Test Suite


## 1. Creating a Quarkus application 

Is very simple to create a Quarkus application using Quarkus CLI. Let's create a Quarkus application with Quarkus CLI:

```shell
quarkus create app dev.matheuscruz:quarkus-wiremock
```

## 2. Add the following dependencies to `pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-rest-client-reactive-jackson</artifactId>
</dependency>

<dependency>
    <groupId>io.quarkiverse.wiremock</groupId>
    <artifactId>quarkus-wiremock</artifactId>
    <version>1.1.1</version>
    <scope>test</scope>
</dependency>
```

# 3. Implementing Resource layer

```java linenums="1"
package dev.matheuscruz;

import java.net.URI;
import dev.matheuscruz.CreateUser.CreateUserInput;
import dev.matheuscruz.CreateUser.CreateUserOutput;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.Response;

@Path("/users")
public class UserResource {

    CreateUser createUser;

    public UserResource(CreateUser createUser) {
        this.createUser = createUser;
    }

    @POST
    public Response create(CreateUserRequest req) {
        CreateUserOutput output = this.createUser.execute(new CreateUserInput(
                req.name(), req.jobTitle, req.jobTitle()));
        return Response.created(URI.create("/users/" + output.id())).build();
    }

    public record CreateUserRequest(String name, String jobTitle, String email) {}
}
```

## 4. Implementing Use Case layer

```java linenums="1"
package dev.matheuscruz;

import java.util.Map;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class CreateUser {

    private EmailSenderRestClient emailSender;

    @Inject
    public CreateUser(EmailSenderRestClient emailSender) {
        this.emailSender = emailSender;
    }

    public CreateUserOutput execute(CreateUserInput input) {
        User user = User.create(input.email(), input.name, input.jobTitle());

        this.emailSender.greetings(new EmailRequest("greetings", Map.of(
                "name", user.getName())));
        return new CreateUserOutput(user.getId());
    }

    public record CreateUserInput(String name, String email, String jobTitle) {}

    public record CreateUserOutput(String id) {}
}
```

## Implementing Integration Tests



## Wiremock

Wiremock is a handy tool for testing HTTP-based APIs. It lets you mock responses from external services during testing, making it easier to create consistent test environments. With Wiremock, you can customize responses, status codes, and more, helping you conduct thorough integration tests with ease.

## Creating the Integration Tests


