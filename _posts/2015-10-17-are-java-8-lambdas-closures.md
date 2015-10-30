---
layout: post
published: true
title: Are Java 8 Lambdas Closures?
---

Based on what I've heard, I was surprised to discover that the short answer
is "yes, with a caveat that, after explanation, isn't really a caveat." So,
yes.

For the longer answer, we must first explore the question of "why, again,
are we doing all this?"

## Abstraction over Behavior

The simplest way to look at the need for lambdas is that they describe
*what* computation should be performed, rather than *how* it should be
performed. Traditionally, we've used *external iteration*, where we specify
exactly how to step through a sequence and perform operations:

```java
// InternalVsExternalIteration.java
import java.util.*;

interface Pet {
    void speak();
}

class Rat implements Pet {
    public void speak() { System.out.println("Squeak!"); }
}

class Frog implements Pet {
    public void speak() { System.out.println("Ribbit!"); }
}

public class InternalVsExternalIteration {
    public static void main(String[] args) {
        List<Pet> pets = Arrays.asList(new Rat(), new Frog());
        for(Pet p : pets) // External iteration
            p.speak();
        pets.forEach(Pet::speak); // Internal iteration
    }
}
```

The `for` loop represents external iteration and specifies exactly how it
is done. This kind of code is redundant, and duplicated throughout our
programs. With the `forEach`, however, we tell it to call `speak` (here,
using a method reference which is more succinct than a lambda) for each
element, but we don't have to specify how the loop works. The iteration is
handled internally, inside the `forEach`.

This "what not how" is the basic motivation for lambdas. But to understand
closures, we must look more deeply, into the motivation for functional
programming itself.

## Functional Programming

Lambdas/Closures are there to aid functional programming. Java 8 is not
suddenly a functional programming language, but (like Python) now has some
support for functional programming on top of its basic object-oriented
paradigm.

The core idea of functional programming is that you can create and
manipulate functions, including creating functions at runtime. Thus,
functions become another thing that your programs can manipulate (instead
of just data). This adds a lot of power to programming.

A *pure* functional programming language includes other restrictions,
notably data invariance. That is, you don't have variables, only
unchangeable values. This sounds overly constraining at first (how can you
get anything done without variables?) but it turns out that you can
actually accomplish everything with values that you can with variables (you
can prove this to yourself using Scala, which is itself *not* a pure
functional language but has the option to use values everywhere). Invariant
functions take arguments and produce results without modifying their
environment, and thus are much easier to use for parallel programming
because an invariant function doesn't have to lock shared resources.

Before Java 8, the only way to create functions at runtime was through
bytecode generation and loading (which quite messy and complex).

Lambdas provide two basic features:

1. More succinct function-creation syntax.

2. The ability to create functions at runtime, which can then be
passed/manipulated by other code.

Closures concern this second issue.

## What is a Closure?

A closure uses variables that are outside of the function scope. This is
not a problem in traditional procedural programming -- you just use the
variable -- but when you start producing functions at runtime it does
become a problem. To see the issue, I'll start with a Python example. Here,
`make_fun()` is creating and returning a function called `func_to_return`,
which is then used by the rest of the program:

```python
# Closures.py

def make_fun():
    # These are outside the scope of the returned function:
    alist = [] # this creates an empty list
    n = 0

    def func_to_return(arg):
        nonlocal n
        # Without 'nonlocal' n += arg produces:
        # local variable 'n' referenced before assignment
        n += arg
        alist.append(arg)
        return alist, n  # Returns a tuple

    return func_to_return

x = make_fun()
y = make_fun()

for i in range(10):
    list_x, xn = x(i)
print(list_x, xn)

for i in range(10, 20):
    list_y, yn = y(i)
print(list_y, yn)

""" Output:
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9] 45
[10, 11, 12, 13, 14, 15, 16, 17, 18, 19] 145
"""
```

Notice that `func_to_return` manipulates two fields that are outside its
scope: `n` and `alist`. The `nonlocal` declaration is required because of
the way Python works: if you just start using a variable, it assumes that
variable is local. Here, the compiler (yes, Python has a compiler and yes,
it actually does some -- admittedly quite limited -- static type checking)
sees that `n += arg` uses `n` which, within the scope of `func_to_return`,
hasn't been initialized, and generates an error message. But if we say that
`n` is `nonlocal`, Python realizes that we're using the `n` that's defined
outside the function scope, and which *has* been initialized, so it's
OK. And even without the `nonlocal` declaration, it also recognizes that
`alist` is a reference to a `list` and allows it (built-in collections get
special treatment).

