# SRFI: Flexvectors

## Title

Flexvectors

## Author

Adam Nelson

## Abstract

A _flexvector_, also known as a dynamic array or an arraylist, is a mutable vector-like data structure with an adjustable size. Flexvectors allow O(1) random access, O(1) insertion/removal at the end, and O(n) insertion/removal elsewhere. This SRFI defines a suite of operations on flexvectors, modeled after [SRFI 133][srfi-133]'s vector operations.

## Issues

The test suite does not yet fully cover the specification. There are several additional flexvector operations that would be useful (`flexvector-append!`, `flexvector-append-map`) that are not in this draft but may be included in a future draft.

## Rationale

Unlike the default vector datatype of many other languages, Scheme vectors have a fixed length. This makes vectors unusable as mutable stacks or queues, and is the reason that [SRFI 133][srfi-133] lacks common collection operations like `filter`.

In fact, no Scheme standard defines a mutable datatype that is suitable for this very common purpose, analogous to a Java `ArrayList` or to the default list data structure in JavaScript or Python. [SRFI 117][srfi-117] defines the commonly-used "tconc" mutable queue, but it is a linked list. And [SRFI 134][srfi-134] defines a deque datatype, but that datatype is immutable. Neither data structure has the (often useful) properties of being a mutable, contiguous, random-access sequence.

This SRFI does not define a comparator or any sorting procedures, in order to ensure that it has no dependencies. These may be defined in future SRFIs.

### Terminology

What this SRFI calls a _flexvector_ is better known as a [_dynamic array_][dynamic-array]. This data structure has a wide variety of names in different languages:

- JavaScript and Ruby call it an _array_
- Python calls it a _list_
- Java calls it an `ArrayList` (and, before that, it was called a `Vector`)

Scheme literature already uses the terms _list_ (for `cons` lists), _vector_ (for fixed-length vectors), and _array_ (for fixed-length numeric arrays), so a new term is needed. Because _array_ in Scheme refers to a numeric array, the term _dynamic array_ is not ideal. _Dynamic vector_ is a possibility, but `dynamic-vector` would be an unwieldy prefix due to its length. The newly-minted term _flexvector_ communicates this data structure's flexible size and its relationship to Scheme vectors, while being only as long as an existing Scheme data type name (`bytevector`).

### Procedure inclusion and naming

This SRFI is primarily modeled on [SRFI 133][srfi-133]. It includes flexvector equivalents of all SRFI 133 procedures, most with the same names and argument order. There are two notable exceptions:

1. `flexvector-unfold` mimics the API of [SRFI 1][srfi-1]'s `unfold`, not SRFI 133's `vector-unfold`. `vector-unfold` is limited by the necessity of a fixed vector length, while `flexvector-unfold` may generate a flexvector of any length, and so the `unfold` API is more useful.
2. The flexvector equivalent of `vector=` is `flexvector=?`. It is conventional for Scheme equality predicates to end in `=?` (e.g., `symbol=?`, `string=?`), and most data structure SRFIs follow this convention (see [SRFI 113][srfi-113], [125][srfi-125], [146][srfi-146]). This SRFI follows established convention, even when it does not match SRFI 133's procedure names.

Additionally, this SRFI includes deque-like operations that reference, add to, and remove from both ends of a flexvector. The naming convention for these operations is taken from [SRFI 134][srfi-134], which uses the terms _front_ and _back_. _Front_ refers to the start of the flexvector (index 0), while _back_ refers to the end (index `(- (flexvector-length x) 1)`).

## Specification

In this section, the following notation is used to specify parameters and examples:

- Parameters named `fv` are always flexvectors.

- `x ...` indicates that zero or more of the parameter `x` are allowed.

- `[x]` indicates that the parameter `x` is optional. Multiple optional parameters may be written `[x [y]]`.

- `;=>` separates an example expression from its expected result.

- `λ` is a shorthand alias for `lambda`.

Additionally, examples include literal flexvector values written using [SRFI 10][srfi-10] notation: `#,(flexvector a b c)` is a flexvector of length 3 containg the symbol values `a`, `b`, and `c`. **This notation is only used for example purposes.** This SRFI does not depend on SRFI 10, and it does not define the `#,(flexvector x ...)` syntax for actual use.

