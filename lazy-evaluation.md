% Lazy evaluation by example
% Thomas Tuegel

...

## Example

Let me begin by defining some types, so that I have some constructors to play with:

``` {#One .haskell}
-- | One contains one Integer.
data One = One Integer
  deriving Show
```

``` {#None .haskell}
-- | None contains no Integers.
data None = None
  deriving Show
```

I can define a simple function to convert `One` into `None`:

``` {#noneOf .haskell}
noneOf :: One -> None
noneOf _ = None
```

Let me also define,

``` {#strictlyNoneOf .haskell}
strictlyNoneOf :: One -> None
strictlyNoneOf (One _) = None
```

The functions `noneOf` and `strictlyNoneOf` are quite similar in most respects, for example,

```
ghci> noneOf (One 2)
None
ghci> strictlyNoneOf (One 2)
None
```

Sometimes we say these functions are "morally" equivalent, because they give the same result for well-defined inputs.
They behave differently with undefined inputs due to Haskell's lazy evaluation strategy:

```
ghci> noneOf undefined
None
ghci> strictlyNoneOf undefined
*** Exception: Prelude.undefined
```

## Explanation

Haskell uses a call-by-name evaluation strategy so that values are computed when a pattern could match a constructor.
`noneOf` never evaluates its argument because it never tries to match a constructor pattern.
`strictlyNoneOf` is stricter because it matches on the constructor `One`; trying to match a constructor pattern triggers evaluation.

Exercise: What is `strictlyNoneOf (One undefined)`?

Solution:

```
ghci> strictlyNoneOf (One undefined)
None
```

Evaluation is triggered by matching on a constructor pattern.
The _argument_ of `One` is never evaluated because `strictlyNoneOf` never tries to match it.

## Weak head normal form

Can we write the following function,

``` {#strictlyAny .haskell}
strictlyAny :: forall a. a -> None
```

so that it evaluates its argument,

```
ghci> strictlyAny undefined
*** Exception: Prelude.undefined
```

simply using constructor pattern matching?

No! The type of the argument is variable, so we do not know which constructors to match.
It is impossible to write this function using plain pattern matching, so the compiler provides `Prelude.seq`:

``` {#seq .haskell .ignore}
seq :: a -> b -> b
```

`Prelude.seq` evaluates its first argument before returning its second argument;
we can use it to implement `strictlyAny`:

``` {.haskell}
strictlyAny a = seq a None
```

As required, we find

```
ghci> strictlyAny undefined
*** Exception: Prelude.undefined
```

`Prelude.seq` evaluates its argument to _weak head normal form_,
as if a pattern had matched on the top-most constructor,
so that `strictlyAny` behaves much the same as `strictlyNoneOf`:

```
ghci> strictlyAny (One undefined)
None
```

## `newtype` versus `data`

The constructors of a `newtype` are not "true" constructors, from the perspective of `Prelude.seq`.
Consider,

``` {#NewOne .haskell}
newtype NewOne = NewOne Integer
  deriving Show
```

```
ghci> strictlyAny (NewOne undefined)
*** Exception: Prelude.undefined
```

## Strict constructors

The argument of a constructor can be made strict by giving it a `!` prefix,

``` {#One1 .haskell}
data One1 = One1 !Integer
  deriving Show

noneOf1 :: One1 -> None
noneOf1 _ = None

strictlyNoneOf1 :: One1 -> None
strictlyNoneOf1 (One1 _) = None
```

Strict arguments are evaluated to weak head normal form when the constructor is evaluated, so that

```
ghci> noneOf1 (One1 undefined)
None
ghci> strictlyNoneOf1 (One1 undefined)
*** Exception: Prelude.undefined
ghci> strictlyAny (One1 undefined)
*** Exception: Prelude.undefined
```

## Lazy patterns

A pattern can be made lazy by giving it a `~` prefix,

``` {#maybeNoneOf .haskell}
maybeNoneOf :: Bool -> One -> Maybe Integer
maybeNoneOf doMatch ~(One x)
  | doMatch   = Just x
  | otherwise = Nothing
```

A lazy pattern match defers evaluation until one of the matched names is demanded,

```
ghci> maybeNoneOf False undefined
Nothing
ghci> maybeNoneOf True undefined
Just *** Exception: Prelude.undefined
```

The last line is not a typo!
`Just` is a lazy constructor, so that the `x` matched by `~(One x)` is not demanded until GHCi prints the result.
The interpreter proceeds as far as it can—even `show`-ing the constructor name—before `x` is finally demanded.
