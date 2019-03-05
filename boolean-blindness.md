# Code Smell: Boolean Blindness

`Bool` represents a single bit of information:

~~~ haskell
data Bool = False | True
~~~

The popular term "boolean blindness" refers to the information lost by functions that operate on `Bool` when richer structures are available.
Using more structure can produce interfaces that are easier to document, use, decompose, and generalize.

`Data.List.filter` is a classic example of a function that would benefit from a richer interface type:

~~~ haskell
filter :: (a -> Bool) -> [a] -> [a]
--         ^~~~~~~~~ predicate: keep or discard each element
~~~

A "filter" has two uses:

1. To collect the fluid (filtrate) and discard the particulate, such as when making coffee.
1. To collect the particulate and discard the filtrate, such as when collecting gold.

Just as there are two uses for a filter, I can never remember if `filter`'s predicate means "keep" or "discard".
For the record, the `Prelude` definition is:

~~~ haskell
filter _    []       = []
filter keep (a : as)
    | keep a         = a : filter keep as
    | otherwise      =     filter keep as
~~~

... I think.

One (ahem) solution is to give `filter` a better name, one that does not have a dual identity:

~~~ haskell
select :: (a -> Bool) -> [a] -> [a]
--         ^~~~~~~~~ predicate: select each element
~~~

Another alternative is to supply more information in the type:

~~~ haskell
data Keep = Discard | Keep

filter :: (a -> Keep) -> [a] -> [a]
filter _    []       = []
filter keep (a : as)
    | Keep <- keep a = a : filter keep as
    | otherwise      =     filter keep as
~~~

Although it informs the programmer, `Keep` contains no more information than `Bool` from the machine's perspective.
The compiler will happily allow this definition, which does not do what you would expect:

~~~ haskell
evilFilter :: (a -> Keep) -> [a] -> [a]
evilFilter _    []       = []
evilFilter keep (a : as)
    | Discard <- keep a  = a : filter keep as
    | otherwise          =     filter keep as
~~~

`evilFilter` isn't a likely failure mode, unless you have a very antagonistic relationship with your team.
On the practical side, this popular style does have a significant disadvantage:
none of the convenient definitions for `Bool` will work for `Keep` unless we reimplement them ourselves.

Including more information in the type of `filter` can make the definition more general and even enlist the compiler's aid.
The most important feature of `filter` is that each element of the input list corresponds to zero (`Discard`) or one (`Keep`) elements of the output.
We can encode that intent in `Keep` by giving it a type parameter,

~~~ haskell
data Keep a = Discard | Keep a
~~~

and attaching zero or one elements of the input to the result of the predicate:

~~~ haskell
filter :: (a -> Keep a) -> [a] -> [a]
filter keep (a : as)
    | Keep b <- keep a = b : filter keep as
    | otherwise        =     filter keep as
~~~

Although this type is highly suggestive to the programmer, it still allows pathological implementations:

~~~ haskell
evilFilter :: (a -> Keep a) -> [a] -> [a]
evilFilter _ as = as  -- Signed-off-by: Your Adversarial Coworker :-P
~~~

We can use parametric polymorphism to rule out this adversary and demonstrate that the correct implementation **only** takes its outputs from `Keep`:

~~~ haskell
filter :: (a -> Keep b) -> [a] -> [b]
filter _    []       = []
filter keep (a : as)
    | Keep b <- keep a = b : filter keep as
    | otherwise        =     filter keep as
~~~

Note that we did not have to change the implementation because it was already correct!
We only needed to change the type to reflect facts about the correct implementation.
Changing the type also made `filter` more general:
the predicate may now transform the list at the same time as filtering it.

`Keep` is identical to `Maybe`, so we may write instead

~~~ haskell
filter :: (a -> Maybe b) -> [a] -> [b]
filter _    []       = []
filter keep (a : as)
    | Just b <- keep a = b : filter keep as
    | otherwise        =     filter keep as
~~~

and enjoy all the convenient functions that the standard library defines for `Maybe`.
This definition already exists as `Data.Maybe.mapMaybe`—it is unfortunately not in the `Prelude`!
This generalized definition can implement the original `filter` with a little help:

~~~ haskell
keeping :: (a -> Bool) -> (a -> Maybe a)
keeping predicate a
    | predicate a = Just a
    | otherwise   = Nothing

originalFilter :: (a -> Bool) -> [a] -> [a]
originalFilter predicate = filter (keeping predicate)
~~~

The generalized `filter` encourages to build a reusable components like `keeping`.
