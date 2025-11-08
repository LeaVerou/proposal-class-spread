# Class Spread Syntax

<details open>
<summary><strong>Table of contents</strong></summary>

1. [Status](#status)
2. [Overview](#overview)
3. [Motivation](#motivation)
4. [Use cases](#use-cases)
	1. [Class modularization](#class-modularization)
	2. [Certain mixin use cases](#certain-mixin-use-cases)
	3. [Supporting both a procedural and OOP API](#supporting-both-a-procedural-and-oop-api)
	4. [Dynamically generating API surface](#dynamically-generating-api-surface)
5. [Limitations](#limitations)
6. [Detailed design](#detailed-design)
	1. [Spreading a class into another class definition](#spreading-a-class-into-another-class-definition)
	2. [Spreading an object into a class definition](#spreading-an-object-into-a-class-definition)
7. [Comparison](#comparison)
	1. [Other languages](#other-languages)
	2. [Other proposals](#other-proposals)
8. [Implementation](#implementation)
9. [Discussion / Q \& A](#discussion--q--a)
	1. [Lexical vs dynamic `super`](#lexical-vs-dynamic-super)
	2. [What should happen with private fields?](#what-should-happen-with-private-fields)
	3. [Is it possible to introspect whether a given class has been spread onto another?](#is-it-possible-to-introspect-whether-a-given-class-has-been-spread-onto-another)
	4. [Can built-in classes be spread?](#can-built-in-classes-be-spread)

</details>

## Status

Champion(s): Lea Verou

Author(s): Lea Verou

Stage: 0


## Overview

This proposal introduces body-order-sensitive composition for classes through a spread syntax inspired from object spread syntax.
Mixin application is treated as a class element, applied in the order it appears, interleaved with method and field definitions.

```js
import * as methods from "./methods.js";

class A {
	foo() { console.log("foo"); }
	static bar = "bar";
}

class C {
	...A;
	...methods;
}

console.log(B.bar); // "bar"
(new B).foo(); // "foo"
```

It is meant as a modularization and code reuse tool for the more tightly coupled use cases, rather than a more general solution to all class composition use cases.

## Motivation

Currently, the only mechanisms in the language for reusing class API surface are
1. Good ol' single inheritance
2. [Subclass factories](https://github.com/tc39/proposal-mixins) to emulate multiple inheritance by piggybacking on single inheritance
3. Good ol' low-level prototype _frobnication_. 😅

However, the inheritance chain is observable from the outside and thus, **inheritance-based methods** can feel heavyweight and/or backwards incompatible, especially for minor code reuse tasks not intrinsically related to the class identity.

**Low-level prototype _frobnication_** can be done transparently, but has awkward ergonomics (needs to be a separate step), and does not have access to all class features (e.g. class fields).
Additionally, having to be done imperatively, after the class definition, means that it cannot be interleaved with other methods — if the host class author wants to override a method, that _also_ needs to be done imperatively.

**Static initialization blocks** help a little with ergonomics, but are executed after any methods have been defined, so any methods they add cannot be overridden by the host class author:

```js
class A {
	foo() { return 1 }
	static {
		this.prototype.foo = function() { return 2 }
	}
	foo() { return 3 }
}

console.log(new A().foo()); // 2
```

As discussed in the [comparison with decorators](#decorators) section below, when class decorators are primarily meant to modify classes and class members, rather than to compose classes from building blocks. While they _can_ be used for this purpose, they simply provide better ergonomics over these two patterns, rather than a new mechanism to compose classes from building blocks.

In other areas of the language, the **spread syntax** can be used to compose an object from multiple other objects of the same or similar type.
We have it for iterables:

```js
const arr = [...arr1, ...arr2];
```

Function arguments:

```js
console.log(...iterable);
```

And for objects:

```js
const obj = { ...obj1, ...obj2 };
```

But we don't have it for **classes**.

This proposal introduces body-order-sensitive composition for classes: mixin application is treated as a class element, applied in the order it appears, interleaved with method and field definitions.
Later elements (whether defined directly in the class or introduced via composition) override earlier ones.

<!-- This makes it hard to abstract class behavior out into separate modules,
even in scenarios where class definitions and such modules can be cooperatively developed.

Lack of a mechanism for spreading within the class body means that the only way to add API surface dynamically is to go back to dealing with prototypes.

Last, even if [`[[Fields]]` remain non-introspectable ](customizable-fields.md),
a specialized class spread syntax could have access to them and take care of copying them over,
something that userland code cannot do today. -->

## Use cases

### Class modularization

For large classes, it is impractical to maintain the entire class API in a single file.
However, the ergonomics of modularization are not great, and require a lot of glue code:


`methods/foo.js`:
```js
export function foo(...args) {
	let arg = this ?? args.shift();
	/* elided */
}
```
`methods/bar.js`:
```js
export function bar() {
	let arg = this ?? args.shift();
	/* elided */
}
```

`class.js`:
```js
import {foo} from "./methods/foo.js";
import {bar} from "./methods/bar.js";

export default class Class {
	foo(...args) {
		return foo.call(this, ...args);
	}
	bar(...args) {
		return bar.call(this, ...args);
	}
	// ...
}
```

With regular JS, one can avoid maintaining the API in two places, by using `...args` as shown above.
However with typed variants of the language (e.g. TypeScript), there is no way to avoid duplicating the function signatures.

With a spread syntax, it could look like this:

`class.js`:
```js
import * as methods from "./methods/index.js";

export default class Class {
	...methods;
}
```

### Certain mixin use cases

Given that class spread essentially has macro semantics, it’s suitable for mixin use cases where the mixin identities applied to a given class do not need to be preserved or introspected, but the separation mainly exists for maintainability purposes.

In some ways it’s analogous to the difference between object spread (which does not preserve where each property came from) vs `Object.create()` (which does).

> TODO: Add more concrete use cases.

### Supporting both a procedural and OOP API

There are use cases where a module wants to provide both a procedural API and an OOP one.
E.g. [Color.js](https://www.npmjs.com/package/colorjs.io) does this.
The procedural API can be better suited to performance-critical tasks and is more tree-shakable,
while still allowing the OOP API to offer better ergonomics by preserving state across method calls.

`procedural.js`:
```js
export function foo(...args) {
	let arg = this ?? args.shift();
	/* elided */
}
export function bar() {
	let arg = this ?? args.shift();
	/* elided */
}
// ...
```

`oop.js`:
```js
import * as methods from "./procedural.js";

export default class Class {
	...methods;
}
```

### Dynamically generating API surface

Note that in the previous case, every method had to include the following boilerplate to cater to being used as either a procedural function, or an instance method:

```js
let arg = this ?? args.shift();
```

We could also write the functions as procedural and *generate* the boilerplate when adding to the class:

```js
export default class Class {
	...Object.fromEntries(Object.entries(methods).map(([key, value]) => [key, (...args) => value(this, this, ...args)]))
}
```

#### Facilitating API glue code for the delegation pattern

A common OOP pattern is to achieve [composition via delegation](https://en.wikipedia.org/wiki/Delegation_pattern) (or [forwarding](https://en.wikipedia.org/wiki/Forwarding_(object-oriented_programming))), where the implementing class uses a separate object to abstract and reuse behavior.

In Web Components, this pattern is known as **Controllers**.
The native [`ElementInternals` API](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals) is an example of this pattern.
Lit even has a [Controller primitive](https://lit.dev/docs/composition/controllers/) to facilitate this pattern.

There is a lot to like in this pattern:
- Separate state and inheritance chain makes it very easy to reason about
- Can be added and removed at any time, even on individual instances
- Can have multiple controllers of the same type for a single class
- Delegate does not need to be built for this purpose. E.g. in many objects the delegate is simply another object (a DOM element in Web Components, a data structure, etc.)

However, a major problem is that adding API surface to the host class involves *a lot of repetitive glue code*.
To reuse the `ElementInternals` example, making a web component behave like a form control involves glue code like:


For example, making a custom element that is also a form control looks like this:
```js
class MyElement extends HTMLElement {
	// This tells ElementInternals that this element is form associated
	static formAssociated = true;

	constructor() {
		super();

		// Cannot use #internals because subclasses need access
		this._internals = this.attachInternals();

		this.addEventListener("input", () => {
			this._internals.setFormValue(this.value);
		});
	}

	// API glue code
	get labels () { return this._internals.labels; }
	get form () { return this._internals.form; }
	get validity () { return this._internals.validity; }
	get validationMessage () { return this._internals.validationMessage; }
	willValidate (...args) { return this._internals.willValidate(...args); }
	reportValidity(...args) { return this._internals.reportValidity(...args); }
	checkValidity(...args) { return this._internals.checkValidity(...args); }
	// it goes on, and on, and on…
}
```

With a programmatic way to add API surface, it could look like this:

```js
const formProperties = [
	'labels', 'form', 'validity', 'willValidate',
	//...
];
class MyElement extends HTMLElement {
	...formProperties.reduce((acc, prop) => Object.defineProperty(acc, prop, { get: () => this._internals[prop] }), {});
}
```


## Limitations

**This is not intended to cover every mixin/trait/multiple inheritance use case.**
It is a low-level feature, intended to make tightly certain coupled use cases easier to manage.
Spreading arbitrary classes onto arbitrary classes will often produce unexpected results and/or errors.

A few limitations that make it unsuitable for some use cases:
- There is no way to introspect whether a given class has been spread onto another
- There is no way to compose functions (e.g. to add side effects to lifecycle hooks)
- There is no declarative mechanism to handle naming collisions gracefully (e.g. via renaming)
- `super` remains lexically bound, which can be surprising
- Referencing private fields will produce errors since they are not spread

For example:

```js
class A {
	#foo = 1;
	get foo() { return this.#foo }
}
class B { ...A }

const b = new B();
console.log(b.foo);
// TypeError: Cannot read private member #foo from an object whose class did not declare it
```

## Detailed design

Like object spread, the semantics are largely those of assignment.
Unlike object spread, copying is done by copying *descriptors*, not *values*.

### Spreading a class into another class definition

Spreading a class copies (descriptors for):
- Instance members
- Static members
- All public [[Fields]]

It does _not_ copy:
- Any fields or members in [[PrivateElements]]
- Static initialization blocks (but it does copy their side effects, since by then they have already executed)

> [!NOTE]
> What should happen with decorators?

It does not affect the class's [[SourceText]], which includes the spread syntax itself.

`super` remains lexically bound, akin to regular assignment (see discussion below).

#### Alternative model: syntactic expansion

An alternative model would be that of syntactic expansion, where we baseically use [[SourceText]] as-is and simply remove everything that is not in [ClassBody](https://tc39.es/ecma262/multipage/ecmascript-language-functions-and-classes.html#prod-ClassBody).

Then `super` would resolve dynamically, we could copy private fields, declarations, etc.
On the other hand, references would not be preserved.

While this seems like it would match author intent more closely, there is no precedent in the language for such a model, and that is not how spread syntax works in any other area of the language.

### Spreading an object into a class definition

Spreading an object copies the object's own descriptors onto the constructor `[[Prototype]]`.
While the copying is generally shallow, it does descend into the `constructor` property, if present.
This allows objects to be constructed such that they add both instance and static members.

There is no way to specify class fields through spreading an object.

> [!NOTE]
> Should there be? E.g. through a known `Symbol` property?

## Comparison

### Other languages

See [prior art](../../prior-art.md) for a discussion on current userland patterns and related features in other languages.

To my knowledge, no existing mainstream language provides this symmetric, body-order-based override behavior for class composition.

This is similar in spirit to Ruby’s `include`/`prepend`, where:
- mixin operations occur inside the class body,
- the order of these operations affects the method lookup chain, and
- class bodies are executed imperatively rather than being purely declarative.

However, Ruby’s precedence rules are asymmetric and not fully order-local:
- Methods defined on the class always override methods from included modules, regardless of where include appears in the body.
- Methods from prepended modules always take precedence over the class’s methods, again regardless of body position.

By contrast, this proposal defines composition in terms of source order of class elements, consistent with the way spread syntax works in other areas of the language.

More languages support mixins-as-macros (e.g. Common Lisp, Nim, D, and Rust — sort of), though in those cases the semantics are purely those of syntactic expansion.

### Other proposals

See [Class Composition: Existing proposals](https://github.com/webplatformco/project-class-composition/tree/main?tab=readme-ov-file#existing-proposals) for a discussion on other proposals related to class composition.

Here we discuss them in the context of this proposal.

#### [Maximally minimal mixins](https://github.com/tc39/proposal-mixins) (Subclass factories)

As discussed in [Motivation](#motivation), subclass factories are a prevalent pattern for class composition, but affect the inheritance chain, which is not always desirable.

#### [Decorators](https://github.com/tc39/proposal-decorators)

Conceptually, decorators are meant as a way to modify a class or class member, rather than as a way to compose classes from other classes or objects.
However, it _is_ possible to use decorators in that way: one can define a class decorator that also takes one or more classes or objects as parameters and extends the class with them.

However:
- Decorators are not interleaved with other class elements, but are executed after all other class elements have been defined, so there is no elegant way to override a method or field defined by a decorator.
- The only ways to add new API surface to the existing class are either to (a) use `addInitializer()` and basically do the same things as prototype fudging, with the same pros & cons OR (b) return a new subclass of the class being decorated, with the same pros & cons as subclass factories.

#### [Class field introspection](https://github.com/leaverou/proposal-class-field-introspection/)

The [class field introspection proposal](https://github.com/leaverou/proposal-class-field-introspection/) is a proposal for exposing a read-only data structure containing class fields.

It is complementary to this proposal: without it, class fields are spread by directly taking them from [[Fields]], whereas with it, it can be used to explain and polyfill class spread syntax.

## Implementation

Spreading a class into another class definition could desugar to something like:

```js
const instanceDescriptors = Object.getOwnPropertyDescriptors(A.prototype);
for (const key in instanceDescriptors) {
	if (key === "constructor") continue;
	Object.defineProperty(B.prototype, key, instanceDescriptors[key]);
}
const staticDescriptors = Object.getOwnPropertyDescriptors(A);
for (const key in staticDescriptors) {
	if (["length", "name", "prototype"].includes(key)) continue;
	Object.defineProperty(B, key, staticDescriptors[key]);
}
```

There is no way to copy or mutate [[Fields]] in userland, but assuming they were [introspectable](../class-fields-introspection) AND mutable, it could look like:

```js
B[Symbol.publicFields].push(...A[Symbol.publicFields]);
```

Note that all of this can be applied to an existing class as well.
Perhaps there could be a helper method to facilitate this, e.g. `Function.extend(Base, ...traits)`,
with the spread syntax desugaring to it.


## Discussion / Q & A

### Lexical vs dynamic `super`

It could be argued that ideally, `super` should resolve based on the new class hierarchy,
but given that `super` is lexically bound, that ship has likely sailed.

It's probably far simpler to have `super` resolve using the same semantics as assignment:

```js
class A {
	foo() { return "A"; }
}
class Trait extends A {
	foo() { return super.foo() + " Trait"; }
}
class B extends A {
	foo() { return "B"; }
}
class C extends B {
	// Same as C.prototype.foo = Trait.prototype.foo
	...Trait;
}

console.log(new C().foo()); // "A Trait"

```

In some ways, this is more predictable.
This is not necessarily a footgun, it can still be useful for cases where the spread class shares some of the same inheritance chain.

When the Trait needs to reference the host class's superclass dynamically, rather than its own superclass, that can still be done with a little more work:

```js
class Trait extends A {
	foo() {
		let thisSuper = Object.getPrototypeOf(this.constructor)?.prototype;
		return thisSuper.foo.call(this) + " Trait";
	}
}

class C extends B {
	...Trait;
}

console.log(new C().foo()); // "B Trait"
```

That said, use cases that need dynamic superclasses might be better suited to subclass factories.

### What should happen with private fields?

Ideally, private fields would be copied as well, and any references to them would resolve based on the host class's corresponding private field.

While at first it seems like that would violate encapsulation, it does not reveal anything that is not already revealed by simply accessing the class's [[SourceText]].

However, it seems like that may be harder to implement, especially if the semantics are not those of syntactic expansion.

### Is it possible to introspect whether a given class has been spread onto another?

By design, no (except through a shared contract, e.g. a `Symbol` property).

References are preserved, but there is no way to disentangle whether that is the result of spreading or regular assignment.

### Can built-in classes be spread?

Yes, per the definition of how spreading works, though that is of limited utility in most cases.
