---
authors:
  - mcruzdev
draft: true 
date: 2023-12-29
categories:
  - Maven
  - Java
  - Quarkus
---

# Creating a Maven plugin

## TL;DR

In this tutorial you will create a Maven Plugin to add a new class (PingResource) to the project as bytecode using Gizmo.

## Why create a Maven plugin?

At times, we have an idea that doesn't exist yet, or it exists but is deprecated or doesn't work well.

We currently have some Maven plugins. Another motivation for creating a Maven plugin is to delve deeper into a technology. One helpful approach is attempting to simplify the implementation of the same technology, thereby understanding its internal workings and challenges more comprehensively.

<!-- more -->
## Getting Started

Maven offers a set of tools that assist us in creating Maven plugins.

A Maven Plugin is a Java project and can be created through Maven Archetypes:

```
mvn archetype:generate 
  -DgroupId=dev.matheuscruz 
  -DartifactId=ping-maven-plugin 
  -Dversion=0.0.1-SNAPSHOT 
  -DarchetypeGroupId=org.apache.maven.archetypes 
  -DarchetypeArtifactId=maven-archetype-mojo
```

## Maven Goals

If you want to know more about Maven Build Lifecycle, the most recommended source is the [official documentation](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html).

A plugin goal represents a specific task.



