---
layout: default
nav_order: 101
status: 
title: "SE-0101 Reconfiguring `sizeof` and related functions into a unified `MemoryLayout` struct"
author: []
proposal: SE-0101
author: []
review_manager: []
scheduled: 
implementation: 
bug: 
categories: jekyll update
---

# Reconfiguring `sizeof` and related functions into a unified `MemoryLayout` struct

* Proposal: [SE-0101](0101-standardizing-sizeof-naming.md)
* Authors: [Erica Sadun](http://github.com/erica), [Dave Abrahams](https://github.com/dabrahams)
* Review Manager: [Chris Lattner](http://github.com/lattner)
* Status: **Implemented (Swift 3)**
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution-announce/2016-July/000244.html)

## Introduction

This proposal addresses `sizeof`, `sizeofValue`, `strideof`, `strideofValue`, `align`, and `alignOf`. It discards the value-style standalone functions and combines the remaining items into a unified structure.

Review 1: 

* [Swift Evolution Review Thread](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160620/021527.html)
* [Original Proposal](https://github.com/apple/swift-evolution/blob/26e1e5b546b13fb66ee8877ad7018a7856e467ca/proposals/0101-standardizing-sizeof-naming.md)

Prior Discussions:

* Swift Evolution Pitch: [\[Pitch\] Renaming sizeof, sizeofValue, strideof,	strideofValue](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160530/019884.html)
* [Earlier Discussions](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160425/016042.html)
* [SE-0101 Review](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160620/021527.html)

## Motivation

Although `sizeof()`, etc are treated as terms of art, these names are appropriated from C. The functions do not correspond to anything named `sizeof` in LLVM. Swift's six freestanding memory functions increase the API surface area while providing lightly-used and unsafe functionality. 

These APIs are not like `map`, `filter`, and `Dictionary`. They're specialty items that you should only reach for when performing unsafe operations, mostly inside the guts of higher-level constructs.

Refactoring this proposal to use a single namespace increases discoverability, provides a single entry point for related operations, and enables future expansions without introducing further freestanding functions.

## Detailed Design

This proposal introduces a new struct, `MemoryLayout`

```swift
/// Accesses the memory layout of `T` through its
/// `size`, `stride`, and `alignment` properties
public struct MemoryLayout<T> {
    /// Returns the contiguous memory footprint of `T`.
    ///
    /// Does not include any dynamically-allocated or "remote" 
    /// storage. In particular, `MemoryLayout<T>.size`, when 
    /// `T` is a class type, is the same regardless of how many 
    /// stored properties `T` has.
     public static var size: Int { return _sizeof(T) }
    
    /// For instances of `T` in an `Array<T>`, returns the number of
    /// bytes from the start of one instance to the start of the
    /// next. This is the same as the number of bytes moved when an
    /// `UnsafePointer<T>` is incremented. `T` may have a lower minimal
    /// alignment that trades runtime performance for space
    /// efficiency. The result is always positive.
    public static var stride: Int { return _strideof(T) }
    
    /// Returns the default memory alignment of `T`.
    public static var alignment: Int { return _alignof(T) }
}
```

With this design, consumers call:

```swift
// Types
MemoryLayout<Int>.size // 8
MemoryLayout<Int>.stride // 8
MemoryLayout<Int>.alignment // 8
```

## Values

This proposal removes `sizeofValue()`, `strideofValue()`, and `alignofValue()` from the standard library. This proposal adopts the stance that sizes relate to types, not values.

Russ Bishop writes in the initial review thread, "Asking about the size of an instance implies things that aren’t true. Sticking _value_ labels on everything doesn’t change the fact that `sizeOf(swift_array)` is not going to give you the size of the underlying buffer no matter how you slice it."

As the following chart shows, type-based calls consistently outnumber instance-based calls in gist, github, and stdlib searches. The Google search for `sizeof` is probably too general based on its use in other languages.

<table>
<tr width = 800>
<th width = 200>Term</td>
<th width = 150>stdlib search</td>
<th width = 150>gist search</td>
<th width = 150>Google site:github.com swift</td>
</tr>
<tr width = 800>
<td width = 200>sizeof</td>
<td width = 150>157</td>
<td width = 150>169</td>
<td width = 150>(18,600, term is probably too general)</td>
</tr>
<tr width = 800>
<td width = 200>sizeofValue</td>
<td width = 150>4</td>
<td width = 150>34</td>
<td width = 150>584</td>
</tr>
<tr width = 800>
<td width = 200>alignof</td>
<td width = 150>44</td>
<td width = 150>11</td>
<td width = 150>334</td>
</tr>
<tr width = 800>
<td width = 200>alignofValue</td>
<td width = 150>5</td>
<td width = 150>5</td>
<td width = 150>154</td>
</tr>
<tr width = 800>
<td width = 200>strideof</td>
<td width = 150>24</td>
<td width = 150>19</td>
<td width = 150>347</td>
</tr>
<tr width = 800>
<td width = 200>strideofValue</td>
<td width = 150>1</td>
<td width = 150>5</td>
<td width = 150>163</td>
</tr>
</table>

If for some reason, the core team decides that there's a compelling reason to include value calls, an implementation might look something like this:

```swift
extension MemoryLayout<T> {
    init(_ : @autoclosure () -> T) {}
    public static func of(_ candidate : @autoclosure () -> T) -> MemoryLayout<T>.Type {
        return MemoryLayout.init(candidate).dynamicType
    }
}

// Value
let x: UInt8 = 5
MemoryLayout.of(x).size // 1
MemoryLayout.of(1).size // 8
MemoryLayout.of("hello").stride // 24
MemoryLayout.of(29.2).alignment // 8
```

#### Known Limitations and Bugs

According to Joe Groff, concerns about existential values (it's illegal to ask for the size of an existential value's dynamic type) could be addressed by 

> "having `sizeof` and friends formally take an Any.Type instead of <T> T.Type. (This might need some tweaking of the underlying builtins to be able to open existential metatypes, but it seems implementable.)" 

This proposal uses `<T>` /  `T.Type` to reflect Swift's current implementation.

**Note:** There is a [known bug](https://lists.swift.org/pipermail/swift-dev/Week-of-Mon-20160530/002150.html) (cite D. Gregor) that does not enforce `.self` when used with `sizeof`, allowing `sizeof(UInt)`. This call should be `sizeof(UInt.self)`. This proposal is written as if the bug were resolved without relying on adoption of [SE-0090](0090-remove-dot-self.md).

## Impact on Existing Code

This proposal requires migration support to replace function calls with struct-based namespacing. This should be a simple substitution with limited impact on existing code that is easily addressed with a fixit.

## Alternatives Considered

The original proposal introduced three renamed standalone functions:

```swift
public func memorySize<T>(ofValue _: @autoclosure T -> Void) -> Int
public func memoryInterval<T>(ofValue _: @autoclosure T -> Void) -> Int 
public func memoryAlignment<T>(ofValue _: @autoclosure T -> Void) -> Int
```

These functions offered human factor advantages over the current proposal but didn't address Dave's concerns about namespacing and overall safety. This alternative has been discarded and can be referenced by reading the original proposal.

## Acknowledgements

Thank you, Xiaodi Wu, Matthew Johnson, Pyry Jahkola, Tony Allevato, Joe Groff, Russ Bishop, and everyone else who contributed to this proposal
