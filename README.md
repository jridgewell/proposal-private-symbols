# ECMAScript Proposal: Private Symbols

## Overview

This proposal allows for the encapsulation of "private" properties to
lexical environments. These are known as **private symbols** because
they behave similarly to regular symbols.

```js
class Example {
  private #foo = 1;
  get foo() {
    return this.#foo;
  }
  set foo(foo) {
    this.#foo = foo;
  }
}
```

A private symbol is semantically identical to a regular symbol, with the
following exceptions:

- Private symbols are not exposed by `Object.getOwnPropertySymbols`.
- Private symbols are not copied by `Object.assign` or object spread.
- Private symbol-keyed properties are not affected by `Object.freeze`
    and `Object.seal`.
- Private symbols cause proxies to transparently interact with the
    target without consulting trap handlers.

See [Semantics](#semantics) for more details.

## Goals

This proposal is intended to provide a missing capability to JavaScript:
the ability to add properties to an object that are inaccessible without
a unique and unforgeable key. Moreover, this proposal is intended to be
*minimal*, completely *generalized* and strictly *orthogonal*.

- It is *minimal* because it introduces the minimum amount of new
    concepts to the language.
- It is *generalized* because it applies to both classes and regular
    objects.
- It is *orthogonal* because it does not attempt to solve any problem
    other than property encapsulation.

The proposal **does not** attempt to provide solutions for:

- Secure branding mechanisms.
- Static shape guarantees.

These are both easily solvable by using `WeakSet`s.

## Changes

### API
Added new method to global `Symbol` built-in object for creating "private"
symbols:
```js
const privateSymbol = Symbol.private('private symbol');
```
The `Symbol` prototype has a `isPrivate` getter function which can be used
to determine whether the symbol is private:
```js
const commonSymbol = Symbol('common symbol');
assert(commonSymbol.isPrivate === false);

const privateSymbol = Symbol.private('private symbol');
assert(privateSymbol.isPrivate === true);
```

### Syntax
While this API is enough to provide `hard-private` encapsulation for any
particular object and class it's still not ergonomic for day-to-day usage.
This section fulfills such gap, by providing new declaration keyword and
variable type.

#### Variable type
Symbols, while still assignable to common `var`/`let`/`const` variables,
now, in addition to already existing heavily different semantic and
use-cases comparing with traditional variables, has its own special
syntax - `#name`.
**EVERY** variable that starts with `#`-prefix is `Symbol` **ALWAYS**.
> **NOTE:** for now it's always `Symbol.private`, but it's still subject
to discuss in committee.

The main benefit of such approach is more expressive property access, which
was kind of pain point of ES from very beginning (especially after
introducing `Symbol`). Consider following code:
```js
console.log(obj[x]);
```
Without additional context you can't understand what this code does just
by reading it. Is it array element access, or dynamic property access, or
`Symboled` property access? In order to understand that you need to have
more context, at least `x` variable declaration. And if `x` isn't constant
declared with `const` keyword, you also have to check which values it might
have by searching for all assignments to it.  
In order to avoid most of misreadings, there are some best practices, like
using `i`/`j`/`k`/`l` (and some other short names) for iterating in array,
`someSpecificElementNumber` or `someSpecificElementIndex` to access specific
element from array, `fieldName`/`fieldProperty` for dynamic property access
and `somePropertySymbol` for `Symbol`ed property access.  
`#`-prefixed variables will simplify this quiz, because
```js
console.log(obj[#x]);
```
and 
```js
console.log(obj.#x);
```
**ALWAYS** means accessing `#x` `Symbol`ed property of `obj`.  
`#x` is always `Symbol`, it even can't be `null` or `undefined`. If it
isn't declared before usage, early `ReferenceError` will be thrown
(in same way as it works for `let`/`const` variables, including TDZ
and shadowing).

In classes (assuming [class-fields-proposal](https://github.com/tc39/proposal-class-fields)) and object literals it could be used in 2 ways: using existing `computed property syntax` like:
```js
class A {
  [#x] = 1;
}
const obj = {
  [#x]: 1,
}
```
or using fully-equivalent shorthand syntax:
```js
class A {
  #x = 1;
}
const obj = {
  #x: 1,
}
```

#### Declaration keyword
In order to declare `#`-prefixed constant, new keyword is introduced -
`private`. It's backwards compatible addition, because this keyword is
already reserved.  
It is allowed to use `private` keyword everywhere, where `const` is allowwed.
And it behaves the same way as `const` declaration (including TDZ
and shadowing), except this small list:  
1. `private` keyword **MUST** be followed by `#`-prefixed name
    ```js
    private #x; // correct
    private x; // throws syntax error
    private [#y]; // throws syntax error
    private [y]; // throws syntax error
    ```
2. `private` declaration **MUST NOT** be followed by assignment
    ```js
    private #x; // correct
    private #y = 1; // throws syntax error
    ```

`private #x` declares `Symbol.private('#x')` that later could be
accessed using `#x`constant.

In addition to all above, `private` keyword could be used inside
`object literals` and `class declarations`, using following syntax:
```js
class A {
  private #x = 1;
}
const obj = {
  private #x: 1,
}
```
In this case two operations performed: declaring lexically
scoped private symbol `#x` and defining corresponding property with
`#x` key for object or class. To clarify, following code is fully equivalent
to sample above:
```js
const A = (() => {
  private #x;
  return class A {
    #x = 1;
  }
})();
const obj = (() => {
  private #x;
  return {
    #x: 1,
  }
})();
```

## Some Questions and Answers

### __*Do objects inherit private symbols from their prototypes?*__

Yes.

Private symbols work just like regular symbols when it comes to
prototype lookup. There is nothing new to learn. This means that
"private methods" on classes just work, without any additional
semantics.

```js
const instance = new Example();
instance.foo;
// => 1

const obj = Object.create(instance)
obj.foo;
// => 1
```

### __*Why don't `Object.assign` and the spread operator copy private symbols?*__

It follows from the definition of `Object.assign` and the spread
operator.

They both work by obtaining a list of property keys from the source
object, using the `[[OwnPropertyKeys]]` internal method. Since we do not
want to allow private symbols to leak, we restrict the definition of
`[[OwnPropertyKeys]]` such that it is not allowed to return private
symbols.

This is exactly the semantics used by the current private fields
proposal.

### __*Why don't `Object.freeze` and `Object.seal` affect private symbol-keyed properties?*__

When an object is frozen or sealed, the object is first marked as
non-extensible (meaning new properties cannot be added to it) and then a
list of property keys is obtained by calling the object's
`[[OwnPropertyKeys]]` internal method. That list is then used to mark
properties as non-configurable (and non-writable in the case of
`Object.freeze`).

Since `[[OwnPropertyKeys]]` is not allowed to return private symbols,
`freeze` and `seal` cannot modify any property definitions that are
keyed with private symbols.

The fundamental idea is that only the code that has access to the
private symbol is allowed to make changes to properties keyed by that
symbol.

This requires us to slightly modify the definition of a "frozen" object:
An object is "frozen" if it is non-configurable and all of it's own
*non-private* properties are non-configurable and non-writable.

This is exactly the semantics used by the current private fields
proposal.

### __*How does this work with Proxies?*__

Proxies are not able to intercept private symbols, and proxy handlers
are not allowed to return any private symbols from the `ownKeys` trap.

For all of the proxy internal methods that accept a property key, if
that property key is a private symbol, then the proxy handler is not
consulted and the operation is forwarded directly to the target object,
as if there were no handler defined for that trap.

### __*Can private symbols be used for branding?*__

The purpose of a branding mechanism is to mark objects such that, when
presented with an arbitrary object, the code that created the "brand"
can determine whether or not the object has been marked. In a typical
scenario, objects are branded by constructor functions so that method
invocations can check whether the `this` value is an object that was
actually created by the constructor function.

While `WeakSet`/`WeakMap`s are a natural solution to this, there is an
option to use `Symbol.private` for same purpose.

```js
class Example {
  private #brand;
  private #foo = 1;

  constructor() {
    this.#brand = this;
  }

  equal(obj) {
    Example.#check(this, obj);
    return this.#foo === obj.#foo;
  }

  static private #check(...objects) {
    for (const obj of objects) {
      if (obj !== obj.#brand) {
        throw new TypeError('non instance');
      }
    }
  }
}
```

### __*Does this replace private class fields and methods?*__

Yes. See [Private Symbols, or Private Fields](./symbols-or-fields.md).

## Semantics

- Symbols have an additional internal value `[[Private]]`, which is
    either **true** or **false**. By definition, private symbols are
    symbols whose `[[Private]]` value is **true**.
- The invariants of `[[OwnPropertyKeys]]` are modified such that it may
    not return any private symbols.
- The definition of `[[OwnPropertyKeys]]` for ordinary objects (and
    exotic objects other than proxy) is modified such that private symbols
    are filtered from the resulting array.
- The proxy definition of `[[OwnPropertyKeys]]` is modified such that an
    error is thrown if the handler returns any private symbols.
- The proxy definitions of all internal methods that accept property
    keys are modified such that if the provided key is a private symbol,
    then the operation is forwarded to the target without consulting the
    proxy handler.


See [Specification Changes](spec-changes.md) for more details.
