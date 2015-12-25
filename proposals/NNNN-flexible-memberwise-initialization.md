# Flexible Memberwise Initialization

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-flexible-memberwise-initializers.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Review**
* Review manager: TBD

## Introduction

The Swift compiler is currently able to generate a memberwise initializer for use in some circumstances however there are currently many limitations to this.  This proposal build on the idea of compiler generated memberwise initialization making it available to any initializer that opts in.

Swift-evolution thread: [Proposal Draft: flexible memberwise initialization](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/003902.html)

## Motivation

When designing initializers for a type we are currently faced with the unfortunate fact that the more flexibility we wish to offer users the more boilerplate we are required to write and maintain.  We usually end up with more boilerplate and less flexibility than desired.  There have been various strategies employed to mitigate this problem including:

1. Sometimes properties that should be immutable are made mutable and a potentially unsafe ad-hoc two-phase initialization pattern is employed where an instance is initialized and then configured immediately afterwards.  This allows the developer to avoid including boilerplate in every initializer that would otherwise be required to initialize immutable properties.

2. Sometimes mutable properties that have a sensible default value are simply default-initialized and the same post-initialization configuration strategy is employed when the default value is not correct for the intended use.  This results in an instance which may pass through several states that are incorrect *for the intended use* before it is correctly initialized for its intended use.

Underlying this problem is the fact initialization scales with M x N complexity (M members, N initializers).  We need as much help from the compiler as we can get!

Flexible and concise initialization for both type authors and consumers will encourages using immutability where possible and removes the need for boilerplate from the concerns one must consider when designing the intializers for a type.

Quoting [Chris Lattner](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151130/000518.html):

	The default memberwise initializer behavior of Swift has at least these deficiencies (IMO):
	1) Defining a custom init in a struct disables the memberwise initializer, and there is no easy way to get it back.
	2) Access control + the memberwise init often requires you to implement it yourself.
	3) We don’t get memberwise inits for classes.
	4) var properties with default initializers should have their parameter to the synthesized initializer defaulted.
	5) lazy properties with memberwise initializers have problems (the memberwise init eagerly touches it).

Add to the list “all or nothing”.  The compiler generates the entire initializer and does not help to eliminate boilerplate for any other initializers where it may be desirable to use memberwise intialization for a subset of members and initialize others manually.

It is common to have a type with a number of public members that are intended to be configured by clients, but also with some private state comprising implementation details of the type.  This is especially prevalent in UI code which may expose many properties for configuring visual appearance, etc.  Flexibile memberwise initialization can provide great benefit in these use cases, but it immediately becomes useless if it is "all or nothing".  

We need a flexible solution that can synthesize memberwise initialization for some members while allowing the type auther full control over initialization of implementation details.

## Proposed solution

I propose adding a `memberwise` declaration modifier for initializers which allows them to *opt-in* to synthesis of memberwise initialization.  The programmer model is as straighforward as possible.  

The set of properties that receive memberwise initialization parameters is determined by considering *only* the initializer declaration and the declarations for all properties that are *at least* as visible as the initializer (including any behaviors attached to the properties).  The rules are as follows:

1. Their access level is *at least* as visible as the memberwise initializer.
2. They do not have a behavior which prohibits memberwise initialization.
3. If the property is a `let` property it *may not* have an initial value.

Two additional eligibiliy rules may be added by the `@nomemberwise` enhancement in the future.

The parameters are synthesized in the parameter list in the location of the `...` placeholder.  They are ordered as follows:

1. All parameters **without** default values precede parameters **with** default values.
2. Within each group, superclass properties precede subclass properties.
3. Finally, follow property declaration order.

Under the current proposal only `var` properties could specify a default value, which would be the initial value for that property.   It may be possible for `let` properties to specify a default value in the future using the `@default` enhancement or some other mechanism allowing the default value to be specified.

## Examples

This section of the document contains several examples of the solution in action.  It *does not* cover every possible scenario.  If there are concrete examples you are wondering about please post them to the list.  I will be happy to discuss them and will add any examples we consider important to this section as the discussion progresses.