### Constructors

#### `make-flexvector`

`(make-flexvector size [fill])`

Creates and returns a flexvector of size `size`. If `fill` is specified, all of the elements of the vector are initialized to `fill`. Otherwise, their contents are indeterminate.

```scheme
(make-flexvector 5 3) ;=> #,(flexvector 3 3 3 3 3)
```

#### `flexvector`

`(flexvector x ...)`

Creates and returns a flexvector whose elements are `x ...`.

```scheme
(flexvector 0 1 2 3 4) ;=> #,(flexvector 0 1 2 3 4)
```

#### `flexvector-unfold`, `flexvector-unfold-right`

`(flexvector-unfold p f g initial-seed ...)`

The fundamental flexvector constructor. `flexvector-unfold` is modeled on SRFI 1 `unfold` instead of SRFI 133 `vector-unfold`, because flexvectors are not limited to a predetermined length.

For each step, `flexvector-unfold` evaluates `p` on the seed value(s) to determine whether it should stop unfolding. If `p` returns `#f`, it then evaluates `f` on the seed value(s) to produce the next element, then evaluates `g` on the seed value(s) to produce the seed value(s) for the next step. The recursion can be described with this algorithm:

```scheme
(let recur ((seeds initial-seed) (fv (flexvector)))
  (if (apply p seeds) fv
      (let-values ((next-seeds (apply g seeds)))
        (recur next-seeds (flexvector-add-back! fv (apply f seeds))))))
```

This is guaranteed to build a flexvector in O(n), because `flexvector-add-back!` is O(1). `flexvector-unfold-right` is a variant that constructs a flexvector right-to-left, and uses `flexvector-add-front!` instead, which may be slower than O(n).

#### `flexvector-copy`, `flexvector-reverse-copy`

`(flexvector-copy fv [start [end]])`

Allocates a new flexvector whose length is `(- end start)` and fills it with elements from `fv`, taking elements from `fv` starting at index `start` and stopping at index `end`. `start` defaults to `0` and `end` defaults to the value of `(flexvector-length fv)`.

```scheme
(flexvector-copy (flexvector 'a 'b 'c)) ;=> #,(flexvector a b c)
(flexvector-copy (flexvector 'a 'b 'c) 1) ;=> #,(flexvector b c)
(flexvector-copy (flexvector 'a 'b 'c) 1 2) ;=> #,(flexvector b)
```

`flexvector-reverse-copy` is the same, but copies the elements in reverse order from `fv`.

```scheme
(flexvector-reverse-copy (flexvector 'a 'b 'c 'd) 1 4)
;=> #,(flexvector d c b)
```

Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`). It is an error if `end` is less than `start`.

#### `flexvector-append`

`(flexvector-append fv ...)`

Returns a newly allocated flexvector that contains all elements in order from the subsequent locations in `fv ...`.

```scheme
(flexvector-append (flexvector 'x) (flexvector 'y))
;=> #,(flexvector x y)

(flexvector-append (flexvector 'a) (flexvector 'b 'c 'd))
;=> #,(flexvector a b c d)

