---
title: "Is Kotlin a Viable Enterprise-Grade Replacement for Java?"
meta_title: "Kotlin vs Java for Enterprise Development"
description: "An evaluation of Kotlin as an enterprise-grade replacement for Java, covering advantages, disadvantages, tooling, and community adoption."
date: 2020-01-15T00:00:00Z
image: ""
categories: ["Programming", "JVM"]
author: "kibertoad"
tags: ["kotlin", "java", "enterprise"]
draft: false
---

We've had an internal discussion regarding whether or not Kotlin is a viable enterprise-grade replacement for Java that is a strictly better (or close to that) alternative. Hopefully our findings might be of interest for a broader audience!

## Advantages

- **Reduction in boilerplate code.** There is anecdotal evidence that after migration from Java to Kotlin codebase decreases in size by 30-40% (e.g. from 12,371 lines of Java to 8,564 lines of Kotlin);
- **Inherent null-safety**, which helps developers to write robust code that wouldn't crash due to dreaded NullPointerExceptions;
- **Additional functional programming features** (such as high-order function support) enable a more functional development style, which is an excellent fit for data manipulation use-cases;
- **Coroutines** are a very powerful tool for asynchronous or highly concurrent programming;
- **Multiplatform development** (same code written in Kotlin can be compiled to both JVM, native binary and JS) so business logic can be written once and run both in backend and frontend.

## Disadvantages

- **Learning curve.** While Kotlin is not an inherently difficult language to learn, its syntax is different from Java and would take some time for developers to get used to. JetBrains provides online exercises for learning Kotlin;
- **Compilation time:** Java compiles 10-15% faster than Kotlin when doing clean builds. When incremental builds are enabled, Kotlin compiles as fast or slightly faster than Java.

## Language Support/Maturity

- The first version of Kotlin came out in 2012;
- Version 1.0 was released in 2016;
- The language is constantly evolving and adding new features, yet keeping backward compatibility;
- It's backed by JetBrains (the company behind IDEs such as IntelliJ IDEA).

## IDE Support

- First-class support in both Community and Ultimate editions of IntelliJ IDEA;
- Mature and popular plugin for Visual Studio Code available.

## Infrastructural Support

No change from Java, as Kotlin can be compiled to Java bytecode. Therefore change is transparent from the deployment perspective.

## Tooling Compatibility

- Compatibility with Maven is provided via Maven plugin;
- Compatibility with Gradle is provided via Gradle plugin; Gradle also supports Kotlin syntax;
- There's a Sonar plugin for Kotlin;
- Kotlin can compile to JVM bytecode, so the same running environment for Java works as well for Kotlin;
- Kotlin can be transpiled to JavaScript and native binaries so, potentially, the same environment for JS and native builds could be reused.

## Community Adoption

In 2019 Google declared Kotlin to be a recommended language for Android development.

## Framework Maturity

- Any Java framework can be used without compatibility issues;
- There're also Kotlin libraries for building backends like HTTP4K or Ktor;
- Spring has "a dedicated Kotlin support in Spring Framework 5.0", which "lets developers write Kotlin applications almost as if Spring Framework were a native Kotlin framework". They even have dedicated documentation;
- Juergen Hoeller, a Spring framework project lead, personally is a Kotlin enthusiast and encouraged Java developers to try out Kotlin in many of his conference speeches.

## Monitoring, Health, Tracing Solutions

Any Java solutions can be used without compatibility issues.

## Performance

No observable performance differences with Java.

## Interoperability

Kotlin supports gradual, step by step migration of codebases from Java to Kotlin; you can call your own Java code or use any external Java libraries from your Kotlin code.

## Literature

Considerable amount of high-quality books, including ones from major publishers are available, for an example: "Kotlin in Action", "Head First Kotlin", "Kotlin Programming: The Big Nerd Ranch Guide", "Kotlin for Android Developers", "The Joy of Kotlin".

## Maturity of Standards/Guidelines

Same best practices that apply to Java work here. They only need to be adapted to the language syntax.