Specific details on how synthesis is performed are contained in the detailed design.

### Replacing the current memberwise initializer

```swift
struct S {
	let s: String
	let i: Int

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String, i: Int) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```

### Var properties with initial values

NOTE: this example is only possible for `var` properties due to the initialization rules for `let` properties.

```swift
struct S {
	var s: String = "hello"
	var i: Int = 42

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```

### Access control

```swift
struct S {
	let s: String
	private let i: Int

	// user declares:
	memberwise init(...) {
		// compiler error, i memberwise initialization cannot be synthesized 
		// for i because it is less visible than the initializer itself
	}
}
```

```swift
struct S {
	let s: String
	private let i: Int

	// user declares:
	memberwise init(...) {
		i = 42
	}
	// compiler synthesizes (suppressing memberwise initialization for properties with lower visibility):
	init(s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's initializer remains
		i = 42
	}
}
```

### Lazy properties and incompatible behaviors

```swift
struct S {
	let s: String
	lazy var i: Int = InitialValueForI()

	// user declares:
	memberwise init(...) {
	}
	// compiler synthesizes:
	init(s: String) {
		/* synthesized */ self.s = s
		
		// compiler does not synthesize initialization for i 
		// because it contains a behavior that is incompatible with 
		// memberwise initialization
	}
}
```

## Detailed design

### Syntax changes

This proposal introduces three new syntactic elements: the `memberwise` initializer declaration modifier and the `...` memberwise parameter placeholder.

Initializers opt-in to synthesized memberwise initialization with the `memberwise` declaration modifier.  This modifier will cause the compiler to follow the procedure outlined later in the design to synthesize memberwise parameters as well as memberwise initialization code at the beginning of the initializer body.  

### Overview

Throughout this design the term **memberwise initialization parameter** is used to refer to initializer parameters synthesized by the compiler as part of **memberwise initialization synthesis**.

#### Terminology

* **direct memberwise initialization parameters** are parameters which are synthesized by the compiler and initialized directly in the body of the initializer.

* **forwarded memberwise initialization parameters** are parameters which are synthesized by the compiler and provided to another initializer that is called in the body of the initializer.  The current proposal does not contain any such parameters, however they may be introduced in the future if the memberwise initializer chaining enhancement eventually becomes an accepted proposal.

* **synthesized memberwise initialization parameters** or simply *memberwise initialization parameters* is the full set of parameters synthesized by the compiler which includes both direct and forwarded memberwise initialization parameters.


#### Algorithm

1. Determine the set of properties elibile for memberwise initialization synthesis.  Properties are eligible for memberwise initialization synthesis if:

	1. Their access level is *at least* as visible as the memberwise initializer.
	2. They do not have a behavior which prohibits memberwise initialization.
	3. If the property is a `let` property it *may not* have an initial value.

2. Determine the default value, if one exists, for each *memberwise initialization parameter*.  Under the current proposal only `var` properties could specify a default value, which would be the initial value for that property.   It may be possible for `let` properties to specify a default value in the future using the `@default` enhancement or some other mechanism allowing the default value to be specified.
	
3. If the initializer declares any parameters with external labels matching the name of any of the properties eligible for memberwise initialization report a compiler error.  If the initializer needs to declare a parameter with an external label matching the name of a property the property **must** be explicitly excluded from memberwise initialization, possibly via the `@nomemberwise` attribute of the initializer if that enhancement to this proposal is adopted (lower visibility is another possibility).

4. Synthesize *memberwise initialization parameters* in the location where the `...` placeholder was specified.  The synthesized parameters should have external labels matching the property name.  Place the synthesized parameters in the following order:
	1. All parameters **without** default values precede parameters **with** default values.
	2. Within each group, follow property declaration order.
	
5. Synthesize initialization of all *direct memberwise initialization parameters* at the beginning of the initializer body.

## Impact on existing code