(flexvector-append (flexvector 'a (flexvector 'b))
                   (flexvector (flexvector 'c)))
;=> #,(flexvector a #,(flexvector b) #,(flexvector c))
```

#### `flexvector-concatenate`

`(flexvector-concatenate list-of-flexvectors)`

Equivalent to `(apply flexvector-append list-of-flexvectors)`, but may be implemented more efficiently.

```scheme
(flexvector-concatenate (list (flexvector 'a 'b) (flexvector 'c 'd)))
;=> #,(flexvector a b c d)
```

#### `flexvector-append-subvectors`

`(flexvector-append-subvectors [fv start end] ...)`

Returns a vector that contains every element of each `fv` from `start` to `end` in the specified order. This procedure is a generalization of `flexvector-append`.

```scheme
(flexvector-append-subvectors (flexvector 'a 'b 'c 'd 'e) 0 2
                              (flexvector 'f 'g 'h 'i 'j) 2 4)
;=> #,(flexvector a b h i)
```

### Predicates

#### `flexvector?`

`(flexvector? x)`

Disjoint type predicate for flexvectors: this returns `#t` if `x` is a flexvector, and `#f` otherwise.

```scheme
(flexvector? (flexvector 1 2 3)) ;=> #t
(flexvector? (vector 1 2 3)) ;=> #f
```

#### `flexvector-empty?`

`(flexvector-empty? fv)`

Returns `#t` if `fv` is empty (i.e., its length is `0`), and `#f` if not.

```scheme
(flexvector-empty? (flexvector)) ;=> #t
(flexvector-empty? (flexvector 'a)) ;=> #f
(flexvector-empty? (flexvector (flexvector))) ;=> #f
```

#### `flexvector=?`

`(flexvector=? elt=? fv ...)`

Flexvector structural equality predicate, generalized across user-specified element equality predicates. Flexvectors `a` and `b` are considered equal by `flexvector=?` iff their lengths are the same and, for each index `i` less than `(flexvector-length a)`, `(elt=? (flexvector-ref a i) (flexvector-ref b i))` is true. `elt=?` is always applied to two arguments.

```scheme
(flexvector=? eq? (flexvector 'a 'b) (flexvector 'a 'b)) ;=> #t
(flexvector=? eq? (flexvector 'a 'b) (flexvector 'b 'a)) ;=> #f
(flexvector=? = (flexvector 1 2 3 4 5) (flexvector 1 2 3 4)) ;=> #f
(flexvector=? = (flexvector 1 2 3 4) (flexvector 1 2 3 4)) ;=> #t
```

`flexvector=?` returns `#t` if it is passed zero or one `fv` arguments. The execution order of comparisons is intentionally left unspecified.

```scheme
(flexvector=? eq?) ;=> #t
(flexvector=? eq? (flexvector 'a)) ;=> #t
```

### Selectors

#### `flexvector-ref`

`(flexvector-ref fv i)`

Flexvector element dereferencing: returns the value at location `i` in `fv`. Indexing is zero-based. It is an error if `i` is outside the range [0, `(flexvector-length fv)`).

```scheme
(flexvector-ref (flexvector 'a 'b 'c 'd) 2) ;=> c
```

#### `flexvector-front`

`(flexvector-front fv)`

Returns the first element in `fv`. It is an error if `fv` is empty. Alias for `(flexvector-ref fv 0)`.

```scheme
(flexvector-front (flexvector 'a 'b 'c 'd)) ;=> a
```

#### `flexvector-back`

`(flexvector-back fv)`

Returns the last element in `fv`. It is an error if `fv` is empty. Alias for `(flexvector-ref fv (- (flexvector-length fv) 1))`.

```scheme
(flexvector-back (flexvector 'a 'b 'c 'd)) ;=> d
```

#### `flexvector-length`

`(flexvector-length fv)`

Returns the length of `fv`, which is the number of elements contained in `fv`.

    (flexvector-length (flexvector 'a 'b 'c)) ;=> 3

### Mutators

#### `flexvector-add!`

`(flexvector-add! fv i x ...)`

Inserts the elements `x ...` into `fv` at the location `i`, preserving their order and shifting all elements after `i` backward to make room. This increases the length of `fv` by the number of elements inserted.

It is an error if `i` is outside the range [0, `(flexvector-length fv)`].

`flexvector-add!` returns `fv` after mutating it.

```scheme
(flexvector-add! (flexvector 'a 'b) 1 'c) ;=> #,(flexvector a c b)
(flexvector-add! (flexvector 'a 'b) 2 'c 'd 'e) ;=> #,(flexvector a b c d e)
```

#### `flexvector-add-front!`, `flexvector-add-back!`

`(flexvector-add-front! fv x ...)`

Inserts the elements `x ...` into the front or back of `fv`, preserving their order. This increases the length of `fv` by the number of elements inserted. `flexvector-add-back!` of one element is guaranteed to execute in O(1).

These procedures return `fv` after mutating it.

```scheme
(flexvector-add-front! (flexvector 'a 'b) 'c) ;=> #,(flexvector c a b)
(flexvector-add-front! (flexvector 'a 'b) 'c 'd) ;=> #,(flexvector c d a b)

(flexvector-add-back! (flexvector 'a 'b) 'c) ;=> #,(flexvector a b c)
(flexvector-add-back! (flexvector 'a 'b) 'c 'd) ;=> #,(flexvector a b c d)
```

#### `flexvector-add-all!`

`(flexvector-add-all! fv i xs)`

Inserts the elements of the list `xs` into `fv` at location `i`. Equivalent to `(apply flexvector-add! fv i xs)`. Returns `fv` after mutating it.

```scheme
(flexvector-add-all! (flexvector 'a 'b) 2 '(c d e)) ;=> #,(flexvector a b c d e)
```

#### `flexvector-remove!`

`(flexvector-remove! fv i)`

Removes and returns the element at `i` in `fv`, then shifts all subsequent elements forward, reducing the length of `fv` by 1.

It is an error if `i` is outside the range [0, `(flexvector-length fv)`).

#### `flexvector-remove-front!`, `flexvector-remove-back!`

`(flexvector-remove-front! fv)`

Removes and returns the first element from `fv`, then shifts all other elements forward. `flexvector-remove-back!` instead removes the last element, without moving any other elements, and is guaranteed to execute in O(1).

It is an error if `fv` is empty.

#### `flexvector-remove-range!`

`(flexvector-remove-range! fv start [end])`

Removes all elements from `fv` between `start` and `end`, shifting all elements after `end` forward by `(- end start)`. If `end` is not present, it defaults to `(flexvector-length fv)`.

Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`). It is an error if `end` is less than `start`.

`flexvector-remove-range!` returns `fv` after mutating it.

#### `flexvector-clear!`

`(flexvector-clear! fv)`

Removes all items from `fv`, setting its length to 0. Returns `fv` after mutating it.

#### `flexvector-set!`

`(flexvector-set! fv i x)`

Assigns the value of `x` to the location `i` in `fv`. It is an error if `i` is outside the range [0, `(flexvector-length fv)`]. If `i` is equal to `(flexvector-length fv)`, `x` is appended after the last element in `fv`; this is equivalent to `flexvector-add-back!`.

Returns the previous value at location `i` in `fv`, or an unspecified value if `i` is equal to `(flexvector-length fv)`.

#### `flexvector-swap!`

`(flexvector-swap! fv i j)`

Swaps or exchanges the values of the locations in `fv` at indexes `i` and `j`. It is an error if either `i` or `j` is outside the range [0, `(flexvector-length fv)`). Returns `fv` after mutating it.

#### `flexvector-fill!`

`(flexvector-fill! fv fill [start [end]])`

Assigns the value of every location in `fv` between `start`, which defaults to 0 and `end`, which defaults to the length of `fv`, to `fill`. Returns `fv` after mutating it.

Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`]. It is an error if `end` is less than `start`.

#### `flexvector-reverse!`

`(flexvector-reverse! fv)`

Destructively reverses `fv` in-place. Returns `fv` after mutating it.

#### `flexvector-copy!`, `flexvector-reverse-copy!`

`(flexvector-copy! to at from [start [end]])`

Copies the elements of flexvector `from` between `start` and `end` to flexvector `to`, starting at `at`. The order in which elements are copied is unspecified, except that if the source and destination overlap, copying takes place as if the source is first copied into a temporary vector and then into the destination. This can be achieved without allocating storage by making sure to copy in the correct direction in such circumstances.

`flexvector-reverse-copy!` is the same, but copies elements in reverse order.

`start` and `end` default to 0 and `(flexvector-length from)` if not present. Both `start` and `end` are clamped to the range [0, `(flexvector-length from)`]. It is an error if `end` is less than `start`.

Unlike `vector-copy!`, `flexvector-copy!` may copy elements past the end of `to`, which will increase the length of `to`.

Both procedures return `to` after mutating it.

#### `flexvector-unfold!`, `flexvector-unfold-right!`

`(flexvector-unfold! f fv start end initial-seed ...)`

Like `flexvector-unfold`, but the elements are copied into the flexvector `fv` starting at element `start` rather than into a newly allocated flexvector. Terminates when `(- end start)` elements have been generated.

`flexvector-unfold-right!` is the same, but copies elements in reverse order, starting at the index preceding `end`.

It is an error if `start` is less than 0 or `end` is less than `start`. Unlike `vector-unfold!`, `flexvector-unfold!` allows `end` to be greater than the length of `fv`; this will grow the length of `fv` to `end`.

Both procedures return `fv` after mutating it.

### Iteration

#### `flexvector-fold`, `flexvector-fold-right`

`(flexvector-fold kons knil fv1 fv2 ...)`

The fundamental flexvector iterator. `kons` is iterated over each value in all of the vectors, stopping at the end of the shortest; `kons` is applied as `(kons state (flexvector-ref fv1 i) (flexvector-ref fv2 i) ...)` where `state` is the current state value—the current state value begins with `knil`, and becomes whatever `kons` returned on the previous iteration—and `i` is the current index.

The iteration of `flexvector-fold` is strictly left-to-right. The iteration of `flexvector-fold-right` is strictly right-to-left.

```scheme
(flexvector-fold (λ (len str) (max (string-length str) len))
                 0
                 (flexvector "baz" "qux" "quux"))
;=> 4

(flexvector-fold-right (λ (tail elt) (cons elt tail))
                       '()
                       (flexvector 1 2 3))
;=> (1 2 3)

(flexvector-fold (λ (counter n)
                   (if (even? n) (+ counter 1) counter))
                 0
                 (flexvector 1 2 3 4 5 6 7))
;=> 3
```

#### `flexvector-map`, `flexvector-map/index`

`(flexvector-map f fv1 fv2 ...)`

Constructs a new flexvector of the shortest size of the flexvector arguments. Each element at index `i` of the new flexvector is mapped from the old flexvectors by `(f (flexvector-ref fv1 i) (flexvector-ref fv2 i) ...)`. The dynamic order of application of `f` is unspecified.

`flexvector-map/index` is a variant that passes `i` as the first argument to `f` for each element.

```scheme
(flexvector-map (λ (x) (* x 10)) (flexvector 10 20 30))
;=> #,(flexvector 100 200 300)

(flexvector-map/index (λ (i x) (+ x (* i 2))) (flexvector 10 20 30))
;=> #,(flexvector 10 22 34)
```

#### `flexvector-map!`, `flexvector-map/index!`

`(flexvector-map! f fv1 fv2 ...)`

Similar to `flexvector-map`, but rather than mapping the new elements into a new flexvector, the new mapped elements are destructively inserted into `fv1`. Again, the dynamic order of application of `f` unspecified, so it is dangerous for `f` to apply either `flexvector-ref` or `flexvector-set!` to `fv1` in `f`.

```scheme
(let ((fv (flexvector 10 20 30)))
  (flexvector-map! (λ (x) (* x 10)) fv)
  fv)
;=> #,(flexvector 100 200 300)

(let ((fv (flexvector 10 20 30)))
  (flexvector-map/index (λ (i x) (+ x (* i 2))) fv)
  fv)
;=> #,(flexvector 10 22 34)
```

#### `flexvector-filter`, `flexvector-filter/index`

`(flexvector-filter pred? fv)`

Constructs a new flexvector consisting of only the elements of `fv` for which `pred?` returns a non-`#f` value. `flexvector-filter/index` passes the index of each element as the first argument to `pred?`, and the element itself as the second argument.

```scheme
(flexvector-filter even? (flexvector 1 2 3 4 5 6 7 8))
;=> #,(flexvector 2 4 6 8)
```

#### `flexvector-filter!`, `flexvector-filter/index!`

`(flexvector-filter! pred? fv)`

Similar to `flexvector-filter`, but destructively updates `fv` by removing all elements for which `pred?` returns `#f`. `flexvector-filter/index!` passes the index of each element as the first argument to `pred?`, and the element itself as the second argument.

```scheme
(let ((fv (flexvector 1 2 3 4 5 6 7 8)))
  (flexvector-filter! odd? fv)
  fv)
;=> #,(flexvector 1 3 5 7)
```

#### `flexvector-for-each`, `flexvector-for-each/index`

`(flexvector-for-each f fv1 fv2 ...)`

Simple flexvector iterator: applies `f` to the corresponding list of parallel elements from `fv1 fv2 ...` in the range [0, `length`), where `length` is the length of the smallest flexvector argument passed, In contrast with `flexvector-map`, `f` is reliably applied in left-to-right order, starting at index 0, in the flexvectors.

`flexvector-for-each/index` is a variant that passes the index as the first argument to `f` for each element.

Example:

```scheme
(flexvector-for-each (λ (x) (display x) (newline))
                     (flexvector "foo" "bar" "baz" "quux" "zot"))
```

Displays:

```
foo
bar
baz
quux
zot
```

#### `flexvector-count`

`(flexvector-count pred? fv1 fv2 ...)`

Counts the number of parallel elements in the flexvectors that satisfy `pred?`, which is applied, for each index `i` in the range [0, *length*) where *length* is the length of the smallest flexvector argument, to each parallel element in the flexvectors, in order.

```scheme
(flexvector-count even? (flexvector 3 1 4 1 5 9 2 5 6))
;=> 3

(flexvector-count < (flexvector 1 3 6 9) (flexvector 2 4 6 8 10 12))
;=> 2
```

#### `flexvector-cumulate`

`(flexvector-cumulate f knil fv)`

Returns a newly-allocated flexvector `new` with the same length as `fv`. Each element `i` of `new` is set to the result of `(f (flexvector-ref new (- i 1)) (flexvector-ref vec i))`, except that, for the first call on `f`, the first argument is `knil`. The `new` flexvector is returned.

```scheme
(flexvector-cumulate + 0 (flexvector 3 1 4 1 5 9 2 5 6))
;=> #,(flexvector 3 4 8 9 14 23 25 30 36)
```

### Searching

#### `flexvector-index`, `flexvector-index-right`

`(flexvector-index pred? fv1 fv2 ...)`

Finds and returns the index of the first elements in `fv1 fv2 ...` that satisfy `pred?`. If no matching element is found by the end of the shortest flexvector, `#f` is returned.

`flexvector-index-right` is similar, but returns the index of the *last* elements that satisfy `pred?`, and requires all flexvector arguments to have the same length.

```scheme
(flexvector-index even? (flexvector 3 1 4 1 5 9))
;=> 2

(flexvector-index < (flexvector 3 1 4 1 5 9 2 5 6) (flexvector 2 7 1 8 2))
;=> 1

(flexvector-index = (flexvector 3 1 4 1 5 9 2 5 6) (flexvector 2 7 1 8 2))
;=> #f

(flexvector-index-right < (flexvector 3 1 4 1 5) (flexvector 2 7 1 8 2))
;=> 3
```

#### `flexvector-skip`, `flexvector-skip-right`

`(flexvector-skip pred? fv1 fv2 ...)`

Finds and returns the index of the first elements in `fv1 fv2 ...` that do *not* satisfy `pred?`. If all the values in the flexvectors satisfy `pred?` until the end of the shortest flexvector, this returns `#f`.

`flexvector-skip-right` is similar, but returns the index of the *last* elements that do not satisfy `pred?`, and requires all flexvector arguments to have the same length.

```scheme
(flexvector-skip number? (flexvector 1 2 'a 'b 3 4 'c 'd))
;=> 2

(flexvector-skip-right number? (flexvector 1 2 'a 'b 3 4 'c 'd))
;=> 4
```

#### `flexvector-binary-search`

`(flexvector-binary-search fv value cmp [start [end]])`

Similar to `flexvector-index` and `flexvector-index-right`, but, instead of searching left-to-right or right-to-left, this performs a binary search. If there is more than one element of `fv` that matches `value` in the sense of `cmp`, `flexvector-binary-search` may return the index of any of them.

The search is performed on only the indexes of `fv` between `start`, which defaults to 0, and `end`, which defaults to the length of `fv`. Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`]. It is an error if `end` is less than `start`.

`cmp` should be a procedure of two arguments that returns either a negative integer, which indicates that its first argument is less than its second; zero, which indicates that they are equal; or a positive integer, which indicates that the first argument is greater than the second argument. An example `cmp` might be:

```scheme
(λ (char1 char2)
  (cond ((char<? char1 char2) -1)
        ((char=? char1 char2) 0)
        (else 1)))
```

#### `flexvector-any`

`(flexvector-any pred? fv1 fv2 ...)`

Finds the first set of elements in parallel from `fv1 fv2 ...` for which `pred?` returns a true value. If such a parallel set of elements exists, `flexvector-any` returns the value that `pred?` returned for that set of elements. The iteration is strictly left-to-right.

#### `flexvector-every`

`(flexvector-every pred? fv1 fv2 ...)`

If, for every index `i` between 0 and the length of the shortest flexvector argument, the set of elements `(flexvector-ref fv1 i) (flexvector-ref fv2 i) ...` satisfies `pred?`, `flexvector-every` returns the value that `pred?` returned for the last set of elements, at the last index of the shortest flexvector. The iteration is strictly left-to-right.

#### `flexvector-partition`

`(flexvector-partition pred? fv)`

Returns two values: a flexvector containing all elements of `fv` that satisfy `pred?`, and a flexvector containing all elements of `fv` that do not satisfy `pred?`. Elements remain in their original order.

### Conversion

#### `flexvector->vector`

`(flexvector->vector fv [start [end]])`

Creates a vector containing the elements in `fv` between `start`, which defaults to 0, and `end`, which defaults to the length of `fv`.

Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`). It is an error if `end` is less than `start`.

#### `vector->flexvector`

`(vector->flexvector vec [start [end]])`

Creates a flexvector containing the elements in `vec` between `start`, which defaults to 0, and `end`, which defaults to the length of `vec`.

Both `start` and `end` are clamped to the range [0, `(vector-length vec)`). It is an error if `end` is less than `start`.

#### `flexvector->list`, `reverse-flexvector->list`

`(flexvector->list fv [start [end]])`

Creates a list containing the elements in `fv` between `start`, which defaults to 0, and `end`, which defaults to the length of `fv`.

`reverse-flexvector->list` is similar, but creates a list with elements in reverse order of `fv`.

Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`). It is an error if `end` is less than `start`.

#### `list->flexvector`, `reverse-list->flexvector`

`(list->flexvector proper-list)`

Creates a flexvector of elements from `proper-list`.

`reverse-list->flexvector` is similar, but creates a flexvector with elements in reverse order of `proper-list`.

#### `flexvector->string`

`(flexvector->string fv [start [end]])`

Creates a string containing the elements in `fv` between `start`, which defaults to 0, and `end`, which defaults to the length of `fv`. It is an error if the elements are not characters.

Both `start` and `end` are clamped to the range [0, `(flexvector-length fv)`). It is an error if `end` is less than `start`.

#### `string->flexvector`

`(string->flexvector string [start [end]])`

Creates a flexvector containing the elements in `string` between `start`, which defaults to 0, and `end`, which defaults to the length of `string`.

Both `start` and `end` are clamped to the range [0, `(string-length string)`). It is an error if `end` is less than `start`.

#### `generator->flexvector`

`(generator->flexvector gen)`

Creates a flexvector containing all elements produced by the [SRFI 158][srfi-158] generator `gen`.

## Implementation

[A sample implementation is available on GitHub.](https://github.com/ar-nelson/scheme-flexvectors) The sample implementation supports Gauche, Sagittarius, and Chibi, and includes a test suite.

## Acknowledgements

Thanks to the authors of [SRFI 133][srfi-133] (John Cowan, and, transitively, Taylor Campbell), on whose work this SRFI is based. Much of the language in this SRFI was copied directly from 133 with only minor changes.

## Copyright

Copyright © Adam Nelson (2020).

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice (including the next paragraph) shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

[dynamic-array]: https://en.wikipedia.org/wiki/Dynamic_array
[srfi-1]: https://srfi.schemers.org/srfi-1/srfi-1.html
[srfi-10]: https://srfi.schemers.org/srfi-10/srfi-10.html
[srfi-113]: https://srfi.schemers.org/srfi-113/srfi-113.html
[srfi-117]: https://srfi.schemers.org/srfi-117/srfi-117.html
[srfi-125]: https://srfi.schemers.org/srfi-125/srfi-125.html
[srfi-133]: https://srfi.schemers.org/srfi-133/srfi-133.html
[srfi-134]: https://srfi.schemers.org/srfi-134/srfi-134.html
[srfi-146]: https://srfi.schemers.org/srfi-146/srfi-146.html
[srfi-158]: https://srfi.schemers.org/srfi-158/srfi-158.html
