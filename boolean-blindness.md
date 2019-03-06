% Code smell: Boolean blindness
% Thomas Tuegel

# Code smell: Boolean blindness

[`Bool`][Bool] represents a single bit of information:

~~~ {.haskell .ignore}
data Bool = False | True
~~~

The popular term "boolean blindness" refers to the information lost by functions that operate on `Bool` when richer structures are available.
Erasing such structure can give code a _bad smell_.
Using more structure can produce interfaces that are easier to document, use, decompose, and generalize.

## Example

[`Data.List.filter`][Data.List.filter] is a classic example of a function that would benefit from a richer interface type:

``` {.haskell .ignore}
filter :: (a -> Bool) -> [a] -> [a]
--         ^^^^^^^^^
--         predicate: keep or discard each element
```

A "filter" has two uses:

1. To collect the fluid (filtrate) and discard the particulate, such as when making coffee.
1. To collect the particulate and discard the filtrate, such as when collecting gold.

Just as there are two uses for a filter, I can never remember if `filter`'s predicate means "keep" or "discard".
For the record, the [`Prelude`][Prelude] definition is:

``` {#filter .haskell .ignore}
filter _    []       = []
filter keep (a : as)
  | keep a           = a : filter keep as
  | otherwise        =     filter keep as
```

... I think.
If I had the _other_ use of "filter" in mind when I wrote the function, I might have written instead,

``` {#filter-prime .haskell .ignore}
filter' :: (a -> Bool) -> [a] -> [a]
filter' _       []       = []
filter' discard (a : as)
  | discard a            =     filter' keep as
  | otherwise            = a : filter' keep as
```

`filter` and `filter'` are both perfectly fine functions, only their intent differs slightly.

## Expressive renaming

One (ahem) solution to clarify intent is to give [`filter`](#filter) a better name, one that does not have a dual identity:

``` {#select .haskell}
select :: (a -> Bool) -> [a] -> [a]
--         ^^^^^^^^^
--         predicate: select each element
select _    []       = []
select keep (a : as)
  | keep a           = a : select keep as
  | otherwise        =     select keep as
```

An alternative is to supply rename the type and constructors expressively,

``` {#filter1 .haskell}
data Keep = Discard | Keep

filter1 :: (a -> Keep) -> [a] -> [a]
filter1 _    []       = []
filter1 keep (a : as)
  | Keep <- keep a    = a : filter1 keep as
  | otherwise         =     filter1 keep as
```

This popular style has a practical disadvantage:
there are already many functions that work with `Bool`,
but none will work with `Keep` unless we reimplement them ourselves.

## Expressive structure

Including more structure in the type of `filter` can make the definition more general while expressing our intent more clearly in the code.
The most important feature of `filter` is that each element of the input list corresponds to zero (`Discard`) or one (`Keep`) elements of the output.
There is already a type which _structurally_ represents zero or one elements,

``` {.haskell .ignore}
data Maybe a = Nothing | Just a
```

We can use `Maybe` to express the intent that each input element yields zero or one output elements:

``` {#filter2 .haskell}
filter2 :: (a -> Maybe a) -> [a] -> [a]
filter2 _    []         = []
filter2 keep (a : as)
  | Just b <- keep a    = b : filter2 keep as
  | otherwise           =     filter2 keep as
```

`filter2` is a generalization of `filter` because we might transform the list as we filter it:

``` {#decrementPositive .haskell}
-- | Select only the positive elements and decrement them.
decrementPositive :: [Integer] -> [Integer]
decrementPositive = filter2 (\x -> if x > 0 then Just (x - 1) else Nothing)
```

Although the type of `filter2` is quite suggestive to the programmer, it is still not fully faithful to our intent!
Consider the following implementation with the same type, which has a subtle bug:

``` {#filter2-prime .haskell}
filter2' :: (a -> Maybe a) -> [a] -> [a]
filter2' _    []         = []
filter2' keep (a : as)
  | Just b <- keep a     = a : filter2' keep as
  | otherwise            =     filter2' keep as
```

We can use parametric polymorphism to express our intent faithfully in the code,

``` {#filter3 .haskell}
filter3 :: (a -> Maybe b) -> [a] -> [b]
filter3 _    []         = []
filter3 keep (a : as)
  | Just b <- keep a    = b : filter3 keep as
  | otherwise           =     filter3 keep as
```

The type of `filter3` ensures that its implementation only collects outputs from the predicate, because that is the only thing in scope which can produce a value of `b`.
Note that we did not have to change the implementation because it was already correct!
We only needed to change the type to express our intent more clearly.
Changing the type made the implementation still more general:
the predicate may now even transform the elements to a different type while filtering the list.

## Generalization, composition, and decomposition

[`filter3`](#filter3) already exists as [`Data.Maybe.mapMaybe`][Data.Maybe.mapMaybe]—it is unfortunately not in the `Prelude`!
The generalized definition can implement the original `filter` with a little help:

``` {#keeping-filter .haskell}
keeping :: (a -> Bool) -> (a -> Maybe a)
keeping predicate a
  | predicate a = Just a
  | otherwise   = Nothing

filter' :: (a -> Bool) -> [a] -> [a]
filter' predicate = filter3 (keeping predicate)
```

The generalized `filter` encourages to build a reusable components like `keeping`,
and `keeping` also allows us to reuse all the convenient definitions `Prelude` provides for `Bool`.

## Post script: How far is too far?

It might be tempting to carry on generalizing.
Instead of allowing zero or one output elements per input, we might allow any number of outputs:

``` {#filter4 .haskell}
filter4 :: (a -> [b]) -> [a] -> [b]
filter4 _    []       = []
filter4 keep (a : as) = keep a ++ filter4 keep as
```

This function is probably too general to be useful as a generalization of `filter`:
by allowing any number of outputs, we allow so many transformations that it makes the intent unclear.
This generalization does appear, though, in the [`Monad`][Monad] instance of lists:

``` {#flip-bind .haskell .ignore}
flip (>>=) :: (a -> [b]) -> [a] -> [b]
-- flip (>>=) === filter4
```

In the context of `filter`, we may have gone too far, but in another context this might be exactly the abstraction we need.

[Bool]: http://hackage.haskell.org/package/base-4.12.0.0/docs/Prelude.html#t:Bool
[Data.List.filter]: http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-List.html#v:filter
[Prelude]: http://hackage.haskell.org/package/base-4.12.0.0/docs/Prelude.html
[Data.Maybe.mapMaybe]: http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Maybe.html#v:mapMaybe
[Monad]: http://hackage.haskell.org/package/base-4.12.0.0/docs/Control-Monad.html#t:Monad