Now we encounter the problem: if we simply return `func_to_return`, what
happens to `n` and `alist`, which are outside the scope of
`func_to_return`? Ordinarily we'd expect those elements to go out of scope
and become unavailable, but if that happens then `func_to_return` won't
work. In order to support dynamic creation of functions, `func_to_return`
must "close over" and keep alive both `n` and `alist` when it's returned,
and that's what happens -- thus the term *closure*.

To test `make_fun()`, we call it twice and capture the resulting function
in `x` and `y`. Because `func_to_return` produces a tuple, we unpack the
tuple into `list_x` and `xn` by saying `list_x, xn = x(i)`. The fact that
`x` and `y` produce completely different results shows that each call to
`make_fun()` produces a completely independent `func_to_return` with
completely independent closed-over storage for `n` and `alist`.

## Java 8 Lambdas

Now let's see what the same example looks like with lambdas. We'll start
with *just* the `List` outside the function scope:

```java
// AreLambdasClosures.java
import java.util.function.*;
import java.util.*;

public class AreLambdasClosures {
    public Function<Integer, List<Integer>> make_fun() {
        // Outside the scope of the returned function:
        List<Integer> alist = new ArrayList<>();
        return arg -> { alist.add(arg); return alist; };
    }
    public void try_it() {
        Function<Integer, List<Integer>> x = make_fun();
        Function<Integer, List<Integer>> y = make_fun();
        List<Integer> list_x = null;
        for(int i = 0; i < 10; i++)
            list_x = x.apply(i);
        System.out.println(list_x);
        List<Integer> list_y = null;
        for(int i = 10; i < 20; i++)
            list_y = y.apply(i);
        System.out.println(list_y);
    }
    public static void main(String[] args) {
        new AreLambdasClosures().try_it();
    }
}
/* Output:
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
[10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
*/
```

This is a fairly straightforward translation of the Python version:

1. `make_fun()` returns a `Function` object, which takes an `Integer`
argument and returns a `List<Integer>`. `x` and `y` require identical
`Function` definitions.

2. I use shorthands in the lambda: no parentheses around the single
argument, and type inference.

The results are the same as the Python version, showing that Java 8 lambdas
do indeed close over the elements in their surrounding *lexical scope*, and
thus lambdas are closures.

There is a difference from the Python version, however. Lambdas only close
over *values* (references), but not variables. So when we try the closure
over `n`, we get:

```java
// AreLambdasClosures2.java
import java.util.function.*;

public class AreLambdasClosures2 {
    public Consumer<Integer> make_fun2() {
        Integer n = 0; // Also true for int n
        return arg -> n += arg;
        // error: local variables referenced
        // from a lambda expression must
        // be final or effectively final
    }
}
```

You might expect `Integer` to produce a reference to a heap object, so is
this a mistake or a bug? No: both `int` and `Integer` get special treatment
as stack-based variables (the compiler has the option of putting things on
the stack, and apparently it will tell us, as it does above, when this
happens). And the argument goes that in a pure functional language there
are no variables, only values, so it's unreasonable to expect lambdas to
close over variables.

How do we fix the problem? By forcing the object to be on the heap. For
example:

```java
// AreLambdasClosures3.java
import java.util.function.*;

class myInt {
    int i = 0;
}

public class AreLambdasClosures3 {
    public Consumer<Integer> make_fun2() {
        myInt n = new myInt();
        return arg -> n.i += arg;
    }
}
```

This compiles without complaint.

So Java 8 lambdas have closure behavior -- and will tell us when it
doesn't, so we can fix the problem. For me, it accomplishes the desired
goal: it's now possible to create functions dynamically.

I asked why the feature wasn't just called "closures" instead of "lambdas,"
since it has the characteristics of a closure? The answer I got was that
closure is a loaded and ill defined term, and was likely to create more
heat than light. When someone says "real closures," it too often means
"what closure meant in the first language I encountered with something
called closures."

I don't see an OO versus FP (functional programming) debate here; that is
not my intention. Indeed, I don't really see a "versus" issue. OO is good
for abstracting over data (and just because Java forces objects on you
doesn't mean that objects are the answer to every problem), while FP is
good for abstracting over behavior. Both paradigms are useful, and mixing
them together has been even more useful for me, both in Python and now in
Java 8. (I have also recently been using Pandoc, written in the pure FP
Haskell language, and I've been extremely impressed with that, so it
seems there is a valuable place for pure FP languages as well).
