# Swift Compile-Time Values

* Previous pitches/discussions/reviews:
    * Pitch #1: https://forums.swift.org/t/pitch-compile-time-constant-values/53606
    * Pitch #2: https://forums.swift.org/t/pitch-2-build-time-constant-values/56762
    * Proposal from 2022: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0359-build-time-constant-values.md
    * Review thread: https://forums.swift.org/t/se-0359-build-time-constant-values/57562
    * Return for revision: https://forums.swift.org/t/returned-for-revision-se-0359-build-time-constant-values/58976

## **Introduction**

A Swift language feature for requiring certain values to be knowable at compile-time. This is achieved through an attribute, `@const`, constraining properties, function parameters and functions to yield compile-time knowable values. Such information forms a foundation for richer compile-time features in the future, such as extraction and validation of values at compile time.

## Motivation

Compile-time constant values are values that can be known or computed during compilation and are guaranteed to not change after compilation. Use of such values can have many purposes, from enforcing desirable invariants and safety guarantees to enabling users to express arbitrarily-complex compile-time algorithms.

The first step towards building out support for compile-time constructs in Swift is a basic primitives consisting of an attribute to declare function parameters and properties to require being known at compile-time as well as simple propagation/evaluation rules on a fixed set of supported operations on supported types. While this requirement explicitly calls out the compiler as having additional guaranteed information about such declarations, it can be naturally extended into a broader sense of build-time known values - with the compiler being able to inform other tools with type-contextual value information. For an example of the latter, see the “Declarative Package Manifest” motivating use-case below.

## **Proposed Solution**

The proposal is to add a new attribute `@const` that will be used to annotate properties, local variables, function parameters, and function declarations as having an additional requirement to be known at compile time or be able to be evaluated at compile time to a constant value. If a `@const` property or variable is initialized with a runtime value, the compiler will emit an error. The `@const` attribute can also be used to annotate specific expressions which must be compile-time evaluable to a constant value by the compiler. If a `@const` expression or function contain code that the compiler cannot symbolically evaluate at compile-time for all possible constant-value inputs, the compiler will emit an error. Accesses to `@const` variables are done via an addressor function which returns a consistent address of an immutable value.

For example, a `@const` property can provide the compiler and relevant tooling build-time knowledge of a type-specific value:

```
struct DatabaseParams {
  @const static let encoding: String = "utf-8"
  @const static let capacity: Int = 256
}
```

A `@const` parameter provides a guarantee of a build-time-known argument being specified at all function call-sites, allowing future APIs to enforce invariants and provide build-time correctness guarantees and perform compile-time input validation:
```
func acceptingURLString(@const _ url: String)
```

And a `@const static let` protocol property requirement allows protocol authors to enforce and get the benefits of build-time known properties for all conforming types:
```
protocol DatabaseSerializableWithKey {
  @const static let key: String
}
```

For the supported types, `@const`-ness of a value can be propagated to `@const` uses:
```
@const let x: Int = 32
...
@const let y: Int = x
```

Non-`@const` values may be referenced in initializer expressions of a `@const` declaration if the compiler is able to infer their value at compile-time on a best-effort basis.

```
let x: Int = 32
@const let doublex: Int = x * 2
```

`@const` values may also be defined with expressions consisting of literal values of supported types, other `@const` values, and select supported operations on the above:

```
struct DatabaseParams {
  @const static let capacity: Int = 256
}
...
@const let padding: Int = DatabaseParams.capacity * 2 + 1
```

User-defined function declarations  may be annotated with `@const` to indicate that the function computes a value guaranteed to be known at compile-time when all specified inputs/arguments are known at compile-time. For example:

```
func radToDeg(_ radians: Double) @const -> Double {
    return radians * 180 / Double.pi
}
@const let fullCircle = radToReg(2 * Double.pi) // ~360
```

A `@const` initializer expression may contain calls to functions which are not explicitly user-annotated to be `@const`, but can be inferred by the compiler as being compile-time evaluable. Explicit `@const` function annotation serves the role of enabling compiler enforcement of this property as well as an API contract if it is `public`. 

