# Unconditional Error Messages

* Proposal: [SE-NNNN](NNNN-unconditional-error-messages.md)
* Authors: [Daryle Walker](https://github.com/CTMacUser), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Awaiting review**

*During the review process, add the following fields as needed:*

* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

This proposal adds the `#error` directive, which unconditionally posts a compile-time error.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170605/037271.html), [1a](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170612/037347.html)

## Motivation

A conditional compilation block permits branches of code based on the compilation conditions.

```
#if compilation condition 1
// statements to compile if compilation condition 1 is true
#elseif compilation condition 2
// statements to compile if compilation condition 2 is true
#else
// statements to compile if both compilation conditions are false
#endif
```

A block lets at most one of its branches to be incorporated into the compiled module.  But compilation always occurs; there is no scenario to flag conditions that should never allow compiling.

When suspending work on a source file for a while, marking where you ended work with an unconditional error message would let the compiler remind you where you left off.

## Proposed solution

The solution is to add a new compiler control statement, one that posts a fatal diagnostic during compilation.  It can include a message to aid the user on how to change their code.

It's called `#error`, after the C preprocessor directive that has the same effects.

For example, instead of checking API compatibility at run-time:

```
guard #available(tvOS 1.0, *) else {
    fatalError("We need a TV.")
}

// Do what we're supposed to do....
```

We can move the check to compile-time:

```
#if os(tvOS)
// Do what we're supposed to do....
#else
#error("We need a TV.")
#endif
```

## Detailed design

Add to the "Grammar of a Compiler Control Statement":

> *compiler-control-statement* → *error-directive-statement­*

Add a new section "Grammar of a Error Directive Statement":

> *error-directive-statement* → **#error** **(** *static-string-literal* **)**

The semantics of an error directive statement are: when such a statement is encountered during translation, and it is not in an inactive compilation block branch, compilation is aborted and the directive's string is included in the diagnostic of the directive's source line.

## Source compatibility

This proposal should cause no problems with source compatibility.  Relative to the current grammar, there is an addition of one token: `#error`.  Since #-leading tokens are illegal except for the explicitly defined ones, there is no possibly of token overlap in legacy code.

## Effect on ABI stability

Since the domain of this proposal is all in the compilation phase of creating object code, there is no effect on ABI stability.

## Effect on API resilience

The domain of this proposal, controlling whether translation completes, does not affect any API.

## Alternatives considered

The alternative is to do nothing.  This would mean any checks would stay being delayed into run-time, and calling `fatalError` or similar functions to avoid implementing uncovered compilation cases.

The original C syntax left open tokens as the message after the `#error` token.  I decided to enclose the message as a string to ease parsing.

## Future directions

The original idea for this proposal was a distraction while bringing an adaptation of C++'s `static_assert`, as defined in [its proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1720.html).  The `static_assert` is a condition-checking mechanic between `assert` and `#error`; Swift already has the former and this proposal would bring the latter.  There'd still be no equivalent for `static_assert`.  Another proposal for non-environment compile-time expressions and their use in generic where clauses would relieve most of need for `static_assert`, but there needs to be either a new construct a declaration-level version or `#if`/`#elseif` need to be allowed to access non-environment compile-time expressions.

## Acknowledgements

* [How to Use the C Preprocessor's #error Directive](https://barrgroup.com/embedded-systems/how-to/c-preprocessor-error-directive) by Nigel Jones
