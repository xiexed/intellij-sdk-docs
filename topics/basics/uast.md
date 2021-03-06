[//]: # (title: UAST)

<!-- Copyright 2000-2020 JetBrains s.r.o. and other contributors. Use of this source code is governed by the Apache 2.0 license that can be found in the LICENSE file. -->

UAST (Unified Abstract Syntax Tree) is an abstraction layer on the [PSI](architectural_overview/psi_elements.md) of different JVM Languages,
which provides a unified API for working with common language elements like classes and method declarations, literal values and control flow operators.

## Motivation

Different JVM Language have its own [PSI](architectural_overview/psi_elements.md) but a lot of IDE features like inspections, gutters, reference injections
work the same way for all these languages.
So to make it possible to write such features that will work for Java, Kotlin, Scala and so on the UAST was introduced.

### When should I use UAST?

If you are going to implement a plugin that should work for all JVM Languages the same way. Good examples are: Spring Framework support, Android Studio,
IDEA Devkit plugin.

### How UAST will help me?

UAST will allow you to read and understand the code regardless of the language it is written in. Making you able to have a single implementation of your feature for all languages that support UAST.

### What about the modifying PSI?

UAST is a read-only API. There are experimental [`UastCodeGenerationPlugin`](upsource:///uast/uast-common/src/org/jetbrains/uast/generate/UastCodeGenerationPlugin.kt) and [`JvmElementActionsFactory`](upsource:///java/java-analysis-api/src/com/intellij/lang/jvm/actions/JvmElementActionsFactory.kt) but currently they are not recommended for external usage.

### Which languages are supported?

* Java - full support.
* Kotlin - full support.
* Scala - full support, but not mature yet.
* Groovy - declarations only support, no method bodies supported.


## Working with UAST

The base element of UAST is a `UElement` and all common base sub-interfaces are listed in the [declarations](upsource:///uast/uast-common/src/org/jetbrains/uast/declarations) and [expressions](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions) directories of the **uast** module.

All these sub-interfaces provide methods to get the information about common syntax elements: 
`UClass` - about class declarations, `UIfExpression` about conditional expressions and so on.

### PSI to UAST conversion

How do I get into UAST?

Given that you have a `PsiElement` of the one of supported languages then to get into UAST you should use the [`UastFacade`](upsource:///uast/uast-common/src/org/jetbrains/uast/UastContext.kt) class or [`UastContextKt.toUElement`](upsource:///uast/uast-common/src/org/jetbrains/uast/UastContext.kt) method (`org.jetbrains.uast.toUElement` for Kotlin).

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


### UAST to Psi conversion `sourcePsi` and `javaPsi`

Sometimes you could need to get from the `UElement` back to sources of the dedicated language.
For that purpose there is `UElement#sourcePsi` property which returns the `PsiElement` of the original language.

The `sourcePsi` is a "physical" `PsiElement` and it is mostly useful for getting text ranges in the original file or putting the inspection warning on it.
We encourage you to avoid casting the `sourcePsi` to specific classes, because this means that you've falling back from the UAST abstraction to the language-specific API. 

Also, there is a `UElement#javaPsi` property which returns a "Java-like" `PsiElement`.
It is a "fake" `PsiElement` to make different Jvm languages emulate Java language to keep compatibility with Java-API.
For instance the `MethodReferencesSearch#search` method accepts the `PsiMethod`, but only Java natively provides `PsiMethod`,
but other Jvm languages could provide a "fake" `PsiMethod` from the `UMethod#javaPsi`.

Note that `UElement#javaPsi` is physical for Java only, so please avoid using `javaPsi` for text-ranges or as anchor element for inspection warnings or gutters placement - for that purpose the `sourcePsi` should be used.

In short:

`sourcePsi`:

 * is physical: represents a really existing `PsiElement` in the sources of the original language
 * could be used for highlighting, psi modifications, creating smart-pointers and so on
 * should not be cast unless you know what you are doing (for instance handling the language-specific case)

`javaPsi`:

 * should be used only as a representation of Jvm-visible declarations: `PsiClass`, `PsiMethod`, `PsiField`
   for getting their names, types, parameters and so on or to pass them to methods which accepts Java-psi declarations
 * not guaranteed to be physical: could not really exist in sources.
 * is not modifiable: calling modification methods could throw exceptions for non-Java languages

Note: both `sourcePsi` and `javaPsi` could be [converted](#psi-to-uast-conversion) back to the `UElement`

## Uast Visitors

## Uast performance hints

Uast is not a zero-cost abstraction, [some methods](https://youtrack.jetbrains.com/issue/KT-29856) could be unexpectedly expensive for some languages 
Uast is lazy, so convert only what you need

Use the "apriori list" of types for really hard cases.

## Sharp corners of UAST

### `UastLiteralExpression`

### `sourcePsi` and `javaPsi`, `psi` and UElement as Psi

### Should I use `UMethod` or `PsiMethod`, `UClass` or `PsiClass` ?

### Ancestor-descendant relation may be not preserved between UAST elements and their sourcePsi