`@const` attribute can also be applied to initializer expressions directly, rather than to the declaration itself:

```
var x: Double = @const radToReg(Double.pi)
```

To have the compiler enforce the property that the initial value of the declared property be computed fully at compile-time.

## **Detailed design**

### Attribute annotations

#### Global variable or Property `@const` attribute

A `static` property on a `struct` or a `class` can be marked with a `@const` attribute to indicate that its value is known at compile-time.

```
@const let specialUpperLimit: Int = 256
struct Foo {
  @const static let title: String = "foo"
}
```

The value of such a property must be default-initialized with a compile-time-known value, unlike a plain `let` property, which can also be assigned a value in the type's initializer. _`@const` properties do not participate in the type's memory layout and their value holds for *all instances* of the type_, similarly to a `static` property but with an added compile-time known immutable value guarantee enforced by the compiler. 

The computed compile-time value of a `@const public` global variable or a property is a _part of the module’s API_ and will be captured in the corresponding declaration appearing in the textual interface of a resilient module.

```
struct Foo {
  // 👍
  @const static let superTitle: String = "Encyclopedia"
  // ❌ error: `title` must be initialized with a const value
  @const static let title: String
  // ❌ error: `subTitle` must be initialized with a const value
  @const static let subTitle: String = bar()
}
```

Similarly to Implicitly-Unwrapped Optionals, the mental model for semantics of this attribute is that it is a flag on the declaration that guarantees that the compiler is able to know its value as shared by all instance of the type.

A `@const let` property declaration will result in a compiler error indicating that the `static` keyword is required for `@const`  properties to indicate their lifetime.

```
@const let padding: Int = 256 // ❌ error: `@const` properties must be `static`
```

#### Parameter `@const` attribute

A function parameter can be marked with a `@const` keyword to indicate that values passed to this parameter at the call-site must be compile-time-known values.

```
func foo(@const input: Int) {...}
```

Passing in a non-compile-time-known value as an argument to `foo` will result in a compilation error:

```
foo(11) // 👍

@const let defaultCount = 256
foo(defaultCount) // 👍

let x: Int = computeRuntimeCount()
foo(x) // ❌ error: 'input' must be initialized with a const value
```

This capability allows expressing API surfaces which require compile-time inputs for which uses of an API in a client compile can be analyzed at build-time. Furthermore, as the scope of `@const` computation expands, library authors will be able to write compile-time input validation code with custom diagnostics. 

#### Protocol `@const` property requirement

A protocol author may require conforming types to default initialize a given property with a compile-time-known value by specifying it as `@const`` let` in the protocol definition. For example:

```
protocol NeedsConstGreeting {
  @const let greeting: String
}
```

Similarly to plain `@const` properties, a conformance-implementing type then shares a single, known, immutable value of this property per-type with the  `static` keyword implied for this property. 

Compile-time guarantees mean that Swift now allows for `let` property requirement in protocol definitions as long as they are `@const`. 

Unlike other property declarations on protocols that require the use of `var` and explicit specification of whether the property must provide a getter `{ get }` or also a setter `{ get set }`, using `var` for build-time-known properties whose values are known to be fixed at runtime is counter-intuitive. Moreover, `@const` implies the lack of a run-time setter and an implicit presence of the value getter.
If a conforming type initializes greeting with something other than a compile-time-known value, a compilation error is produced:

```
struct Foo: NeedsConstGreeting {
  // 👍
  @const let greeting = "Hello, Foo"
}
struct Bar: NeedsConstGreeting {
  // error: ❌ 'greeting' must be initialized with a const value
  @const let greeting = "\(Bool.random ? "Hello" : "Goodbye"), Bar"
}
```

#### Basic Supported Types

The requirement that values of `@const` global values, properties and parameters be known at compile-time restricts the allowable types for such declarations. The current scope of the proposal includes:

* Enum cases with no associated values.
* Certain standard library types that are expressible with literal values.
    * Integer and Floating-Point types (`Int`, `Float`, `Double`, `Half`), `String` (excluding interpolated strings), `Bool`.