This proposal will also support generating an *implicit* memberwise initializer for classes and structs when the following conditions are true:

1. All members have at least `internal` visibility.
2. The type declares no initializers explicitly.
3. The type:
	1. a struct
	2. a root class
	3. a class whose superclass has a designated intializer requiring no arguments

The *implicitly* synthesized initializer will be identical to an initializer declared *explicitly* as follows:

1. For structs and root classes: `memberwise init(...) {}`
2. For derived classes: `memberwise init(...) { super.init() }`

NOTE: If the *implict* initializer is synthesized and the previous declartion is placed in an extension of the type a compiler error will result.

The changes described in this proposal are *alomost* entirely additive.  The only impact on existing code will be in the case of structs with `private` stored properties which had been receiving an implicitly synthesized memberwise initializer.  In this case a mechanical migration could generate the explicit code necessary to declare the previously implicit initializer.

## Future enhancements

In the spirit of incremental change, the current proposal is focused on core functionality.  It is possible to enhance that core functionality with additional features.  These enhancements may be turned into proposals after the current proposal is accepted.

### @default

It is not possible under the current proposal to specify a default value for memberwise initialization parameters of `let` properties.  This is an unfortunate limitation and a solution to this is a highly desired enhancement to the current proposal.

One possible solution would be to introduce the `@default` attribute allowing `let` properties to specify a default value for the parameter the compiler synthesizes in memberwise initializers.

There are two possible syntactic approaches that could be taken by `@default`:

1. Make `@default` a modifier.  The same syntax is used as for initial values, but when the `@default` attribute is specified for a property the specified value is a default rather than an initial value.
2. Allow the default value to be specified using an attribute argument.

Each syntax has advantages and disadvantages:

1. The first syntax is arguably cleaner and more readable.
2. The first syntax makes it impossible to specify both an initial value **and** a default value for the same property.  This is advantageous because a `let` property should never have both and initial values for `var` properties are effectively just a default value anyway.
3. The second syntax may have less potential for confusion and thus more clear as it uses significantly different syntax for specifying initial and default values.

#### Example using the first syntax option

```swift
struct S {
	@default let s: String = "hello"
	@default let i: Int = 42

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```

#### Example using the second syntax option

```swift
struct S {
	@default("hello") let s: String
	@default(42) let i: Int

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```



### @nomemberwise

There may be cases where the author of a type would like to prevent a specific property from participating in memberwise initialization either for all initializers or for a specific initializer.  The `@nomemberwise` attribute for properties and initializers supports this use case.

Memberwise initializers can explicitly prevent memberwise initialization for specific properties by including them in a list of property names provided to the `@nomemberwise` attribute like this: `@nomemberwise(prop1, prop2)`.

Properties will be able to opt-out of memberwise initialization with the `@nomemberwise` attribute.  When they do so they will not be eligible for memberwise initialization synthesis.  Because of this they must be initialized directly with an initial value or initialized directly by every initializer for the type.

The `@nomemberwise` attribute would introduce two additional eligibility rules when deterimining which properties can participtate in memberwise initialization.

1. The property **is not** annotated with the `@nomemberwise` attribute.
2. The property **is not** included in the `@nomemberwise` attribute list attached of the initializer.  If `super` is included in the `@nomemberwise` attribute list **no** superclass properties will participate in memberwise initialization.

This enhancement would also update the rule for subclass designated intializers allowing them to call non-memberwise intializers in the superclass:

Memberwise designated initializers in subclasses **must** *either* call a memberwise initializer in the superclass *or* include `super` in its `@nomemberwise` attribute list as follows: `@nomemberwise(super)`.

#### Examples

NOTE: This example doesn't really save a lot.  Imagine ten properties with only one excluded from memberwise initialization.

