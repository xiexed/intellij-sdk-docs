[//]: # (title: UAST - Unified Abstract Syntax Tree)

<!-- Copyright 2000-2021 JetBrains s.r.o. and other contributors. Use of this source code is governed by the Apache 2.0 license that can be found in the LICENSE file. -->

UAST (Unified Abstract Syntax Tree) is an abstraction layer on the [PSI](psi_elements.md) of different JVM languages.
It provides a unified API for working with common language elements like classes and method declarations, literal values, and control flow operators.

## Motivation

Different JVM languages have their own [PSI](psi_elements.md), but many IDE features like inspections, gutter markers, reference injection, and many others 
work the same way for all these languages.

Using UAST allows providing features that will work across all [supported JVM languages](#which-languages-are-supported) using a single implementation.

### When should I use UAST?

For plugins that should work for all JVM languages in the same way.

Some known examples are:
* Spring Framework support
* Android Studio
* Plugin DevKit

### Which languages are supported?

* Java - full support
* Kotlin - full support
* Scala - full support, but not mature yet
* Groovy - declarations only, method bodies not supported

### What about modifying PSI?

UAST is a read-only API. 
There are experimental [`UastCodeGenerationPlugin`](upsource:///uast/uast-common/src/org/jetbrains/uast/generate/UastCodeGenerationPlugin.kt) and [`JvmElementActionsFactory`](upsource:///java/java-analysis-api/src/com/intellij/lang/jvm/actions/JvmElementActionsFactory.kt) classes, but they are currently not recommended for external usage.
               

## Working with UAST

The base element of UAST is [`UElement.kt`](upsource:///uast/uast-common/src/org/jetbrains/uast/baseElements/UElement.kt). 
All common base sub-interfaces are located in the [declarations](upsource:///uast/uast-common/src/org/jetbrains/uast/declarations) and [expressions](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions) directories of the **uast** module.

All these sub-interfaces provide methods to get the information about common syntax elements: 
[`UClass.kt`](upsource:///uast/uast-common/src/org/jetbrains/uast/declarations/UClass.kt) about class declarations, [`UIfExpression.kt`](upsource:///uast/uast-common/src/org/jetbrains/uast/controlStructures/UIfExpression.kt) about conditional expressions, and so on.

### PSI to UAST Conversion

To obtain UAST for given `PsiElement` of one of supported languages, use [`UastFacade`](upsource:///uast/uast-common/src/org/jetbrains/uast/UastContext.kt) class or [`UastContextKt.toUElement()`](upsource:///uast/uast-common/src/org/jetbrains/uast/UastContext.kt) method (`org.jetbrains.uast.toUElement` for Kotlin).

To convert `PsiElement` to the specific `UElement`, use one of the following approaches:

- for simple conversion: 

```java
  UastContextKt.toUElement(element, UCallExpression.class)
```

- for conversion to one of different given options:

```java
  UastFacade.INSTANCE.convertElementWithParent(element, new Class[]{UInjectionHost.class, UReferenceExpression.class})
```

- in some cases, `PsiElement` could represent several `UElement`s. For instance, the parameter of a primary constructor in Kotlin is `UField` and `UParameter` at the same time. 
  When needing all options, use:

```java
  UastFacade.INSTANCE.convertToAlternatives(element, new Class[]{UField.class, UParameter.class}
```

 > It is always better to convert to the specific type of `UElement`, rather than to convert without type and then cast to the specific type:
 > * Because of performance: `toUElement()` with type is fail-fast
 > * Because of possibly getting different results in some cases: conversion with type is more predictable
  >
  {type="note"}

### UAST to PSI Conversion

Sometimes it's required to get from the `UElement` back to sources of the underlying language.
For that purpose, `UElement#sourcePsi` property returns the corresponding `PsiElement` of the original language.

The `sourcePsi` is a "physical" `PsiElement`, and it is mostly used for getting text ranges in the original file (e.g., for highlighting).
Avoid casting the `sourcePsi` to specific classes because it means falling back from the UAST abstraction to the language-specific PSI. 
Some `UElement` are "virtual" and thus do not have `sourcePsi`. 
For some `UElement`, the `sourcePsi` could be different from the element from which the `UElement` was obtained.

Also, there is a `UElement#javaPsi` property that returns a "Java-like" `PsiElement`.
It is a "fake" `PsiElement` to make different JVM languages emulate Java language to keep compatibility with Java-API.
For instance, when calling `MethodReferencesSearch.search(PsiMethod)`, only Java natively provides `PsiMethod`;  
other JVM languages thus provide a "fake" `PsiMethod` via `UMethod#javaPsi`.

Note that `UElement#javaPsi` is physical for Java only. Thus `UElement#sourcePsi` should be used to obtain text-range or an anchor element for inspection warnings/gutter marker placement.

In short:

`sourcePsi`:

 * is physical: represents a real existing `PsiElement` in the sources of the original language
 * can be used for highlighting, PSI modifications, creating smart-pointers, etc.
 * should not be cast unless absolutely required (for instance, handling a language-specific case)

`javaPsi`:

 * should be used only as a representation of JVM-visible declarations: `PsiClass`, `PsiMethod`, `PsiField`
   for getting their names, types, parameters, etc., or to pass them to methods that accept Java-PSI declarations
 * not guaranteed to be physical: could not exist in sources
 * is not modifiable: calling modification methods could throw exceptions for non-Java languages

Note: both `sourcePsi` and `javaPsi` can be [converted](#psi-to-uast-conversion) back to the `UElement`.

## UAST Visitors

In UAST there is no unified way to get _children_ of the `UElement` (though it is possible to get its parent via `UElement#uastParent`).
Thus, the only way to walk the UAST as a tree is passing the
[`UastVisitor`](upsource:///uast/uast-common/src/org/jetbrains/uast/visitor/UastVisitor.kt) to `UElement.accept()` method.

Note: there is a convention in UAST-visitors that a visitor will not be passed to children if `visit*()` returns `true`.
Otherwise, `UastVisitor` will continue the walk into depth.

`UastVisitor` can be converted to [`PsiElementVisitor`](upsource:///platform/core-api/src/com/intellij/psi/PsiElementVisitor.java) using [`UastVisitorAdapter`](upsource:///java/java-analysis-api/src/com/intellij/uast/UastVisitorAdapter.java)
or [`UastHintedVisitorAdapter`](upsource:///java/java-analysis-api/src/com/intellij/uast/UastHintedVisitorAdapter.kt).
The latter is preferable as it offers better performance and more predictable results.

As a general rule, it's recommended to abstain from using `UastVisitor`: if you don't need to process many `UElement`s of different types and if the structure of elements is not very important, then it is better to walk the PSI-tree using `PsiElementVisitor` and [convert](#psi-to-uast-conversion) each `PsiElement` to its corresponding UAST explicitly via `UastContext.toUElement()`.

## UAST Performance Hints

UAST is not a zero-cost abstraction: [some methods](https://youtrack.jetbrains.com/issue/KT-29856) could be unexpectedly expensive for some languages,
so be careful with optimizations because it could yield the opposite effect.

[Converting](#psi-to-uast-conversion) to `UElement` also could require resolve for some languages in some cases, again, possibly unexpectedly expensive.
Converting to UAST should be performed only when necessary. 
For instance, converting the whole `PsiFile` to `UFile` and then walk it solely to collect `UMethod` declarations is inefficient.
Instead, walk the `PsiFile` and convert each encountered matching element to `UMethod` explicitly.

UAST is lazy when you pass visitors to `UElement.accept()` or getting `UElement#uastParent`.

For really hard performance optimisation consider using `UastLanguagePlugin.getPossiblePsiSourceTypes()` to pre-filter `PsiElement`s before converting them to UAST.

## UAST Caveats

### `ULiteralExpression` should not be used for strings

[`ULiteralExpression`](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions/ULiteralExpression.kt) represents 
literal values like numbers, booleans, and string. 
Although string values are also literals, `ULiteralExpression` is not very handy to work with them.
For instance, it doesn't handle Kotlin string interpolations.
To process string literals when evaluating their value or to perform language injection, use [`UInjectionHost`](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions/UInjectionHost.kt) instead.

### `sourcePsi` and `javaPsi`, `psi` and `UElement` as PSI

For historical reasons, the relations between `UElement` and `PsiElement` are complicated.
Some `UElement`s implement `PsiElement`; for instance, `UMethod` implements `PsiMethod`.
It is strongly discouraged to use `UElement` as `PsiElement`, and Plugin DevKit provides a corresponding inspection (_Plugin DevKit \| Code \| UElement as PsiElement usage_).
This _"implements"_ is considered deprecated and might be removed in the future.

Also, there is `UElement#psi` property; it returns the same element as `javaPsi` or the `sourcePsi`.
As it is hard to guess what will be returned, it is also deprecated.

Thus `sourcePsi` and `javaPsi` should be the only ways to obtain `PsiElement` from `UElement`. See the [corresponding section](#uast-to-psi-conversion).

### Should I use `UMethod` or `PsiMethod`, `UClass` or `PsiClass` ?

UAST provides a unified way to represent JVM compatible declarations via `UMethod`, `UField`, `UClass`, and so on.
But at the same time, all JVM language plugins implement `PsiMethod`, `PsiClass`, and so on to be compatible with Java.
These implementations could be [obtained](#uast-to-psi-conversion) via `UElement#javaPsi` property.

So the question is: "What should I use to represent the Java-declaration in my code?".
The answer is: We encourage using `PsiMethod`, `PsiClass` as common interfaces for Java-declarations regardless of the JVM language
and discourage exposing the UAST interfaces in the API.

Note: for method bodies, there are no such alternatives, so exposing, for instance, the `UExpression` is not discouraged.
Still, consider exposing the raw `PsiElement` instead.

### UAST/PSI Tree Structure Mismatch

UAST is an abstraction level on top of PSI of different languages and tries to build a unified tree.
It leads to the fact that the tree structure could seriously diverge between UAST and original language,
so no ancestor-descendant relation preserving is guaranteed.

For instance, results of
                                                                     
```kotlin
      generateSequence(uElement, UElement::uastParent).mapNotNull { it.sourcePsi }
      generateSequence(uElement.sourcePsi) { it.parent }
```

could be different, not only in the number of elements, but also in their order.
