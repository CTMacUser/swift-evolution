# Arbitrary Compiler Constant Expressions

* Proposal: [SE-NNNN](NNNN-compiler-constant-expressions.md)
* Authors: [Daryle Walker](https://github.com/CTMacUser), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

This proposal adds compile-time constant expressions, and how to generate them, to the language

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170612/037348.html)

## Motivation

Conditional compilation blocks can shape the design of code based on criteria known at compile time.  Generic where clauses do the same for limiting which generics can get instantiated.  The criteria for either is extremely limited; Boolean combinations of environment state for conditional compilation blocks, and type equivalence, inheritance, or conformance for generic where clauses.

Other languages, like C++, allow values computed from specially-marked ("`constexpr`") objects and functions.  Adding this to Swift permits user-side generated tests today, and is a first step to non-Boolean compile-time values, like value-based generic parameters or static-sized array extents.

## Proposed solution

1. Define a new declaration attribute that can mark off data and code as being compatible with compile-time expressions (CTE).  The definitions of the marked items may have restrictions to ensure realization at compile-time.
2. Designate parts of certain constructs that they can only take CTEs.  Expressions outside of those constructs are evaluated at run-time as usual.
3. Specify the compile-time realization engine.
4. Speculate on what parts of the existing standard library should be updated to support CTEs.

(I have no concrete examples here.  But I do think about if we had something like C++'s type-traits library, then we could replace direct type-testing generic where clauses with ones that check the desired traits from our type-traits equivalent structure.)

## Detailed design

### Computation

The language to process the Swift source file for CTEs before compilation is... the compiler itself!  The Swift compiler can translate Swift as a compiled language or an interpreted language.  CTEs are computed in an interpreted pass of the source code.  If the translation is a Swift interpretation, then the translation goes over the CTE directly instead of a second interpreter.

There cannot be any recursion.  If the CTE is within a statement or part of a generic where clause, then the CTE cannot refer either directly or indirectly to the function its modifying.  If the CTE is within an initializer, then it cannot refer to the object being initialized.  A fatal diagnostic shall be emitted.  One shall also be emitted if CTE evaluation reaches a non-returning function, an uncaught exception, or dereferenced nil optional or similar.

Floating-point operations of the interpreter do not have to exactly match the ones from the compiler.  The interpreter will make its best interpolation to a value supported by the compiler.

The directly-referred symbols of a CTE (the ones seen at the CTE-accepting construct) can only be constant declarations and/or routines that do not have `inout` parameters or `mutating` `self`.  Constants have to be literals, top-level objects, or type-level properties.  A type-level variable computed property can be used if it's getter-only.

Complier implementations may place restrictions on the complexity, resource usage, time taken, etc., of CTE evaluation.  (Put reasonable minimal limits here.)

### Attribute

The `define` attribute shall indicate its owning declaration can be included in CTEs.  The corresponding definition must be available through the source code; the attribute is ignored for items with their realization only as binary code.

Such items still have their run-time realization, as if the attribute was never assigned.  The attribute can be applied to these declarations:

* constant
* variable
* function (including operator definitions and closures)
* initializer
* subscript

Constants and variables must be of a CTE-compatible type.  Routines must have parameters, returns, and `self` (if applicable) of a CTE-compatible type.  Declarations within a protocol declaration cannot have the attribute, but ones defined in an extension or conforming type can.  For methods in an extension for a protocol, the attribute is ignored if the conforming type's implementation methods cannot support it.

A compile-time expression can be of any type except:

* One without any CTE-compatible initializers
* Tuple types with any non-CTE members
* Static-sized array types (if added) with a non-CTE element type
* `class` types
* `struct` types with instance-level stored properties of a non-CTE type
* Raw-value `enum` types with a non-CTE implementation type
* Union `enum` types with a non-CTE type as an associated value in at least one case

Types can have non-CTE (type-level and/or computed) properties and/or methods with non-CTE parameters/returns; they just can't be accessed/called during a CTE context.

### Minimum Symbol Set Needed to Be CTE-Certified

* Operators and any other feasible operations associated with the default Boolean, string/character, and numeric types
* The variables and methods of `MemoryLayout`
* (Probably other stuff I haven't considered)

### CTE Clients

CTE can only be used (for now) in conditional compilation blocks and generic where clauses.  Since the former currently uses a custom hard-coded Boolean calculus, CTEs will be inserted with a wrapping syntax so the whole dynamic doesn't have to be converted at once.

For both conditional compilation blocks and generic where clauses, the CTE has to evaluate to a Boolean-compatible type.

Add to the "Grammar of a Conditional Compilation Block":

> *compilation-condition* → **inline** **(** *compilation-expression* **)**

> *compilation-expression* → *expression*

Add to the "Grammar of a Generic Parameter Clause":

> *requirement* → *compilation-expression*

## Source compatibility

The changes in this proposal should be additive, so existing source should work as-is.

## Effect on ABI stability

CTE computation happens at compile-time, so there is no direct effect on the ABI.  Generic types and functions from the standard library may change their availability if they take on additional generic where clauses with CTE-based tests.

## Effect on API resilience

The API would not be affected by this proposal, modulo any changes future standard library authors make to take advantage of complex tests.

## Alternatives considered

The main alternative is to do nothing.  That would keep conditional compilation blocks and generic where clauses as-is.  Value-based generic arguments couldn't be implemented and static-sized arrays would use integer literals and/or custom syntax for their extents list.
