[//]: # (title: UAST)

<!-- Copyright 2000-2020 JetBrains s.r.o. and other contributors. Use of this source code is governed by the Apache 2.0 license that can be found in the LICENSE file. -->

UAST (Unified Abstract Syntax Tree) is an abstraction layer on the [PSI](architectural_overview/psi_elements.md) of different JVM Languages.

## Motivation

Different JVM Language have its own [PSI](architectural_overview/psi_elements.md) but a lot of IDE features like inspections, gutters, reference injections
work the same way for all these languages.
So to make it possible to write such features that will work for Java, Kotlin, Scala and so on the UAST was introduced.

### When should I use UAST?

If you are going to implement a plugin that should work for all JVM Languages the same way. Good examples are: Spring Framework support, Android Studio,
IDEA Devkit plugin.

### How UAST will help me?

UAST will allow you to read and understand the code regardless of the language it is written in.

### What about the modifying PSI?

UAST is a read-only API. There are experimental [`UastCodeGenerationPlugin`](upsource:///uast/uast-common/src/org/jetbrains/uast/generate/UastCodeGenerationPlugin.kt) and [`JvmElementActionsFactory`](upsource:///java/java-analysis-api/src/com/intellij/lang/jvm/actions/JvmElementActionsFactory.kt) but currently they are not recommended for external usage.

### Which languages are supported?

* Java - full support.
* Kotlin - full support.
* Scala - full support, but not mature yet.
* Groovy - declarations only support, no method bodies supported.

## PSI to UAST conversion

Given that you have a `PsiElement` of the one of supported languages then to get into UAST you should use the [`UastFacade`](upsource:///uast/uast-common/src/org/jetbrains/uast/UastContext.kt) class or [`UastContextKt.toUElement`](upsource:///uast/uast-common/src/org/jetbrains/uast/UastContext.kt) method (`org.jetbrains.uast.toUElement` for Kotlin).

The base element of UAST is a `UElement` and all common base sub-interfaces are listed in the [declarations](upsource:///uast/uast-common/src/org/jetbrains/uast/declarations) and [expressions](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions) directories of the **uast** module.

To convert `PsiElement` to the specific `UElement` type you could use the following approaches:

- for simple conversion: 
```java
UastContextKt.toUElement(element, UCallExpression.class)
```

- for conversion to the one of different options:
```java
UastFacade.INSTANCE.convertElementWithParent(element, new Class[]{UInjectionHost.class, UReferenceExpression.class})
```

- in some cases the one `PsiElement` could represent several `UElement`s. For instance the parameter of a primary constructor in Kotlin is the `UField` and the `UParameter` the same time. So if you need to have all options you could use:
```java
UastFacade.INSTANCE.convertToAlternatives(element, new Class[]{UField.class, UParameter.class}
```

Note: it is always better to convert to the specific type of `UElement` than to convert without type and then to cast to the specific type:
1. Because of performance:`toUElement` with type is fail-fast.
2. Because you could get different results in some cases: conversion with type is more predictable


## UAST to Psi conversion `sourcePsi` and `javaPsi`

Sometimes you could need to get from the `UElement` back to sources of the dedicated language.
For that purpose there is `UElement#sourcrPsi`

> Note: Ancestor-descendant relation may be not preserved between UAST elements and their sourcePsi

## Uast Visitors

