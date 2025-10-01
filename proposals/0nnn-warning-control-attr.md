# Source-Level Control Over Compiler Warnings

* Proposal: [SE-0nnn](0nnn-warning-control-attr.md)
* Authors: [Artem Chikin](https://github.com/artemcm), [Doug Gregor](https://github.com/douggregor), [Holly Borla](https://github.com/hborla)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [swiftlang/swift#NNNNN](https://github.com/swiftlang/swift/pull/NNNNN)
* Experimental Feature Flag: `SourceWarningControl`

## Introduction

This proposal introduces a new declaration attribute for controlling compiler warning behavior in specific code regions: to be emitted as warnings, errors, or suppressed.

## Motivation

[SE-0443](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0443-warning-control-flags.md) introduced control over compiler warnings with command-line flags which control behaviors of specific warning groups. Module-level controls are a blunt instrument, applying the same behavior to all code regardless of whether it's appropriate everywhere, making it desirable to provide a source-level control mechanism which can be used to refine default or module-wide behaviors. Furthermore, [SE-0443](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0443-warning-control-flags.md) review identified module-wide warning *suppression* as problematic; however, suppression of warnings in some circumstances remains a desired use-case, one which would be well-suited to a fine-grained source-level application. 

Source-level warning control addresses the practical reality of codebases undergoing migration or incremental adoption of new Swift features. During deprecation cycles, teams must maintain compatibility with older APIs in specific functions or declarations while enforcing strict usage elsewhere. Similarly, when adopting stricter warning policies, such as `-warnings-as-errors`, teams may encounter legitimate edge cases or temporary technical debt that necessitate warning suppression or deescalation in isolated scopes without compromising module-wide policy.

As Swift evolves to include more advanced static analyses and stricter language modes (such as strict memory safety and explicit ownership checking) that enforce certain practices and patterns through diagnostics, their deployment would greatly benefit from fine-grained warning control: it would enable the use of diagnostics to guide developers toward safer patterns while allowing certain warnings to be temporarily suppressed or escalated during adoption or in specific implementation contexts.

Declaration-level warning behavior controls provide the precision needed for these scenarios, allowing developers to document exceptions exactly where they occur, and allowing stricter language modes to be incrementally adopted by annotating exceptional cases while ensuring the diagnostic rules remain enforced throughout the majority of the codebase.

## Proposed solution

This proposal introduces a new declaration attribute to the language which will allow the behavior of warnings to be controlled in the lexical scope of the annotated declaration, for a specific diagnostic group.

```swift
@warn(Deprecate, as: error)
public func foo() {
  ...
}
```

Within the scope of the `foo` declaration in this example, the effect of the attribute is equivalent to `-Werror Deprecate`, but without affecting diagnostic behavior on any other source code in the current module. The attribute supports 3 `as:` behavior specifiers: `error`, `warning`, and `ignored`. The attribute also supports an optional `reason:` parameter which accepts a string literal. With `error` and `warning` behavior specifiers, the attribute has the effect equivalent to that of `-Werror <groupID>` and `-Wwarning <groupID>`, respectively, without affecting other source code in the module. With the `ignored` behavior specifier, warnings belonging to the diagnostic group `<groupID>` are fully suppressed (akin to a global (module-wide) `-Wsuppress`, which does not exist). 

The `@warn` attribute overrides diagnostic behavior for the specified diagnostic group, relative to its *enclosing scope* - with command-line behavior specifiers representing global (module-wide) scope: `-warnings-as-errors`, `-Werror`, `-Wwarning`. 

For example, on a compilation which specifies either `-warnings-as-errors`  or  `-Werror FruitTooDelicious`, the `@warn` attribute can be used to lower the severity of `FruitTooDelicious` diagnostics to a warning within the scope of a specific method, while the same code pattern elsewhere in the module would be emitted as an error:

```swift
let bestFruit = 🍏 // 🟥 error: 🍏s may be too delicious for this program [#FruitTooDelicious]

@warn(FruitTooDelicious, as: warning)
func testFunc() {
    let snack = 🍏 // 🟨 warning: 🍏s may be too delicious for this program [#FruitTooDelicious]
}
```

Diagnostic behavior can be refined further by modifying the severity of diagnostics belonging to the same group in a nested declaration:

```swift
let bestFruit = 🍏 // 🟥 error: 🍏s may be too delicious for this program [#FruitTooDelicious]

@warn(FruitTooDelicious, as: warning)
func testFunc() {
    let snack = 🍏 // 🟨 warning: 🍏s may be too delicious for this program [#FruitTooDelicious]

    @warn(FruitTooDelicious, as: ignored)
    func ageusiaFunc() {
        let treat = 🍏 // No diagnostic is emitted
        
        @warn(FruitTooDelicious, as: error)
        func chefFunc() {
            let saladIngredient = 🍏 // 🟥 error: 🍏s may be too delicious for this program [#FruitTooDelicious]
        }
    }
}
```

Furthermore, the `@warn` attribute can be used to restrict behavior of diagnostic **sub**groups. For example, for the subgroup `CitrusTooDelicious` of the `FruitTooDelicious` group, module-wide control (`-Werror FruitTooDelicious`) of the latter can be refined with a fine-grained scoped attribute for the former:

```swift
@warn(CitrusTooDelicious, as: warning)
func testFunc() {
    let snack1 = 🍊 // 🟨 warning: 🍊s may be too delicious for this program [#CitrusTooDelicious]
    let snack2 = 🍏 // 🟥 error: 🍏s may be too delicious for this program [#FruitTooDelicious]
}
```

 `import` statements can generate various warnings related to deprecation, cross-import overlays, and `import` access control violations. The `@warn` attribute can be used for fine-grained control over which import-related warnings should be treated as errors, warnings, or temporarily ignored.

## Detailed design

### @warn attribute on declarations

A `@warn` attribute's argument list must have at least two arguments: a diagnostic group identifier in the first position, and a diagnostic behavior specifier in the second position of a parameter labelled `as:`, supporting arguments `error`, `warning`, `ignored`. The attribute may have a third, optional string literal argument in the third position of a parameter labelled `reason:`.  The reason argument must not have any string interpolation.

```swift
@warn(<groupID>, as: <behavior>, reason: <explanation>)
```

This attribute only affects warning diagnostics belonging to the specified `<groupID>` diagnostic group. 
Compilation **error** diagnostics, even when belonging to a specified diagnostic group, cannot be controlled by either diagnostic control compiler options or the `@warn` attribute.

The `@warn` attribute can be applied on:

* *`function-declaration`*, *`initializer-declaration`, `deinitializer-declaration`, `subscript-declaration`*, *`getter-clause`*, *`setter-clause`*, computed property declaration (with a *`code-block`)*
    Setting behavior of all warning diagnostics belonging to the indicated group in the ***lexical scope*** of the body of the corresponding declaration.
* `*enum-declaration*`*, `struct-declaration`, `extension-declaration`,  `class-declaration`, `actor-declaration`, `protocol-declaration`*
    Setting behavior of all warning diagnostics belonging to the indicated group in the ***lexical scope*** of the declaration, affecting all declarations contained within.
* `*import-declaration*`
    Setting behavior of all warning diagnostics emitted on the import statement itself.

```swift
// Import statement
@warn(Deprecate, as: ignored)
import bar

// Function declaration
@warn(Deprecate, as: ignored, reason: "Proposal Example")
func foo() {...}

// Initializer and Deinitializer
struct Foo {
    @warn(Deprecate, as: ignored)
    init() {...}
    @warn(Deprecate, as: ignored)
    deinit() {...}
}

// Subscript and Operator
struct FooCollection<T> {
    @warn(Deprecate, as: ignored)
    `subscript``(``index``:`` ``Int``)`` ``->`` T ``{...}`
`    ``@warn``(``Deprecate``,`` ``as``:`` ignored``)`
`    ``static`` func ``+++`` ``(``lhs``:`` T``,`` rhs``:`` T``)`` ``->`` T`` ``{...}`
}

// Getter and Setter
struct Foo {
    var property: Int {
        @warn(Deprecate, as: ignored)
        get {...}
        @warn(Deprecate, as: ignored)
        set {...}
    }
}

// Enum
@warn(Deprecate, as: ignored)
enum Foo {...}

// Struct 
@warn(Deprecate, as: ignored)
struct Foo {...}

// Class
@warn(Deprecate, as: ignored)
class Foo {...}

// Extension
@warn(Deprecate, as: ignored)
extension Foo {...}

// Actor
@warn(Deprecate, as: ignored)
actor Foo {...}

// Protocol
@warn(Deprecate, as: ignored)
protocol Foo {...}
```

### Interaction with compiler options and evaluation order

For a given warning diagnostic group, its global (module-wide) behavior is defined by the diagnostic’s default behavior and  [compiler option evaluation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0443-warning-control-flags.md#compiler-options-evaluation). A warning diagnostic’s default behavior is either to be always emitted or fully suppressed - where default-suppressed warnings can be enabled with `-Wwarning` and `-Werror`.

At the top-level file scope, a `@warn` attribute overrides the specified diagnostic group’s global behavior within the lexical scope of the declaration it is applied to.  For example, for a globally-escalated (with `-Werror groupID`) diagnostic group, `@warn(groupID, as: warning)` top-level function declaration defines the behavior of `groupID`  warnings within the lexical scope of the function’s body.

A `@warn` attribute applied to a declaration at a nested lexical scope overrides the specified diagnostic group’s behavior as defined for the parent lexical scope, either by the global behavior, or a `@warn` attribute applied to the parent scope declaration. For example, `@warn` attribute for diagnostic group `groupID` applied to a method definition in a `struct` declaration may override the diagnostic group’s global behavior as configured with compiler flags, or it may override the diagnostic group’s behavior as defined by a `@warn` attribute on the containing `struct`’s declaration. 

#### Multiple `@warn` attributes on the same declaration

More than one `@warn` attribute on a given declaration for the same diagnostic group is not valid and results in an error.

```swift
@warn(Deprecate, as: error)
@warn(Deprecate, as: warning) // 🟥 error: multiple conflicting `@warn` attributes for group `Deprecate`.
public func foo()
```

Multiple `@warn` attributes on a given declaration for nested diagnostic groups are evaluated in the order they are specified in source. For example:

```swift
@warn(FruitTooDelicious, as: error)
@warn(CitrusTooDelicious, as: ignored)
public func foo()
```

Where `CitrusTooDelicious` is a subgroup of  `FruitTooDelicious`, this annotation will first apply the `error` behavior for the broader parent group, and then apply the `ignored` behavior to the sub-group. In the opposite case:

```swift
@warn(CitrusTooDelicious, as: ignored)
@warn(FruitTooDelicious, as: warning) // 🟨 warning: `warning` diagnostic behavior for `FruitTooDelicious` overrides prior attribute for `CitrusTooDelicios` `ignored` behavior
public func foo()
```

The second attribute completely overrides the the first attribute’s diagnostic severity directive, with a corresponding compiler warning. 

### Effect on the public interface

This attribute is not emitted into textual module interfaces and does not affect emitted binary module contents.

## Source compatibility

This proposal is purely additive and has no effect on source compatibility.

## ABI compatibility

This proposal has no effect on ABI compatibility.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Future directions

### Local lexical scope control

It may be desirable to provide a mechanism for an even finer-grained warning behavior control within lexical scopes which do not correspond to a declaration, such as a `do {}` block. For example, one can imagine extending the file-level `using @warn` mechanism to allow its use anywhere in a given lexical scope, affecting warnings emitted anywhere in entirety of the lexical scope (or strictly the code which follows the `using` statement within said lexical scope). This proposal focuses on providing such control at the granularity of a declaration, leaving this direction for future consideration.

### *`closure-expression`*

It may be desirable to provide a mechanism to control the behavior of warning diagnostics emitted in the body of a specific closure. Such a mechanism would need to be carefully weighed against any possible extension to this proposal for even finer-grained control as closures represent an intermediate granularity between full declarations and arbitrary code blocks.


### File-scope warning behavior control

A complementary proposal (TODO: insert link here) of the file-scope `using <attribute>` syntax further expands on this capability by allowing the use of `using @warn(<group>, as: <behavior>)` to define file-scope warning behavior for a given diagnostic group, but is out of scope of this proposal.

## Alternatives considered

Clang and other C-family compilers use `#pragma` directives to control diagnostic behavior within arbitrary regions of code:

```c++
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
...
#pragma clang diagnostic pop
```

While this mechanism is flexible, it is not particularly elegant and fitting a modern language like Swift. 

* Swift generally favors attaching metadata and behavior to declarations, making developer intent clear and self-documenting. 
*  Asking the developer to define the “end” of a region means requiring careful manual state management - forgetting or misplacing a region delimiter could lead to complex unintended behaviors when multiple scopes are overlapping and potentially affect nested diagnostic groups.

## Acknowledgments

Allan Shortlidge and Kavon Farvardin and Aviral Goel for their input on the attribute design and use cases.

