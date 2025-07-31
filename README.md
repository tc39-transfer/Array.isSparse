# Array.isSparse

## Authors

- **Author**: Ruben Bridgewater
- **Champions**: Ruben Bridgewater, Jordan Harband
- **Stage**: 0
- **Related Proposals**: [Object.propertyCount()](https://github.com/tc39/proposal-object-property-count)

---

## Motivation

JavaScript arrays can be either *dense* or *sparse*. A sparse array is an array where one or more indices between `0` and `length - 1` are not present as own properties (a "hole"). Sparse arrays are a core part of JavaScript semantics, but there is no built-in way to determine whether an array is sparse.

Sparse arrays have long been discouraged in the developer community, largely for performance and correctness concerns, and most JS developers understand they "should" avoid them. However, they do exist, and thus must be accounted for if a correct implementation is to also be performant.

Today, developers must use cumbersome and very inefficient workarounds to detect sparse arrays. These methods typically involve iterating over keys, comparing `Object.keys(array).length` to `array.length`, or checking for missing elements manually. These approaches are error-prone, inefficient for large arrays, and not always semantically correct.

### Real-World Examples

1. **Data validation**: Ensuring an input array has no holes before processing.
2. **Serialization**: Sparse arrays can behave unexpectedly when serialized to JSON.
3. **Performance optimization**: Engines may deoptimize when arrays become sparse; libraries may wish to warn or convert. Algorithms can skip checks currently necessary to guard against sparse arrays when working with dense arrays.
4. **Correctness**: Developers sometimes unintentionally create sparse arrays, not knowing about the implications. This allows to easily test for that.
5. **Security**: If someone were to add an index onto `Array.prototype`, it would "leak" through the hole.

A direct, standard method is necessary to reduce boilerplate, improve code clarity, and provide a consistent and performant way to detect array sparseness.

### Summary

The main **goal** is to implement this for **correctness**, **performance**, and usage **simplicity**.

The latter is especially important for handling **dense** arrays faster when correctness is important!

### Concrete usage examples

- Node.js - all logging, all deep comparison methods
- lodash - dense-array fallback for every iterator
- jest - snapshot & diff logic skips holes
- fast-deep-equal - special-cases sparse vs. dense in deep-compare
- [deep-equal](https://www.npmjs.com/package/deep-equal) - would allow for optimizations
- [object-inspect](https://www.npmjs.com/package/object-inspect) - describing sparse arrays for debugging

Many use this in pattern in polyfilled code. It is possible that this allows *future* polyfills to also be more performant.

---

## Proposal

Introduce a new static method:

```js
Array.isSparse(value)
```

This method determines whether a given value is a sparse array. If the value is not an Array, it returns `false`.

### What is a Sparse Array?

An array `a` is *sparse* if there exists an integer index `i` such that:

- `0 <= i < a.length`
- `a` does **not** have an own property at index `i`

That is, it has at least one "hole" in the array indices.

# TODO: should arraylikes with holes be "sparse"?
 - pros: consistency with all other array methods (everything but Array.isArray, flat/flatMap callback return, IsConcatSpreadable, and JSON serialization treats arraylikes the same as arrays)
 - cons: wouldn't be performant for an arraylike
 - Suggestion: arraylikes with holes should be "sparse"

### Examples

```js
Array.isSparse([])                    // false
Array.isSparse([1, 2, 3])             // false
Array.isSparse(new Array(0))          // false
Array.isSparse({ 0: 'a', length: 1 }) // false (not an array)
Array.isSparse('abc')                 // false (not an array)

Array.isSparse([1, , 3])              // true
Array.isSparse(Array(5))              // true
Array.isSparse(new Array(1))          // true
```

### Why a Static Method?

- **Consistency**: Similar to `Array.isArray`, `Array.from`, etc.
- **Simplicity**: Does not depend on prototype chains or inheritance.
- **Strategy**: Does not carry web compatibility risk (browsers have set a very high bar for future Array.prototype methods)

---

## Specification

**Abstract Operation: Array.isSparse ( ***value*** )**

1. If `IsArray(_value_)` is *false*, return *false*.
2. Let *len* be `ToLength(Get(_value_, *"length"*))`.
3. For each integer _i_ from `0` to `_len_ - 1`:
   1. If `HasOwnProperty(_value_, ToString(_i_))` is *false*, return *true*.
4. Return *false*.

This specification should easily be optimizable by engines in case no proxy is used.
V8, SpiderMonkey, and JavaScriptCore all have internal representations for sparse arrays (while the current types might not indicate all possible cases right now).

---

## Polyfill / Implementation Notes

A simple polyfill:

```js
Array.isSparse ??= function isSparse(value) {
  if (!Array.isArray(value)) {
    return false;
  }
  for (let i = 0; i < value.length; i++) {
    if (!Object.hasOwn(value, i)) {
      return true;
    }
  }
  return false;
}
```

This polyfill is spec-compliant but will be significantly slower than native implementations, especially for large arrays in case no proxy is used. Notably, though, this will be identically performant to current userland solutions.

---

## Potential Risks or Drawbacks

- **No performance improvement**: Linear time scan on proxies may be expensive on large arrays, but this is equivalent to most existing workarounds. Other code would likely have an optimization to just return an internal type for non-Proxy cases.

---

## Relationship to Prior Work

### Existing Workarounds

- Checking `Object.keys(arr).length !== arr.length` (unsafe, because of non-index properties)
- Using `for` or `forEach` to detect holes
- Using `Array.prototype.every` (or another pre-ES6 callback-taking array method) with index checks

### Libraries

- Lodash, Ramda, and others do not offer built-in sparse detection. They do handle it internally though.
- Some frameworks sanitize arrays to avoid sparsity.

### Engines

- All major engines (V8, SpiderMonkey, JavaScriptCore) track sparsity internally for optimization.

### Related Proposals

- `Object.propertyCount()` proposal seeks to expose `[[OwnPropertyCount]]`, which would allow indirect sparse detection. However, it is more general-purpose and does not address this use case directly.
  Both proposals provide benefits as standalone proposal while providing additional benefits when being used together.

---

## References

- [Object.propertyCount Proposal](https://github.com/tc39/proposal-object-property-count)
- [MDN on Sparse Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array#sparse_arrays)
