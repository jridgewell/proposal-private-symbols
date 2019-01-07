# ECMAScript Proposal: Private Symbols

## Overview

This proposal allows for the encapsulation of "private" properties to
lexical environments. These are known as **private symbols** because
they behave similarly to regular symbols, but at this time there is no
actual reification of private symbols. Private symbols can only be
access through `base.#priv` syntax.

```js
class Example {
  #foo = 1;
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

`WeakSet`s are a natural solution to this.

```js
const brand = new WeakSet();
function check(...objects) {
  for (const obj of objects) {
    if (!brand.has(obj)) {
      throw new TypeError('non instance');
    }
  }
}

class Example {
  #foo = 1;

  constructor() {
    brand.add(this);
  }

  equal(obj) {
    check(this, obj);
    return this.#foo === obj.#foo;
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
