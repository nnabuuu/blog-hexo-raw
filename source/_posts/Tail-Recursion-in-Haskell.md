title: Tail Recursion in Haskell
date: 2015-10-31 22:30:36
tags: Haskell
---

Today I was solving the problem of "calculate Fibonacci sequence" in Haskell.

The normal fibonacci function is implemented as:

```haskell
fib :: Int -> Int
fib 0 = 0
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)
```
However, this implement runs very slow.

There is one possible way to optimize the code as follow:

```haskell
fib :: Int -> Int
fib n = aux n (0, 1)
        where aux n (a, b) | n == 0 = a
                          | otherwise = aux (n - 1) (b, a + b)
```

This version is a lot faster than the original version. Why?

The trick is tail recursion.

In previous code, when executing `fib n = fib (n - 1) + fib (n - 2)`, there is context switch by pushing current context into call stack in order to resume after `fib (n - 1)` and `fib (n - 2)` are calculated.

However, by using tail recursion, because our function `aux n (a, b)` directly return the result from `aux (n - 1) (b, a + b)`, program can re-use current stack space without change.

This video is a very good explanation:

[https://www.youtube.com/watch?v=L1jjXGfxozc](https://www.youtube.com/watch?v=L1jjXGfxozc)