* `Vector`s initialized with literal values consisting of `@const` elements of above types.
    * The property/variable must be explicitly `Vector`, with the values inferred to be of  `Array` type currently not supported.
* Tuple literals consisting of the above list items.

This list will expand in the future to include more literal-value kinds or potential new compile-time valued constructs. For more, see **Future Directions**.

#### Values of Function Type

An additional supported type for `@const` global variables and properties is a Function type. `@const` function type properties, like any other `@const` property, are immutable and shared across all instances of the type they belong to. Such declarations must be initialized to contain _a reference to a `@convention(c)` function defined in the same module_.

```
`@convention(c)`
public func testV1() { ... }

struct DiscoverableTestEntries {
  @const let t1: () -> () = testV1
}
```

 `@const` values of a Function type _are not able to participate in `@const` expressions_ or be called from `@const` functions: although they constitute an individual compile-time-known value, in that the value is an immutable reference to a specific function’s address, the exact bit-pattern of such address value at runtime is not determined until link time. `@const` is a guarantee that the address is of a _specific function known to the compiler_, which, by the start of the user’s program, will have the correct expected value that will not require lazy initialization. 

The `@convention(c)` requirement allows the set of possible function values to be currently restricted to the use-case of trivial function pointers, without allowing e.g. closures which capture arbitrary program state. The scope of the kind of function-type values can be `@const` may be expanded in the future.

`@const` values of Function type are designed to allow for dynamic and static inspection of Swift programs with the aim of discovering specific kinds of client-required code. See the **Test Discovery** Motivating example. 

#### User-Defined Types (Frozen structs with “trivial” initializers)

Simple `struct` types with a default initializer (or a “trivial” `public` `@const` initializer) and properties of one of the above supported types can be marked `@const` and have values known at compile-time. `@const` implies `@frozen` with added enforcement of only containing supported-type properties. When used as a `@const` property declaration, such structs, like any other `@const` property, are immutable and shared across all instances of the parent type.

```
@const struct SchemaLayout {
    let padding: Int
    let sizeLimit: Int
}

@const let layout = SchemaLayout(padding: 256, sizeLimit: 4096)
```

When used in a `@const` context (a `@const` expression or an initializer expression for a `@const` declaration), all arguments to the default initializer must be compile-time known values, much like any other declaration references in a `@const` expression.

The compiler can infer that a value of a  `struct`  type which us not explicitly-annotated as `@const` but is otherwise `@frozen` and contains only compile-time-value-supported type properties and has a default or compile-time-evaluable initializer can be inferred as `@const` and used as a compile-time value. 

```
@frozen public struct InstLayout {
    let numOps: Int
    let opCode: Int
    public init(_ numOps: Int, _opCode: Int) { self.numOpts = numOpts; self.opCode = opCode}
}
@const let layout = InstLayout(2, 32)
```

#### `@const` Propagation rules and supported operations

Values of `@const` declared variables or properties can be directly propagated into initializers of other `@const` variables or properties. For example:

```
@const let x = 4
@const let y = x
```

It is also possible to have the compiler infer, in the initializer expression of a `@const` declaration, a reference to a non-`@const` but otherwise known at compile-time variable or property, meaning such variables are not required to be marked `@const`. 

```
static let universalMultiplier = 11
struct Modifier {
  @const let basicModificationConstant = universalMultiplier
}
```

Values declared in imported modules do not expose their value as a part of the module interface unless they are `@const`, meaning the compiler is only able to infer compile-time values of non-`@const` declarations declared in the same module.

The compiler supports compile-time evaluation of expressions consisting of a minimal set of binary expressions on supported types composed of standard library integer and floating-point operators and user-defined `@const` functions (non-recursive).

* Supported compile-time-evaluable integer and floating-point binary operations on `@const` and literal values:
    * add, and, or, mul, sdiv, srem, sub, udiv, urem, xor
    * cmp_eq, cmp_ne, cmp_sle, cmp_slt, cmp_sge, cmp_sgt, cmp_ule, cmp_ult, cmp_uge, cmp_ugt