```swift
struct S {
	let s: String
	let i: Int

	// user declares:
	@nomemberwise(i)
	memberwise init(...) {
		i = getTheValueForI()
	}
	// compiler synthesizes (suppressing memberwise initialization for properties mentioned in the @nomemberwise attribute):
	init(s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's initializer remains
		i = getTheValueForI()
	}
}
```

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init() {
		// compiler error, i is not initialized
	}
}
```

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init() {
		i = 42
	}
	// compiler synthesizes:
	memberwise init(s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's intializer remains
		i = 42
	}
}
```

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init(configuration: SomeTypeWithAnIntMember, ...) {
		i = configuration.intMember
	}
	// compiler synthesizes (suppressing memberwise intialization for properties with the @nomemberwise attribute):
	init(configuration: SomeTypeWithAnIntMember, s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's initializer remains
		i = configuration.intMember
	}
}
```

### Memberwise initializer chaining

Ideally it would be possible to define memberwise convenience and delegating initializers.  It would also be ideal if it were possible for designated initializers to publish forwarding memberwise parameters and forward the supplied arguments when calling a superclass memberwise initializer.

One requirement for any such feature is that the body of the forwarding initializer must contain an unambiguous call to the forwarded initializer.  There are several other currently unsolved design challenges which must be addressed by the design of any such feature.

Examples of how this might look in user code follow.  The eventual design of a feature supporting this may or may not look somewhat different.

#### Delegating and convenience initializers

NOTE: the call to the chained initializer **must** be unambiguous when only the explitly provided arguments are considered.  However it is allowed to be invalid on its own as long as it becomes valid after forwarding of memberwise arguments is completed (i.e. forwarding can be used to supply memberwise arguments where the parameter *does not* have a default value).

```swift

struct S {
	private let s: String = "hello"
	let i: Int = 42

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	private init(s: String = "hello", i: Int = 42) {
		self.s = s
		self.i = i
	}
	
	// user declares:
	memberwise init(describable: CustomStringConvertible, ...) {
		self.init(s: describable.description, ...)
	}
	// compiler synthesizes (adding forwarded memberwise parameters):
	init(describable: CustomStringConvertible, i: Int = 42) {
		self.init(s: describable.description, i: i)
	}
}
```

#### Subclass designated initializers

NOTE: the call to the superclass initializer **must** be unambiguous when only the explitly provided arguments are considered.  However it is allowed to be invalid on its own as long as it becomes valid after forwarding of superclass memberwise arguments is completed (i.e. forwarding can be used to supply memberwise arguments where the parameter *does not* have a default value).

```swift
class Base {
	let baseProperty: String

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(baseProperty: String) {
		self.baseProperty = baseProperty
		super.init()
	}
}

class Derived: Base {
	let derivedProperty: Int

	// user declares:
	memberwise init(...) { super.init(...) }
	// compiler synthesizes (adding forwarded memberwise parameters):
	init(baseProperty: String, derivedProperty: Int) {
		self.derivedProperry = derivedProperty
		super.init(baseProperty: baseProperty)
	}
}

