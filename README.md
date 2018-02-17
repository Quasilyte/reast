# reast

Re-AST: rewrite Go AST to functionally equivalent, but syntactically simpler form.

## Go AST simplifications rules

Problem: `go/ast` is not very convenient for certain kinds of code manipulations.  
The `SSA`-like formats can be too low-level
and cumbersome to work with if it's advantages are not required.

This file describes how Go AST can be re-written in a way that respects evaluation order,  
semantics and functional behaviour, yet is simpler to process.

One of the most flexible construction, assignment and it's
forms (like `var` declaration + initilization) is re-written
in multiple steps. When combined, they lead simpler structure.

This process mutates the initial AST, it is not suitable
for tools those output is intended to be human-readable,
as close to original as possible, form.
It is suitable for different kinds of code transcompilers, for example.

Transformations require type information from `go/types`.
Some rules may modify provided `types.Info` object to
make output AST types discoverable.

Note that for compilation-like tasks this can't replace dedicated 
MIR/LIR (or just IR). 
It makes IR construction easier though.

Some rules depend on each other.
Where possible, dependent rules are listed below rules
they depend on. They may also list Go code which is a
result of previous rule application.

There are issues that are not addressed directly, like:
* Comments and their positions after transformations.
* Source code positions for new/modified nodes.

## Rule 1. No auto dereference shorthands

From [index expressions](https://golang.org/ref/spec#Index_expressions):
> For a of pointer to array type: a[x] is shorthand for (*a)[x].

```go
// a1 *T[N]
// a2 *T[N]
a1[i] = a2[j]       // Before
(*a1)[i] = (*a2)[j] // After
```

From [selectors](https://golang.org/ref/spec#Selectors) section:
> For a value x of type T or *T where T is not a pointer or interface type, 
> x.f denotes the field or method at the shallowest depth in T where there is such an f.
> As an exception, if the type of x is a defined pointer type 
> and (*x).f is a valid selector expression denoting a field (but not a method), 
> x.f is shorthand for (*x).f. 

```go
// s1 *struct{x T}
// s2 *struct{x T}
s1.x = s2.x       // Before
(*s1).x = (*s2).x // After
```

**Reason**: makes selector expression and index expression less ambiguous.

## Rule 2. Explicit type in var declarations

```go
// Before:
var x = 10
var y = new([]int)
// After:
var x int = 10
var y *[]int = new([]int)
```

**Reason**: makes `var` declarations more uniform.

## Rule 3. Explicit default initialization

With this, `var` declaration always has initializer expression.

```go
// Before:
var x int
var y T
var z *T
// After:
var x int = 0
var y T = T{}
var z *T = nil
```

**Reason**: makes `var` declarations more uniform.

## Rule 4. Split declarations

Makes each `var` declaration introduce exactly one name.

```go
// Before:
var x, y T = T{}, T{}
// After:
var x T = T{}
var y T = T{}
```

This also works at value-spec level:

```go
// Before:
var (
    x, y int = 1, 2
    z T
)
// After:
var x int = 1
var y int = 2
var z T = T{}
```

**Reason**: makes multi-value context, where `len(lhs)!=len(rhs)`, 
applicable for tuple assignments only.

## Rule 5. Explicit value discard.

Removes all `ast.StmtExpr` with `ast.AssignStmt`.

```go
// Before:
foo(x)
bar()
// After:
_ = foo(x)
_, _ = bar()
```

**Reason**: merging of two functionally identical forms into one.
