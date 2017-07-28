# Optional Chaining for JavaScript

## Status
Current Stage:
* Stage 1

## Authors
* Claude Pache (@claudepache)
* Gabriel Isenberg (@the_gisenberg)

## Overview and motivation
When looking for a property value deeply in a tree structure, one has often to check whether intermediate nodes exist:

```javascript
var street = user.address && user.address.street;
```

Also, many API return either an object or null/undefined, and one may want to extract a property from the result only when it is not null:

```javascript
var fooInput = myForm.querySelector('input[name=foo]')
var fooValue = fooInput ? fooInput.value : undefined
```

The Optional Chaining Operator allows a developer to handle many of those cases without repeating themselves and/or assigning intermediate results in temporary variables:

```
var street = user.address?.street
var fooValue = myForm.querySelector('input[name=foo]')?.value
```

## Prior Art
* C#: [Null-conditional operator](https://msdn.microsoft.com/en-us/library/dn986595.aspx)
* Swift: [Optionals](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/TheBasics.html#//apple_ref/doc/uid/TP40014097-CH5-ID330)
* Groovy: [Safe navigation operator](http://groovy-lang.org/operators.html#_safe_navigation_operator)
* Ruby: [Safe navigation operator](http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/)
* CoffeeScript: [Existential operator](http://coffeescript.org/#existential-operator)

## Syntax

The Optional Chaining operator is spelled `?.`. It may appear in three positions:
```
obj?.prop       // optional static property access
obj?.[expr]     // optional dynamic property access
func?.(...args) // optional function or method call
```

### Notes
* In order to allow `foo?.3:0` to be parsed as `foo ? .3 : 0` (as required for backward compatibility), a simple lookahead is added at the level of the lexical grammar, so that the sequence of characters `?.` is not interpreted as a single token in that situation (the `?.` token must not be immediately followed by a decimal digit).
* We don’t use the `obj?[expr]` and `func?(...arg)` syntax, because of the difficulty for the parser to distinguish those forms from the conditional operator, e.g.` obj?[expr].filter(fun):0` and `func?(x - 2) + 3 :1`.

## Semantics

### Base case
If the operand at the left-hand side of the `?.` operator evaluates to undefined or null, the expression evaluates to undefined. Otherwise the targeted property access, method or function call is triggered normally.

Here are basic examples, each one followed by its desugaring. (The desugaring is not exact in the sense that the LHS should be evaluated only once.)
```js
a?.b                          // undefined if `a` is null/undefined, `a.b` otherwise.
a == null ? undefined : a.b   

a?.[x]                        // undefined if `a` is null/undefined, `a.[x]` otherwise.
a == null ? undefined : a[x]

a?.b()                        // undefined if `a` is null/undefined
a == null ? undefined : a.b() // throws a TypeError if `a.b` is not a function 
                              // otherwise, evaluates to `a.b()`

a?.()                        // undefined if `a` is null/undefined
a == null ? undefined : a()  // throws a TypeError if `a` is neither null/undefined, nor a function
                             // invokes the function `a` otherwise
```

### Short-circuiting

If the expression on the LHS of `?.` evaluates to null/undefined, the RHS is not evaluated. This concept is called *short-circuiting*.

```js
a?.[++x]         // `x` is incremented if and only if `a` is not null/undefined
a == null ? undefined : a[++x]
```

### Long short-circuiting

In fact, short-circuiting, when triggered, skips not only to the current property access, method or function call, but also the whole chain of property accesses, method and function calls following directly the Optional Chaining operator.

```js
a?.b.c(++x).d  // if `a` is null/undefined, evaluates to undefined. Variable `x` is not incremented.
               // otherwise, evaluates to `a.b.c(++x).d`. 
a == null ? undefined : a.b.c(++x).d
```

Note that the check for nullity is made on `a` only. If, for example, `a` is not null, but `a.b` is null, a TypeError will be thrown when attempting to access the property `"c"` of `a.b`.

This feature is implemented by, e.g., C# and CoffeeScript [TODO: provide precise references].

[TODO: since long short-circuiting is a criticised by some, write a justification why we *want* that. In the meanwile, ]
see [Issue #3 (comment)](https://github.com/tc39/proposal-optional-chaining/issues/3#issuecomment-306791812).

### Stacking 

Let’s call *Optional Chain* an Optional Chaining operator followed by a chain of property accesses, method and function calls.

An Optional Chain may be followed by another Optional Chain.

```js
a?.b[3].c?.(x).d
a == null ? undefined : a.b[3].c == null ? undefined : a.b[3].c.(x).d
  // (as always, except that `a` and `a.b[3].c` are evaluated only once)
```

### Edge case: grouping

Should parentheses limit the scope of short-circuting?

```js
(a?.b).c
(a == null ? undefined : a.b).c  // this?
a == null ? undefined : (a.b).c  // or that?
```
Given that parentheses are useless in that position, it should not have impact in day-to-day use.

Unless there is a strong reason for one way or the other, the answer will mostly depend on how the feature is specified.

### Optional deletion

Because the `delete` operator is very liberal in what it accepts, we have that feature for free:
```js
delete a?.b
delete (a == null ? undefined : a.b) // does work as expected
```

### Optional assignment

Should optional property assignement as in: `a?.b = c` be implemented? This is supported and used in CoffeeScript.

## Not supported

The following are not implemented for lack of real-world use cases.

* optional construction: `new a?.()`
* optional template literal: ``a?.`{b}` ``
* constructor or template literals in/after an Optional Chain: `new a?.b()`, ``a?.b`{c}` ``.

All the above cases will be forbidden by the grammar.


## TODO
Per the [TC39 process document](https://tc39.github.io/process-document/), here is a high level list of work that needs to happen across the various proposal stages.

* [x] Identify champion to advance addition (stage-1)
* [x] Prose outlining the problem or need and general shape of the solution (stage-1)
* [x] Illustrative examples of usage (stage-1)
* [x] High-level API (stage-1)
* [x] [Initial spec text](https://claudepache.github.io/es-optional-chaining/) (see also [WIP of a revision](ShortCircuitingWithoutNil.md)) (stage-2)
* [x] [Babel plugin](https://github.com/babel/babel/pull/5813) (stage-2)
* [ ] Finalize and reviewer signoff for spec text (stage-3)
* [ ] Test262 acceptance tests (stage-4)
* [ ] tc39/ecma262 pull request with integrated spec text (stage-4)
* [ ] Reviewer signoff (stage-4)

## References
* [TC39 Slide Deck: Null Propagation Operator](https://docs.google.com/presentation/d/11O_wIBBbZgE1bMVRJI8kGnmC6dWCBOwutbN9SWOK0fU/edit?usp=sharing)
* [es-optional-chaining](https://github.com/claudepache/es-optional-chaining) (@claudepache)
*  [ecmascript-optionals-proposal](https://github.com/davidyaha/ecmascript-optionals-proposal) (@davidyaha)

## Related issues
* [Babylon implementation](https://github.com/babel/babylon/issues/328)
* [estree: Null Propagation Operator](https://github.com/estree/estree/issues/146)
* [TypeScript: Suggestion: "safe navigation operator"](https://github.com/Microsoft/TypeScript/issues/16)
* [Flow: Syntax for handling nullable variables](https://github.com/facebook/flow/issues/607)

## Prior discussion
* https://esdiscuss.org/topic/existential-operator-null-propagation-operator
* https://esdiscuss.org/topic/optional-chaining-aka-existential-operator-null-propagation
* https://esdiscuss.org/topic/specifying-the-existential-operator-using-abrupt-completion
