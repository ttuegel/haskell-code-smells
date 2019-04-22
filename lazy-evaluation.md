% Demystifying lazy evaluation
% Thomas Tuegel

...

## Example

Let me begin by defining some types, so that I have some constructors to play with:

``` {#one-none .haskell}
-- | One contains one Integer.
data One = One Integer

-- | None contains no Integers.
data None = None
```

I can also define a simple function to convert `One` into `None`:

``` {#noneOf .haskell}
noneOf :: One -> None
noneOf _ = None
```

Well, that was easy!
The type `One` seems to contain one `Integer`, that is,
if I have an `Integer` I can construct `One`, and then I can call `noneOf`:

```
ghci> noneOf (One 2)
None
```
