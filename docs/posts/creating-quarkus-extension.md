---
draft: true 
date: 2024-01-02
categories:
  - Java
  - Quarkus
---

# Developing a Quarkus Extension

## TL;DR

This guide demonstrates how to create a Quarkus extension using the Quarkus CLI.

## Quarkus CLI

To start, let's set up the Quarkus CLI. This tool is incredibly useful for working with Quarkus.

There are several ways to install the CLI:

- SDKMAN!
- Homebrew
- Chocolatey
- Scoop

[Refer to the official documentation for installation instructions](https://quarkus.io/guides/cli-tooling#installing-the-cli)

## What Will Our Extension Do?

- Notify an API regarding the application's running status
- Offer a straightforward implementation with Gizmo
- Provide support for the [Resend library](https://resend.com/docs/send-with-java#1-install) within Quarkus

!!! note "Note"
  
    This feature is simple compared to real-world applications, but it serves as a great starting point to grasp and initiate your journey into the world of Quarkus extensions!

## Creating the Extension

Once Quarkus CLI is installed, let's utilize it:

```bash
quarkus create extension dev.matheuscruz:quarkus-useful:0.0.1-SNAPSHOT