This proposal’s goal is to establish a baseline compile-time value feature which can grow in scope and functionality over time. See ‘**Future Directions and Roadmap’** section for more details. 

#### `@const` initializer expressions

A variable or a property may be initialized with a `@const`-annotated initializer expression in order to ensure that the initial value is one that is known/evaluated at compile-time, but the initialized entity does not require immutability property. For example, the `@section` attribute, which indicates a requirement to place data of a certain variable into a user-specified binary section, requires static initialization, which means that it requires that the annotated variable’s initializer expression be compile-time evaluable, i.e. that the expression is `@const`. This can also be useful for other scenarios where a compile-time known initial value is required for an otherwise mutable entity. 

```
var seed = @const 2^9 + specialConstant
```

If the expression following the `@const` attribute contains non-compile-time-evaluable values or operations, a compilation error is produced:

```
// 👍
var seed = @const 2^9 + specialConstant

//  error: ❌ @const expression must not contain runtime values
var randomSeed = @const 2^9 + Int.random(in: 0..<6)
```

Initializer expression of a `@const let` declaration is implicitly `@const`. Explicitly marking such initializer expression with the attribute will result in a compilation error diagnostic (with a Fix-It to remove the redundant attribute).

```
@const let x = @const 11 // error: ❌ 'x' initializer is implicitly @const
```

#### Integer generic parameters implicitly `@const`

The Integer Generic Parameters [proposal](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0452-integer-generic-parameters.md) states that such parameters must be integer literal values. This proposal expands on the notion of integer generic parameters, making them implicitly `@const` and allowing the parameter specified to be a `@const` compile-time value of Integer type. For example:

```
struct IntParam<let x: Int> { }
@const let k: Int = 32
let a: IntParam<2> // OK
let b: IntParam<k> // OK, previously an error

```

#### `@const` Functions

A function may be annotated as `@const` to indicate that it is guaranteed to be compile-time evaluated to a returned value in the context of being invoked in a `@const` expression with compile-time-known parameters. `@const` functions can only have `@const`-supported parameter and return value types and their body can only consist of trivial supported operations on these types as described in the above section. Notably, in the current proposal, `@const` functions cannot have control-flow. The scope of supported Swift language constructs inside `@const` functions is meant to increase over time and will include support for control-flow constructs in the future, the ternary operator, and switch expressions.  `@const` functions are called from non-`@const` contexts as normal functions evaluated at runtime. 

```
public func celsiusToFahrenheit(_ degrees: Int) @const -> Int {
  return degrees * (9 / 5) + 32
}
```

If a `@const`-annotated function contains statements which cannot be evaluated at compile-time, a compilation error is produced:

```
//  error: ❌ `somewhatMore` function must be compile-time evaluable
//  note: 'Int.random(in: 0..<2)' value not known at compile-time
public func somewhatMore(_ value: Int) @const -> Int {
  return value + Int.random(in: 0..<2)
}
```

Similarly, passing in a non-`@const` value to a `@const` function results in a compilation error.

```
//  error: ❌ 'celsiusToFahrenheit` expects a compile-time-known argument
//  note: 'Int.random(in: 0..<10)' value not known at compile-time
@const let coolF: Int = celsiusToFahrenheit(Int.random(in: 0..<10))
```

Non-`@const` functions can be inferred to be compile-time evaluable and used in `@const` expression evaluation. Calling a non-`@const` function from a body of a  `@const` function, if said callee is inferred to be compile-time evaluable, results in a compilation warning:

```
@usableFromInline
 func fahrenheitToCelsius(_ degrees: Int)-> Int {
  return (degrees - 32) * (5 / 9)
}

public func coolDownBy5C(_ degreesFahrenheit: Int) @const -> {
  return fahrenheitToCelsius(degreesFahrenheit - 5) // warning: ⚠️ 'fahrenheitToCelsius' us not explicitly `@const` but used in a `@const` public API 
}
```

Calling a non-`@const` function, defined in a different module, from the body of a `public` `@const` function, even if said function is inlinable able to be inferred as compile-time-evaluable, results in a compilation error:

```
import Conversions

