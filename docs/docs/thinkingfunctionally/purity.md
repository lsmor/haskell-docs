---
comments: true
---


Given a function, for example a function `f` of type `Int -> Bool`, the effect of calling the function on an `Int` must be nothing other than returning a `Bool`.

For example, `f` cannot read a file, write to command line, or start a thread, as a *side effect* of being called.

Of course, it is possible to do all these things, but requires a change to the type:

```hs
add1AndPrint :: Int -> IO Bool
add1AndPrint x = (print x) >> (return (x > 1)) -- (1)!
```

1. Fewer brackets are fine (`#!hs add1AndPrint x = print x >> return (x > 1)`), and are here just for clarity.

This applies not just to functions, but all values:

```hs title="repl example"
> let boolean = print "hello" >> return True
> :t boolean
boolean :: IO Bool -- (1)!
> boolean
"hello"
True
```

1. `boolean` cannot have type `Bool` - because its definition calls `print`, it has to mark in its type the fact that it involves the operation of printing.

## The benefit of purity

Because of purity, a function will give the same answer no matter where or when it is called, as long as the input is the same. This makes parallelism easy, often almost trivial:

=== "Run tests sequentially"

    ```hs
    main :: IO ()
    main = hspec $ do
        ... tests ...
    ```

=== "Run tests in parallel"

    ```hs
    main :: IO ()
    main = hspec $ parallel $ do
        ... tests ...
    ```

Purity also lends itself to modular code, and easy refactoring. Consider:

```hs
graphicalUserInterface = runWith complexFunction
    where 
        complexFunction :: UserInput -> Picture
        complexFunction = ...

        runWith = ... -- e.g., a handler function
```

Suppose we want to replace `complexFunction` with `simpleFunction`, which also has the type 
`#!hs UserInput -> Picture`.


Because Haskell is pure (but see [caveats](/thinkingfunctionally/purity/#caveats)) and so `complexFunction` is not creating/mutating global variables, or opening or closing files, we can be confident that there will be no unexpected implications of making the change, such as a subtle error with changed variable assignment, when `runWith` takes `complexFunction` as input. 

## Equational reasoning

Because of purity, you may always replace a function call in Haskell with its resulting value. For instance, if the value of `positionOfWhiteKing chessboard` is `"a4"`, then this

```hs
exampleProgram = someFunction (positionOfWhiteKing chessboard)
```

is equivalent to

```hs
exampleProgram = someFunction "a4"
```

!!! Tip
    Use this fact to understand complex programs, by substituting complex expressions for their values:

    ```haskell
    data Piece = Bishop | Rook | King
    take 2 [Bishop, Rook, Bishop]
    ```

    To work out what this does, we consult the definition of `take` (shown here with some aesthetic simplifications for clarity):

    ```hs linenums="1"
    take 0 ls = []
    take _ [] = []
    take n (firstElem : rest) = firstElem : take (n-1) rest
    ```

    Following this definition, we replace `take 2 [1,2,3]` (or more explicitly, `#1hs take 2 (1 : [2,3])`) with the pattern that it matches:

    ```haskell
        take 2 (Bishop : [Rook, Bishop]) 
        = Bishop : take (2-1) [Rook, Bishop] 
        = Bishop : take 1 (Rook : [Bishop])
    ```

    We can continue in this vein, repeatedly consulting the definition of `take`:

    ```haskell
        = Bishop : take 1 (Rook : [Bishop])
        = Bishop : (Rook : take (1 - 1) [Bishop])
        = Bishop : (Rook : take 0 [Bishop]) 
        = Bishop : (Rook : [])
        = [Bishop, Rook]
    ```

    This technique is **always applicable**, no mater how complex the program.

## Totality

A function is total if it returns a result for any possible input. For example, `head` is not total:

```hs title="repl example"
> head [1,2]
1
> head []
*** Exception: Prelude.head: empty list
```

In Haskell, non-total (aka *partial*) functions are permitted, although they are discouraged. Functions may be partial by throwing an runtime error on some inputs (like `head`), or by running indefinitely, (like `last [1..]`). Haskell will generally warn you about the first kind, but not the second, since it is harder to detect.

### Caveats

Haskell allows a backdoor, mainly useful for debugging. 

This is the ability for functions to throw an "unsafe" error:

```hs title="repl example"
let x = undefined
> :t x
x :: a
> x
"*** Exception: Prelude.undefined..."
```

`undefined` has the type `forall a. a`, so it can appear anywhere in a program and assume the correct type (see here for [more details on how universal quantification works](/basics/types/#how-to-use)). 

As such, it is useful as a "to do" marker (see [type driven development](/thinkingfunctionally/typeinference/#type-driven-development)).


