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
We encourage you to avoid casting the `sourcePsi` to specific classes, because this means that you've falling back from the UAST abstraction to the language-specific API. Some `UElement` are "virtual" and doesn't have `sourcePsi`. For some `UElement` the `sourcePsi` could be different to element
from which the `UElement` was obtained.

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

In Uast there is no unified way to get _children_ of the `UElement` 
(though it is possible to get parent via `UElement#uastParent`). Thus, the only way to walk the Uast as a tree is passing the
[`UastVisitor`](upsource:///uast/uast-common/src/org/jetbrains/uast/visitor/UastVisitor.kt) to the `UElement#accept` method.

Note: there is a convention in UAST-visitors that a visitor will not be passed to children if `visit*` will return `true`,
otherwise `UastVisitor` will continue the walk into depth.

Also you can convert the `UastVisitor` to `PsiElementVisitor` using the [`UastVisitorAdapter`](upsource:///java/java-analysis-api/src/com/intellij/uast/UastVisitorAdapter.java)
or [`UastHintedVisitorAdapter`](upsource:///java/java-analysis-api/src/com/intellij/uast/UastHintedVisitorAdapter.java).
The latter is preferable because it shows better performance and more predictable results.

As a general rule we're recommending to abstain from using `UastVisitor` if you don't need to process a lot of `UElement`s of different type.
Usually if the structure of elements is not very important to you it is better to walk the PSI-tree with `PsiElementVisitor` and [convert](#psi-to-uast-conversion) each PsiElement to Uast
separately via `UastContext.toUElement`.

## Uast performance hints

Uast is not a zero-cost abstraction, [some methods](https://youtrack.jetbrains.com/issue/KT-29856) could be unexpectedly expensive for some languages,
so be careful with optimizations because you could get the opposite.

[Converting](#psi-to-uast-conversion) to `UElement` also could require resolve for some languages in some cases and also could appear unexpectedly expensive. So don't convert to Uast without real necessity. 
For instance converting the whole `PsiFile` to `UFile` and the walk it just to collect `UMethod` declarations - is a bad idea.
Instead, you could walk the `PsiFile` and convert to `UMethod` each element separately.

Uast is lazy when you pass visitors to the `UElement#accept` or getting `UElement#uastParent`,
so convert only what you need, and it will save a lot of CPU and Memory.

For really hard perforance optimisation consider using the `UastLanguagePlugin#getPossiblePsiSourceTypes` method 
to pre-filter `PsiElement`s before converting them to Uast.

## Sharp corners of UAST

### `ULiteralExpression` should not be used for strings

[`ULiteralExpression`](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions/ULiteralExpression.kt) is a `UElement` for representing
literal values like numbers, booleans and string. 
Although, string values are also literals but `ULiteralExpression` is not very handy to work with them.
For instance, it doesn't handle Kotlin string interpolations.
If you need to process string literals to evaluate the value or to perform the language injection consider using the 
[`UInjectionHost`](upsource:///uast/uast-common/src/org/jetbrains/uast/expressions/UInjectionHost.kt) element.

### `sourcePsi` and `javaPsi`, `psi` and UElement as Psi

For historical reasons the relations between `UElement` and `PsiElement` are complicated.
Some `UElement`-s implement the `PsiElement`, for instance `UMethod` implements `PsiMethod`.
Currently, we strongly discourage using the `UElement` as `PsiElement` and the Devkit provides an inspection for that.
This _"implements"_ is deprecated, and we hope we get rid of it one day.

Also, there is `UElement#psi` property, it returns the same element as `javaPsi` or the `sourcePsi`,
because it is actually hard to guess what will be returned now it is deprecated.

Thus `sourcePsi` and `javaPsi` should be the only ways to get `PsiElement` from `UElement`. See the [corresponding section](#uast-to-psi-conversion-sourcepsi-and-javapsi).

### Should I use `UMethod` or `PsiMethod`, `UClass` or `PsiClass` ?

Uast provides a unified way to represent JVM compatible declarations via `UMethod`, `UField`, `UClass` and so on.
But the same time all Jvm language plugins somehow implement `PsiMethod`, `PsiClass` and so on to be compatible with Java.
These implementations could be [obtained](#uast-to-psi-conversion-sourcepsi-and-javapsi) via `UElement#javaPsi` property.

So the question is: "What should I use to represent the Java-declaration in my code?".
The answer is: We encourage using `PsiMethod`, `PsiClass` as common interfaces for Java-declarations regarless of the Jvm language
and discourage exposing the Uast interfaces in the API.

Note: for method bodies there is no such alternatives, so we don't discourage exposing for instance the `UExpression`,
but you could still consider exposing the raw `PsiElement` instead.

### Ancestor-descendant relation may be not preserved between UAST elements and their sourcePsi

Uast is an abstraction level on top of PSI of different languages and tries to build a unified tree.
It leads to the fact that tree structure could seriously diverge between UAST and original language,
so no ancestor-descendant relation preserving is guaranteed.
For instance result of the following lines:

      generateSequence(uElement, UElement::uastParent).mapNotNull { it.sourcePsi }
      generateSequence(uElement.sourcePsi) { it.parent }

could be different not only in the number of elements but also in their order.