[![Build
Status](https://travis-ci.org/andrewcooke/ParserCombinator.jl.png)](https://travis-ci.org/andrewcooke/ParserCombinator.jl)
[![Coverage Status](https://coveralls.io/repos/andrewcooke/ParserCombinator.jl/badge.svg)](https://coveralls.io/r/andrewcooke/ParserCombinator.jl)
[![ParserCombinator](http://pkg.julialang.org/badges/ParserCombinator_release.svg)](http://pkg.julialang.org/?pkg=ParserCombinator&ver=release)



# ParserCombinator

* [Example](#example)
* [Install](#install)
* [Manual](#manual)
* [Design](#design)

A parser combinator library for Julia that tries to strike a balance between
being both (moderately) efficient and simple (for the end-user and the
maintainer).  It is similar to parser combinator libraries in other languages
(eg. Haskell's Parsec or Python's pyparsing) - it can include caching
(memoization) but does not handle left-recursive grammars.

ParserCombinator can parse any iterable type (not just strings).

## Example

```julia
using ParserCombinator


# the AST nodes we will construct, with evaluation via calc()

abstract Node
==(n1::Node, n2::Node) = n1.val == n2.val
calc(n::Float64) = n
type Inv<:Node val end
calc(i::Inv) = 1.0/calc(i.val)
type Prd<:Node val end
calc(p::Prd) = Base.prod(map(calc, p.val))
type Neg<:Node val end
calc(n::Neg) = -calc(n.val)
type Sum<:Node val end
calc(s::Sum) = Base.sum(map(calc, s.val))


# the grammar

sum = Delayed()
val = S"(" + sum + S")" | PFloat64()

neg = Delayed()             # allow multiple negations (eg ---3)
neg.matcher = val | (S"-" + neg > Neg)

mul = S"*" + neg
div = S"/" + neg > Inv
prd = neg + (mul | div)[0:end] |> Prd

add = S"+" + prd
sub = S"-" + prd > Neg
sum.matcher = prd + (add | sub)[0:end] |> Sum

all = sum + Eos()


# and test 

# this prints 2.5
calc(parse_one("1+2*3/4")[0])

# this prints [Sum([Prd([1.0]),Prd([2.0])])]
parse_one("1+2")
```

Some explanation of the above:

* I used rather a lot of "syntactic sugar".  You can use a more verbose,
  "parser combinator" style if you prefer.  For example, `Seq(...)` instead of
  `+`, or `App(...)` instead of `>`.

* The matcher `S"xyz"` matches and then discards the string `"xyz"`.

* Every matcher returns a list of matched values.  This can be an empty list
  if the match succeeded but matched nothing.

* The operator `+` matches the expressions to either side and appends the
  resulting lists.  Similarly, `|` matches one of two alternatives.

* The operator `|>` calls the function to the right, passing in the results
  from the matchers on the left.

* `>` is similar to `|>` but interpolates the arguments (ie uses `...`).  So
  instead of passing a list of values, it calls the function with multiple
  arguments.

* `Delayed()` lets you define a loop in the grammar.

* The syntax `[0:end]` is a greedy repeat of the matcher to the left.  An
  alternative would be `Star(...)`, while `[3:4]` would match only 3 or 4
  values.

And it supports packrat parsing too (more exactly, it can memoize results to
avoid repeating matches).

Still, for large parsing tasks (eg parsing source code for a compiler) it
would probably be better to use a wrapper around an external parser generator,
like Anltr.

## Install

```julia
julia> Pkg.add("ParserCombinator")
```

## Manual

In what follows, remember that the power of parser combinators comes from how
you combine these.  They can all be nested, refer to each other, etc etc.

* [Basic Matchers](#basic-matchers)
  * [Equality](#equality)
  * [Sequences](#sequences)
  * [Empty Values](#empty-values)
  * [Alternates](#alternates)
  * [Regular Expressions](#regular-expressions)
  * [Repetition](#repetition)
  * [Full Match](#full-match)
  * [Transforms](#transforms)
  * [Lookahead And Negation](#lookahead-and-negation)
* [Other](#other)
  * [Coding Style](#coding-style)
  * [Adding Matchers](#adding-matchers)
  * [More Information](#more-information)

### Basic Matchers

#### Equality

```julia
julia> parse_one("abc", Equal("ab"))
1-element Array{Any,1}:
 "ab"

julia> parse_one("abc", Equal("abx"))
ERROR: ParserCombinator.ParserException("cannot parse")
```

This is so common that there's a corresponding [string
literal](http://julia.readthedocs.org/en/latest/manual/strings/#non-standard-string-literals)
(it's "s" for "string").

```julia
julia> parse_one("abc", s"ab")
1-element Array{Any,1}:
 "ab"
```

#### Sequences

Matchers return lists of values.  Multiple matchers can return lists of lists,
or the results can be "flattened" a level (usually more useful):

```julia
julia> parse_one("abc", Series(Equal("a"), Equal("b")))
2-element Array{Any,1}:
 "a"
 "b"

julia> parse_one("abc", Series(Equal("a"), Equal("b"); flatten=false))
2-element Array{Any,1}:
 Any["a"]
 Any["b"]

julia> parse_one("abc", Seq(Equal("a"), Equal("b")))
2-element Array{Any,1}:
 "a"
 "b"

julia> parse_one("abc", And(Equal("a"), Equal("b")))
2-element Array{Any,1}:
 Any["a"]
 Any["b"]

julia> parse_one("abc", s"a" + s"b")
2-element Array{Any,1}:
 "a"
 "b"

julia> parse_one("abc", s"a" & s"b")
2-element Array{Any,1}:
 Any["a"]
 Any["b"]
```

Where `Series()` is implemented as `Seq()` or `And()`, depending on the value
of `flatten` (default `true`).

#### Empty Values

Often, you want to match something but then discard it.  An empty (or
discarded) value is an empty list.  This may help explain why I said
flattening lists was useful above.

```julia
julia> parse_one("abc", And(Drop(Equal("a")), Equal("b")))
2-element Array{Any,1}:
 Any[]   
 Any["b"]

julia> parse_one("abc", Seq(Drop(Equal("a")), Equal("b")))
1-element Array{Any,1}:
 "b"

julia> parse_one("abc", ~s"a" + s"b")
1-element Array{Any,1}:
 "b"

julia> parse_one("abc", S"a" + s"b")
1-element Array{Any,1}:
 "b"
```

Note the `~` (tilde / home directory) and capital `S` in the last two
examples, respectively.

#### Alternates

```julia
julia> parse_one("abc", Alt(s"x", s"a"))
1-element Array{Any,1}:
 "a"

julia> parse_one("abc", s"x" | s"a")
1-element Array{Any,1}:
 "a"
```

#### Regular Expressions

```julia
julia> parse_one("abc", Pattern(r".b."))
1-element Array{Any,1}:
 "abc"

julia> parse_one("abc", p".b.")
1-element Array{Any,1}:
 "abc"

julia> parse_one("abc", P"." + p"b.")
1-element Array{Any,1}:
 "bc"
```

As with equality, a capital prefix to the string literal ("p" for "pattern" by
the way) implies that the value is dropped.

#### Repetition

```julia
julia> parse_one("abc", Repeat(p"."))
3-element Array{Any,1}:
 "a"
 "b"
 "c"

julia> parse_one("abc", Repeat(p".", 2))
2-element Array{Any,1}:
 "a"
 "b"

julia> collect(parse_all("abc", Repeat(p".", 2, 3)))
2-element Array{Any,1}:
 Any["a","b","c"]
 Any["a","b"]    

julia> parse_one("abc", Repeat(p".", 2; flatten=false))
2-element Array{Any,1}:
 Any["a"]
 Any["b"]

julia> collect(parse_all("abc", Repeat(p".", 0, 3)))
4-element Array{Any,1}:
 Any["a","b","c"]
 Any["a","b"]    
 Any["a"]        
 Any[]           

julia> collect(parse_all("abc", Repeat(p".", 0, 3; greedy=false)))
4-element Array{Any,1}:
 Any[]           
 Any["a"]        
 Any["a","b"]    
 Any["a","b","c"]
```

You can also use `Depth()` and `Breadth()` for greedy and non-greedy repeats
directly (but `Repeat()` is more readable, I think).

The sugared version looks like this:

```julia
julia> parse_one("abc", p"."[1:2])
2-element Array{Any,1}:
 "a"
 "b"

julia> parse_one("abc", p"."[1:2,:?])
1-element Array{Any,1}:
 "a"

julia> parse_one("abc", p"."[1:2,:&])
2-element Array{Any,1}:
 Any["a"]
 Any["b"]

julia> parse_one("abc", p"."[1:2,:&,:?])
1-element Array{Any,1}:
 Any["a"]
```

Where the `:?` symbol is equivalent to `greedy=false` and `:&` to
`flatten=false` (compare with `+` and `&` above).

There are also some well-known special cases:

```julia
julia> collect(parse_all("abc", Plus(p".")))
3-element Array{Any,1}:
 Any["a","b","c"]
 Any["a","b"]    
 Any["a"]        

julia> collect(parse_all("abc", Star(p".")))
4-element Array{Any,1}:
 Any["a","b","c"]
 Any["a","b"]    
 Any["a"]        
 Any[]           
```

#### Full Match

To ensure that all the input is matched, add `Eos()` to the end of the
grammar.

```julia
julia> parse_one("abc", Equal("abc") + Eos())
1-element Array{Any,1}:
 "abc"

julia> parse_one("abc", Equal("ab") + Eos())
ERROR: ParserCombinator.ParserException("cannot parse")
```

#### Transforms

Use `App()` or `>` to pass the current results to a function (or datatype
constructor) as individual values.

```julia
julia> parse_one("abc", App(Star(p"."), tuple))
1-element Array{Any,1}:
 ("a","b","c")

julia> parse_one("abc", Star(p".") > string)
1-element Array{Any,1}:
 "abc"
```

The action of `Appl()` and `|>` is similar, but everything is passed as a
single argument (a list).

```julia
julia> type Node children end

julia> parse_one("abc", Appl(Star(p"."), Node))
1-element Array{Any,1}:
 Node(Any["a","b","c"])

julia> parse_one("abc", Star(p".") |> x -> map(uppercase, x))
3-element Array{Any,1}:
 "A"
 "B"
 "C"
```

#### Lookahead And Negation

Sometimes you can't write a clean grammar that just consumes data: you need to
check ahead to avoid something.  Or you need to check ahead to make sure
something works a certain way.

```julia
julia> parse_one("12c", Lookahead(p"\d") + PInt() + Dot())
2-element Array{Any,1}:
 12   
   'c'

julia> parse_one("12c", Not(Lookahead(p"[a-z]")) + PInt() + Dot())
2-element Array{Any,1}:
 12   
   'c'
```

More generally, `Not()` replaces any match with failure, and failure with an
empty match (ie the empty list).


### Other

#### Coding Style

Don't go reinventing regexps.  The built-in regexp engine is way, way more
efficient than this library could ever be.  So call out to regexps liberally.
The `p"..."` syntax makes it easy.

Drop stuff you don't need.

Transform things into containers so that your result has nice types.  Look at
how the [example](#example) works.

#### Adding Matchers

First, are you sure you need to add a matcher?  You can do a *lot* with
[transforms](#transforms).

If you do, here are some places to get started:

* `Equal()` in [matchers.jl](src/matchers.jl) is a great example for
  something that does a simple, single thing, and returns success or failure.

* Most matchers that call to a sub-matcher can be implemented as transforms.
  But if you insist, there's an example in [case.jl](test/case.jl).

* If you want to write complex, stateful matchers then I'm afraid you're going
  to have to learn from the code for `Repeat()` and `Series()`.  Enjoy!

#### More Information

For more details, I'm afraid your best bet is the source code:

* [types.jl](src/types/jl) introduces the types use throughout the code

* [matchers.jl](src/matchers.jl) defines things like `Seq` and `Repeat`

* [sugar.jl](src/sugar.jl) adds `+`, `[...]` etc

* [extras.jl](src/extras.jl) has parsers for Int, Float, etc

* [parsers.jl](src/parsers.jl) has more info on creating the `parse_one` and
  `parse_all` functions

* [transforms.jl](src/trasnforms.jl) defines how results can be manipulated

* [tests.jl](test/tests.jl) has a pile of one-liner tests that might be useful

* [debug.jl](test/debug.jl) shows how to enable debug mode

## Design

### Overview

Julia does not support tail call recursion, and is not lazy, so a naive
combinator library would be limited by recursion depth and strict evaluation
(no caching).  Instead, the "combinators" in ParserCombinator construct a tree
that describes the grammar, and which is "interpreted" during parsing, by
dispatching functions on the tree nodes.  The traversal over the tree is
implemented via trampolining, with an optional cache to avoid repeated
evaluation (and, possibly, in the future, detect left-recursive grammars).

The advantages of this approach are:

  * Recursion is reduced (repetition and sequential matching are iterative,
    but the grammar itself can still contain loops).

  * Method dispatch on node types leads to idiomatic Julia code (well,
    as idiomatic as possble, for what is a glorified state machine).

  * Caching can be isolated to within the trampoline (and has access to exact,
    explicit state for the matcher, which makes keying the lookup trivial).

  * The trampoline also dispatches on Config type, which means that new
    behaviours (like automatic removal of spaces) can be added to the parser
    "globally" in an idiomatic way.

It would also have been possible to use Julia tasks (coroutines).  I avoided
this approach because my understanding is (although I have no proof) that
tasks are significantly "heavier".

Note - this is not a magic bullet.  There is still a "stack" in the trampoline
(effectively a "continuation"), although in user-space and significantly more
compact.

### Matcher Protocol

Below I try to give a "high level picture" of how evaluation proceeds.  For
the full details, please see the source.

Consider the matchers `Parent` and `Child` which might be used in some way to
parse "hello world":

```
immutable Child<:Matcher
  text
end

immutable Parent<:Matcher
  child1::Child
  child2::Child  
end

# this is a very vague example, don't think too hard about what it means
hello = Child("hello")
world = Child("world")
hello_world_grammar = Parent(hello, world)
```

Each matcher has some associated types that store state (the matchers
instances themselves describe only the *static* grammar; the state describes
the associated state during matching and backtracking).  Two states, `CLEAN`
and `DIRTY`, are used globally to indicate that a matcher is uncalled, or has
exhausted all matches, respectively.

Transitions between these states are made by calling two methods (one for
evaluating a match, and one for returning a result - see below).  Functions
for these methods are associated with combinations of matchers and state to
implement the necessary logic.

These transitions are triggered via `Message` types - one matching each
method.  So a method function associated with a matcher (and state) can return
one of the messages and the trampoline will call the corresponding code for
the target.

I've tried to be exact, but that sounds horribly opaque.  In practice, it's
quite simple.  For example:

```
function execute(k::Config, p::Parent, s::ParentState, iter)
  # the parent wants to match the source text at offset iter against child1
  Execute(p, s, p.child1, ChildStateStart(), iter)
end

function execute(k::Config, c::Child, s::ChildStateStart, iter)
  # the above returns an Execute instance, which tells the trampoline to
  # make a call here, where we check if the text matches
  if compare(c.text, k.source[iter:])
    Response(ChildStateSucceeded(), iter, Success(c.text))
  else
    Response(ChildStateFailed(), iter, FAILURE)
  end
end

function response(k::Config, p::Parent, s::ParentState, t::ChildState, iter, result::Success)
  # the Response message containing Success above triggers a call here, where
  # we do something with the result (like save it in the ParentState)
  ...
  # and then perhaps evaluate child2...
  Execute(p, s, p.child2, ChildStateStart(), iter)
end
```

Hopefully you can see how each returned `Execute` and `Response` results in
the calling of an `execute()` or `response()` function.  In this way we can
write code that works as though it is recursive, without exhausting Julia's
stack.

Finally, to simplify caching in the trampoline, it is important that the
different matchers appear as simple calls and responses.  So internal
transitions between states in the same matcher are *not* made by messages, but
by direct calls.  This explains why, for example, you see both `Execute(...)`
and `execute(...)` in the source - the latter is an internal transition to the
given method.

### Source (Input Text) Protocol

The source text is read using the [standard Julia iterator
protocol](http://julia.readthedocs.org/en/latest/stdlib/collections/?highlight=iterator).

This has the unfortunate result that `Dot()` returns characters, not strings.
But in practice that matcher is rarely used (particularly since, with strings,
you can use regular expressions - `p"pattern"` for example).

