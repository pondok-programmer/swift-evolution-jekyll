---
layout: default
nav_order: 148
status: 
title: "SE-0148 Generic Subscripts"
author: []
proposal: SE-0148
author: []
review_manager: []
scheduled: 
implementation: 
bug: 
categories: jekyll update
---

# Generic Subscripts

* Proposal: [SE-0148](0148-generic-subscripts.md)
* Author: [Chris Eidhof](https://github.com/chriseidhof)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)
* Status: **Implemented (Swift 4)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170123/031048.html)
* Bug: [SR-115](https://bugs.swift.org/browse/SR-115)

## Introduction

Make it possible to have generic subscripts. Example:

```swift
extension Collection {
  subscript<Indices: Sequence>(indices: Indices) -> [Iterator.Element] where Indices.Iterator.Element == Index {
    // ...
  }
}
```

Or a generic return type:

```swift
extension JSON {
  subscript<T: JSONConvertible>(key: String) -> T? {
    // ...
  }
}
```

Swift-evolution thread: [Generic Subscripts](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170109/030064.html).

## Motivation

Currently, subscripts can't be generic. This is limiting in a number of ways: 

- Some subscripts are very specific and could be made more generic.
- Some generic methods would feel more natural as a subscript, but currently can't be. This also makes it impossible to use them as lvalues.

This feature is also mentioned in the generics manifesto under [generic subscripts](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md#generic-subscripts). The [Rationalizing Sequence end-operation names](https://github.com/apple/swift-evolution/blob/master/proposals/0132-sequence-end-ops.md) proposal could greatly benefit from this, as well as the ideas in the [String Manifesto](https://github.com/apple/swift/blob/master/docs/StringManifesto.md).

## Proposed solution

Add generics to subscripts. There are two pieces to this: where to add the generic parameter list, and where to add the `where`-clause. The most straightforward way would be to use the same syntax as methods:

```swift
extension Dictionary {
  subscript<Indices: Sequence>(indices: Indices) -> [Iterator.Element] where Indices.Iterator.Element == Index {
    // ...
  }
}
```

*Update Jan 20*: during the review it came up that while we're at it, we should add default arguments to subscripts. For example, the following (contrived) example:

```swift
subscript<A>(index: A? = nil) -> Element {
    // ...
}
```

Adding default arguments would unify the compiler's handling of subscripts and functions.

## Source compatibility

This is a purely additive change. We don't propose changing the Standard Library to use this new feature, that should be part of a separate proposal. (Likewise, we should consider making subscripts `throws` in a [separate proposal](https://github.com/brentdax/swift-evolution/blob/throwing-properties/proposals/0000-throwing-properties.md)).

## Effect on ABI stability

It won’t change the ABI of existing subscript calls.

## Effect on API resilience

It won’t change the ABI of existing subscript calls, but if the standard library introduces new generic subscripts that replace older non-generic subscripts, it will impact ABI.

## Alternatives considered

None.
