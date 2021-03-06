---
type: doc
layout: reference
category: "Syntax"
title: "Inline Functions"
---

# Inline Functions

Using [higher-order functions](lambdas.html) imposes certain runtime penalties: each function is an object, and it captures a closure,
i.e. those variables that are accessed in the body of the function.
Memory allocations (both for function objects and classes) and virtual calls introduce runtime overhead.

But it appears that in many cases this kind of overhead can be eliminated by inlining the function literals.
The functions shown above are good examples of this situation. I.e., the `lock()` function could be easily inlined at call-sites.
Consider the following case:

``` kotlin
lock(l) {foo()}
```

Instead of creating a function object for the parameter and generating a call, the compiler could emit the following code

``` kotlin
lock.lock()
try {
  foo()
}
finally {
  lock.unlock()
}
```

Isn't it what we wanted from the very beginning?

To make the compiler do this, we need to annotate the `lock()` function with the `inline` annotation:

``` kotlin
inline fun lock<T>(lock: Lock, body: () -> T): T {
  // ...
}
```

Inlining may cause the generated code to grow, but if we do it in a reasonable way (do not inline big functions)
it will pay off in performance, especially at "megamorphic" call-sites inside loops.

## \[noinline\]

In case you want only some of the lambdas passed to an inline function to be inlined, you can mark some of your function
parameters with `[noinline]` annotation:

``` kotlin
inline fun foo(inlined: () -> Unit, [noinline] notInlined: () -> Unit) {
  // ...
}
```

Inlinable lambdas can only be called inside the inline functions or passed as inlinable arguments,
but `[noinline]` one can be manipulated in any way we like: stored in fields, passed around etc.

Note that if an inline function has no inlinable function parameters and no
[reified type parameters](#reified-type-parameters), the compiler will issue a warning, since inlining such functions is
 very unlikely to be beneficial (you can suppress the warning if you are sure the inlining is needed).

## Non-local returns

In Kotlin, we can only use a normal, unqualified `return` to exit a named function..
This means that to exit a lambda, we have to use a [label](returns.html#return-at-labels), and a bare `return` is forbidden
inside a lambda, because a lambda can not make the enclosing function return:

``` kotlin
fun foo() {
  ordinaryFunction {
     return // ERROR: can not make `foo` return here
  }
}
```

But if the function the lambda is passed to is inlined, the return can be inlined as well, so it is allowed:

``` kotlin
fun foo() {
  inlineFunction {
    return // OK: the lambda is inlined
  }
}
```

Such returns (located in a lambda, but exiting the enclosing function) are called *non-local* returns. We are used to
this sort of constructs in loops, which inline functions often enclose:

``` kotlin
fun hasZeros(ints: List<Int>): Boolean {
  ints.forEach {
    if (it == 0) return true // returns from hasZeros
  }
  return false
}
```

> `break` and `continue` are not yet available in inlined lambdas, but we are planning to support them too

## Reified type parameters

Sometimes we need to access a type passed to us as a parameter:

``` kotlin
fun <T> TreeNode.findParentOfType(clazz: Class<T>): T? {
    var p = parent
    while (p != null && !clazz.isInstance(p)) {
        p = p?.parent
    }
    [suppress("UNCHECKED_CAST")]
    return p as T
}
```

Here, we walk up a tree and use reflection to check if a node has a certain type.
It’s all fine, but the call site is not very pretty:

``` kotlin
myTree.findParentOfType(javaClass<MyTreeNodeType>())
```

What we actually want is simply pass a type to this function, i.e. call is like this:

``` kotlin
myTree.findParentOfType<MyTreeNodeType>()
```

To enable this, inline functions support *reified type parameters*, so we can write something like this:

``` kotlin
inline fun <reified T> TreeNode.findParentOfType(): T? {
    var p = parent
    while (p != null && p !is T) {
        p = p?.parent
    }
    return p as T
}
```

We qualified the type parameter with the reified modifier, now it’s accessible inside the function,
almost as if it were a normal class. Since the function is inlined, no reflection is needed, normal operators like `!is`
and `as` are working now. Also, we can call it as mentioned above: `myTree.findParentOfType<MyTreeNodeType>()`.

Though reflection may not be needed in many cases, we can still use it with a reified type parameter: `javaClass()` gives us access to it:

``` kotlin
inline fun methodsOf<reified T>() = javaClass<T>().getMethods()

fun main(s: Array<String>) {
  println(methodsOf<String>().joinToString("\n"))
}
```

Normal functions (not marked as inline) can not have reified parameters.
A type that does not have a run-time representation (e.g. a non-reified type parameter or a fictitious type like `Nothing`)
can not be used as an argument for a reified type parameter.

For a low-level description, see the [spec document](https://github.com/JetBrains/kotlin/blob/master/spec-docs/reified-type-parameters.md).