```


### Objective-C Class Import

Objective-C frameworks are extremely important to (most) Swift developers.  In order to provide the call-site advantages of flexible memberwise initialization to Swift code using Cocoa frameworks a future proposal could recommend introducing a `MEMBERWISE` attribute that can be applied to Objective-C properties and initializers.

Mutable Objective-C properties could be marked with the `MEMBERWISE` attribute.  Readonly Objective-C properties **could not** be marked with the `MEMBERWISE` attribute.  The `MEMBERWISE` attribute should only be used for properties that are initialized with a default value (not a value provided directly by the caller or computed in some way) in **all** of the class's initializers.

Objective-C initializers could also be marked with the `MEMBERWISE` attribute.  When Swift imports an Objective-C initializer marked with this attribute it could allow callers to provide memberwise values for the properties declared in the class that are marked with the `MEMBERWISE` attribute.  At call sites for these initializers the compiler could perform a transformation that results in the memberwise properties being set with the provided value immediately after initialization of the instance completes.

It may also be desirable to allow specific initializers to hide the memberwise parameter for specific properties if necessary.  `NOMEMBERWISE(prop1, prop2)`

It is important to observe that the mechanism for performing memberwise initialization of Objective-C classes (post-initialization setter calls) must be implemented in a different way than native Swift memberwise initialization.  As long as developers are careful in how they annotate Objective-C types this implementation difference should not result in any observable differences to callers.  

The difference in implementation is necessary if we wish to use call-site memberwise initialization syntax in Swift when initializing instances of Cocoa classes.  There have been several threads with ideas for better syntax for initializing members of Cocoa class instances.  I believe memberwise initialization is the *best* way to do this as it allows full configuration of the instance in the initializer call. 

Obviously supporting memberwise initialization with Cocoa classes would require Apple to add the `MEMBERWISE` attribute where appropriate.  A proposal for the Objective-C class import provision is of significantly less value if this did not happen.  My recommendation is that an Objective-C import proposal should be drafted and submitted if this proposal is submitted, but not until the core team is confident that Apple will add the necessary annotations to their frameworks.

## Alternatives considered

### Require stored properties to opt-in to memberwise initialization

This is a reasonable option and and I expect a healthy debate about which default is better.  The decision to require opt-out was made for several reasons:

1. The memberwise initializer for structs does not currently require an annotation for properties to opt-in.  Requiring an annotation for a mechanism designed to supercede that mechanism may be viewed as boilerplate.
2. Stored properties with public visibility are often intialized directly with a value provided by the caller.
3. Stored properties with **less visibility** than a memberwise initializer are not eligible for memberwise initialization.  No annotation is required to indicate that.

I do think a strong argument can be made that it may be **safer** and **more clear** to require an `@memberwise` attribute on stored properties in order to opt-in to memberwise initialization.  I am very interested in community input on this.

### Allow all initializers to participate in memberwise initialization

This option was not seriously considered.  It would impact existing code and it would provide no indication in the declaration of the initializer that the compiler will synthesize additional parameters and perform additional initialization of stored properties in the body of the initializer.

### Require initializers to opt-out of memberwise initialization

This option was also not seriously considered.  It has the same problems as allowing all initializers to participate in memberwise initialization.

### Allow parameters to be synthesized for properties with a lower access level than the initializer

I considered allowing parameters to be synthesized for properties that are not directly visible to callers of the initializer.  There is no direct conflict with the access modifier and it is possible to write such code manually.  I decided against this approach because as it is unlikely to be the right approach most of the time.  In cases where it is the right approach I think it is a good thing to require developers to write this code manually.

### Require initializers to explicitly specify memberwise initialization parameters

The thread "[helpers for initializing properties of the same name as parameters](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151130/000428.html)" discussed an idea for synthesizing property initialization in the body of the initializer while requiring the parameters to be declard explicitly.  

```swift
struct Foo {
    let bar: String
    let bas: Int
    let baz: Double
    init(self.bar: String, self.bas: Int, bax: Int) {
		  // self.bar = bar synthesized by the compiler
		  // self.bas = bas synthesized by the compiler
        self.baz = Double(bax)
    }
}

```

The downside of this approach is that it does not address the M x N scaling issue mentioned in the motivation section.  The manual initialization statements are elided, but the boilerplate parameter declarations still grow at the rate MxN (properties x initializers).  It also does not address forwarding of memberwise initialization parameters which makes it useless for convenience and delegating initializers.

Proponents of this approach believe it provides additional clarity and control over the current proposal.  

Under the current proposal full control is still available.  It requires initializers to opt-in to memberwise initialization.  When full control is necessary an initializer will simply not opt-in to memberwise initialization synthesis.  The boilerplate saved in the examples on the list is relatively minimal and is tolerable in situations where full control of initialization is required.

I believe the `memberwise` declaration modifier on the initializer and the placeholder in the parameter list make it clear that the compiler will synthesize additional parameters.  Furthermore, IDEs and generated documentation will contain the full, synthesized signature of the initializer.  

Finally, this idea is not mutually exclusive with the current proposal.  It could even work in the declaration of a memberwise initializer, so long the corresponding property was made ineligible for memberwise intialization synthesis.