public func coolDownBy5C(_ degreesFahrenheit: Int) @const -> {
  return Conversions.fahrenheitToCelsius(degreesFahrenheit - 5) // error: ❌ 'fahrenheitToCelsius' is not explicitly `@const`
}
```

`@const` annotation on a `public` function implies `@inlinable`, since this attribute is a guarantee to clients that this function can be used in `@const` contexts, client compilation must have access to the full body of the function as a part of the module API. Non-`@const` functions called from within a `@const` `public` function must be at least `@usableFromInline`.

#### Participation in the Type System

Swift compile-time values and evaluation of compile-time expressions are implemented at SIL-level and therefore do not participate in the Type System. That is, types cannot be parameterized with *specific* values known at compile-time, and values of compile-time expressions are not known during type-checking. 

SIL provides a natural surface for engineering a general-purpose compile-time evaluation over a subset of the Swift language. Incorporating compile-time values in the type system would require evaluation/folding of expressions as represented by a type-checked AST - an approach of questionable tractability given the format and design goals of Swift’s AST. 

This decision has several implications worth highlighting:

* Since Macros are expanded before source-code is lowered into SIL, Macros cannot receive evaluated compile-time constant values as parameters. Macros are free to take `@const` parameter values and expressions and expand them into other values expected to be compile-time-known.
* Types cannot be parameterized with compile-time values. For example, for `Vector` while the specification of the length generic parameter may benefit from being enforced as `@const`, the value itself is not known to the type system, meaning all `Vector` types are ‘opaque’ w.r.t. the length.
* `#if` expansion cannot be predicated on compile-time boolean values, since `#if` expansion is handled prior to SIL lowering. 

Also, see the ‘Placing `@const` on the declaration type’ section of considered alternatives for a discussion on why this proposal centers on capturing compile-time **values** in a way not captured by the type system itself. i.e. why `@const`-ness does not affect the type of a value in a way that e.g. optionality does. 

#### ABI and Memory Placement and Runtime Initialization

For each `@const` global variable and property, the compiler defines a global symbol for an addressor function which will be used to access the corresponding immutable value. A `@const` protocol requirement would cause the protocol witness table to include an entry for an addressor (rather than for a getter, as it would today). 

If one can be emitted statically, the addressor would then directly return the address of an immutable object, or else memoize the allocation and initialization of that object.

Static initialization of a property or a global variable’s value in memory is not a part of the semantics of the `@const` attribute and will be performed by the compiler for such values whenever possible. If a value is deemed statically-initializable by the compiler then the addressor would then directly return the address of the initialized immutable object; otherwise, the addressor will memoize the allocation and initialization of that object. 

While the supported types outlined in this proposal should all lend themselves to be statically-initialized, the semantic carve-out is meant to allow future expansion of use-cases and semantics of the attribute to apply to values which may not be able to be statically-initialized but require the immutability and interactions in `@const` contexts guarantees. For example, `class` types when Objective-C interoperability is enabled require initialization hooks that call into the Objective-C runtime; pointer types whose values may require relocation, etc.

For static initialization guarantees, see the `@section` placement annotation feature pitch. 

## Motivating Example Use-Cases

* Enforcement of Compile-Time Attribute Parameters

For example, a `@const` version of the [@Clamping](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md#clamping-a-value-within-bounds) property wrapper that requires that lower and upper bounds be compile-time-known values can ensure that the clamp values are fixed and cannot change for different instantiations of wrapped properties which may occur if runtime values are used to specify the bounds:

```swift
@propertyWrapper
struct Clamping<V: Comparable> {
  var value: V
  let min: V
  let max: V

  init(wrappedValue: V, @const min: V, @const max: V) {
    value = wrappedValue
    self.min = min
    self.max = max
    assert(value >= min && value <= max)
  }
  ...
```
It could also allow the compiler to generate more-efficient comparison and clamping code, in the future.

Or imagine a property wrapper that declares a property is to be serialized and that it must be stored/retrieved using a specific string key. `Codable` requires users to provide a `CodingKeys` `enum` boilerplate, relying on the `enum`’s `String` raw values. Alternatively, such key can be specified on the property wrapper itself:

```swift
struct Foo {
  @SpecialSerializationSauce(key: "title") 
  var someSpecialTitleProperty: String
}

@propertyWrapper
struct SpecialSerializationSauce {
  init(@const key: String) {...}
}
```

Having the compiler enforce the compile-time constant property of the `key` parameter eliminates the possibility of an error where a run-time value is specified which can cause serialized data to not be able to be deserialized, for example. 

Enforcing compile-time constant nature of the parameters is also the first step to allowing attribute/library authors to be able to check uses by performing compile-time sanity checking and having the capability to emit custom build-time error messages.

* Enforcement of Non-Failable Initializers

Ergonomics of the recently-pitched [Foundation.URL](https://forums.swift.org/t/foundation-url-improvements/54057) would benefit greatly from the ability to require the string argument to be compile-time constant. With evolving compile-time evaluation facilities, Swift may even gain an ability to perform compile-time validation of such URLs  even though the user may never be able to express a fully compile-time constant `Foundation.URL` type because this type is a part of an ABI-stable SDK. While a type like `StaticString` may be used to require that the argument string must be static, which string is chosen can still be determined at runtime, e.g.:

```swift
URL(Bool.random() ? "https://valid.url.com" : "invalid url . com")
```

* Facilitate Compile-time Extraction of Values

The [Result Builder-based SwiftPM Manifest](https://forums.swift.org/t/pre-pitch-swiftpm-manifest-based-on-result-builders/53457) pre-pitch outlines a proposal for a manifest format that encodes package model/structure using Swift’s type system via Result Builders. Extending the idea to use the builder pattern throughout can result in a declarative specification that exposes the entire package structure to build-time tools, for example:

```swift
let package = Package {
  Modules {
    Executable("MyExecutable", public: true, include: {
        Internal("MyDataModel")
      })
    Library("MyLibrary", public: true, include: {
        Internal("MyDataModel", public: true)
      })
    Library("MyDataModel")
    Library("MyTestUtilities")
    Test("MyExecutableTests", for: "MyExecutable", include: {
        Internal("MyTestUtilities")
        External("SomeModule", from: "some-package") 
      })
    Test("MyLibraryTests", for: "MyLibrary")
  }
  Dependencies {
    SourceControl(at: "https://git-service.com/foo/some-package", upToNextMajor: "1.0.0")
  } 
}
```

A key property of this specification is that all the information required to know how to build this package is encoded using compile-time-known concepts: types and literal (and therefore compile-time-known) values. This means that for a category of simple packages where such expression of the package’s model is possible, the manifest does not need to be executed in a sandbox by the Package Manager - the required information can be extracted at manifest *build* time.

To *ensure* build-time extractability of the relevant manifest structure, a form of the above API can be provided that guarantees the compile-time known properties. For example, the following snippet can guarantee the ability to extract complete required knowledge at build time:

```swift
Test("MyExecutableTests", for: "MyExecutable", include: {
        Internal("MyTestUtilities")
        External("SomeModule", from: "some-package") 
      })
```
By providing a specialized version of the relevant types (`Test`, `Internal`, `External`) that rely on parameters relevant to extracting the package structure being `const`:

```swift
struct Test {
  init(@const _ title: String, @const for: String, @DependencyBuilder include: ...) {...} 
}
struct Internal {
  init(@const _ title: String)
}
struct External {
  init(@const _ title: String, @const from: String)
}
```
This could, in theory, allow SwiftPM to build such packages without executing their manifest. Some packages, of course, could still require run-time (execution at package build-time) Swift constructs. More-generally, providing the possibility of declarative APIs that can express build-time-knowable abstractions can both eliminate (in some cases) the need for code execution - reducing the security surface area - and allow for further novel use-cases of Swift’s DSL capabilities (e.g. build-time extractable database schema, etc.). 

#### Enabling Static or Dynamic Code Discovery

Suppose a testing framework relies on a Macro annotation to mark user-defined testing methods.

```
@DiscoverableTest
func myTest1() { ... }
@DiscoverableTest 
func myTest2() { ... }
```

One possible implementation for the testing library is to be able to perform dynamic lookup of such tests in the loaded/linked binary of the user program. A possible implementation would have the `@DiscoverableTest` macro expand to generate a peer declaration for each test with a compile-time Function Value placed into a pre-determined binary section. For example:

```
@DiscoverableTest
func myTest1() { ... } // --> Expands to add Peer Declaration -->

@section("__DATA", "my_discoverable_test_hooks")
@const let __t1: (String, () -> ()) = ("myTest1", myTest1)
```

Similarly, with this approach, a static test discovery mechanism could enumerate all user-defined tests by examining the program binary and scanning the requisite section. 

## **Source compatibility**

This proposal is purely additive (adding new attributes), without impact on source compatibility.

## **Effect on ABI stability and API resilience**

The new `@const` attribute does not affect name mangling. The value of `public` `@const` properties and globals is a part of a module's API. See discussion on **Memory Placement** for details.

## **Future Directions and Roadmap**

It is a goal of this proposal to establish a baseline feature with a clear direction for expanding the scope of compile-time computation possible in Swift in the future. The end state of these changes will:

* Make compile-time programming convenient, clear at the point of use, and powerful-enough to handle common use-cases.
    * Compile-time generation of immutable data used by computer programs in-code, instead of maintaining large tables.
    * Moving a class of possible errors from run-time to compile-time.
        * Compile-time input validation on an API  (`#static_assert`)
    * Allow to express in-code zero-cost constructs that make the program more human-readable and maintainable.
        * `@const let GiB = 1024 * 1024 * 1024`
    * Allow for enforcement of compile-time knowability of values with specific static-initialization and memory placement requirements. 

The introduction and implementation of a robust compile-time programming system will span multiple Swift release. In the first phase, the feature, as outlined in this proposal, will establish the baseline language constructs required for declaring simple compile-time values, computed from literal values and simple binary operators on supported types; and, specifying compile-time value requirements on an API surface. 

Followup work to improve the breadth and power of compile-time constructs in Swift beyond what’s outlined in this pitch will include:

#### Expanding the set of supported types for values that can be declared `@const` and participate in compile-time computation via compile-time operators on these types or `@const` expressions/functions. 

* Support for some KeyPath expressions in `@const` expressions. 
* Enums with associated values of supported types
* Generic `@const` functions
* Developing an approach for a compile-time hash map equivalent to Swift’s `Dictionary` type, potentially with an ability to generate a perfect hashing function to obviate the need for today’s required runtime mechanisms. 
* `@const let` properties in generic types, where `static let` properties are not supported.
* Further expansion of user-defined `@const` type support.
    * Allowing user-defined type `@const` values to participate in `@const` expressions. 

#### Expanding the scope of the Swift language surface that is usable in compile-time computation.

* `@const` local variables
    * Values which are used/useful at compile-time for the purposes of writing expressive compile-time expressions and functions.
* Basic control flow operations: `if` conditionals, loops, with the corresponding requirements on the condition values and loop range values being known at compile-time.
* Ternary operator
* `@const switch`  expressions
* Allowing for String Interpolation in Strings to be valid for `@const` Strings when the interpolated values are known at compile-time.
* Expanding the set of available standard library functionality.
* More of various StdLib operations, e.g. Trigonometric functions.
* Making available a wider set of the  `Vector` API surface, e.g. `.count`.
* Transforming runtime fault statements (e.g. `assert`, `fatalError` etc.) into compile-time error diagnostics

#### Toolchain support for extracting compile-time values at build time.

The current proposal covers an attribute that allows clients to build an API surface that is capable of carrying semantic build-time information that may be very useful to build-time tooling, such as SwiftPM plugins. The next step towards this goal would include toolchain support for tooling that extracts such information in a client-agnostic fashion so that it can be adopted equally by use-cases like the manifest example in **Facilitate Compile-time Extraction of Values** and others.


#### Non-`static` `@const` properties

Although `@const` properties do not participate in the type’s memory layout and their value is fixed for all instances of the type, they are not required to be `static`. This means that a compile-time-known property can be expressed, conceptually, as an instance property, without the cost of one. For example, the following are equivalent with respect to generated code:

```
struct HailStorm {
  @const static let designationLabel: String = "Severe"
}
@const let x = HailStorm.designationLabel
```

and

```
struct HailStorm {
  @const let designationLabel: String = "Severe"
}
let storm1 = HailStorm()
@const let x = storm1.designationLabel
```

Where in the latter example, the author wishes accesses to the `@const` property to be modeled as belonging to instances 

## **Alternatives Considered**

### Placing @const on the declaration type

One alternative to declaring compile-time known values as proposed here with the declaration attribute:

```
@const let x = 11
```

Is to instead shift the annotation to declared property's type:

```
let x: @const Int = 11
```

This shifts the information conveyed to the compiler about this declaration to be carried by the declaration's type. Semantically, this departs from, and widely broadens the scope from what we intend to capture: the knowability of the declared value. Encoding the compile-time property into the type system would force us to reckon with a great deal of complexity and unintended consequences. Consider the following example:

```
typealias CI = @const Int
let x: CI?
```

What is the type of `x`? It appears to be `Optional<@const Int>`, which is not a meaningful or useful type, and the programmer most likely intended to have a `@const Optional`. And although today Implicitly-Unwrapped optional syntax conveys an additional bit of information about the declared value using a syntactic indicator on the declared type, without affecting the declaration's type, the historical context of that feature makes it a poor example to justify requiring consistency with it.

### Alternative attribute names

More-explicit spellings of the attribute's intent were proposed in the form of `@buildTime`/`@compileTime`/`@comptime`/`@compEval`/`@known`, and the use of `const` was also examined as a holdover from its use in C++.

While build-time knowability of values this attribute covers is one of the intended semantic takeaways, the potential use of this attribute for various optimization purposes also lends itself to indicate the additional immutability guarantees on top of a plain `let` (which can be initialized with a dynamic value), as well as capturing the build-time evaluation/knowledge signal. For example, in the case of global variables, thread-safe lazy initialization of `@const` variables may no longer be necessary, in which case the meaning of the term `const` becomes even more explicit.

Similarly with the comparison to C++, where the language uses the term `const` to describe a runtime behaviour concept, rather than convey information about the actual value. The use of the term `const` is more in line with the mathematical meaning of having the value be a defined constant.

### Using a keyword or an introducer instead of an attribute

`@const` being an attribute, as opposed to a keyword or a new introducer (such as `const` instead of `let`), is an approach that is more amenable to applying to a greater variety of constructs in the futures, in addition to property and parameter declarations. In addition, as described in comparison to Implicitly-Unwrapped Optionals above, this attribute does not fundamentally change the behavior of the declaration, rather it restricts its handling by the compiler, similar to `@objc`.

### Difference to `StaticString`-like types

As described in the ‘Enforcement of Non-Failable Initializers’, the key difference to types like `StaticString` that require a literal value is the `@const` attribute's requirement that the exact value be known at compile-time. `StaticString` allows for a runtime selection of multiple compile-time known values.

### `Builtin.*` instead of user-defined functions

Instead of initially using a pre-determined set of standard operations on select supported standard library types expressible by literals, for the purposes of supporting basic `@const` binary expressions, the compile-time evaluation system could rely on strictly the use of explicit `Builtin.*` methods. For example, instead of allowing:

```
@const let x = 11
@const let y = x + 1
```

We could instead require that addition be expressed as:

```
@const let x = 11
@const let y = Builtin.add(x, 1)
```

While largely equivalent, asking the user to spell out the individual operations places an undue burden of writing cumbersome chained builtin function call expressions. With support for a much wider surface of the language being able to be evaluated at compile-time, this proposal instead allows the use of a restricted set of corresponding standard library operators directly.
* * *