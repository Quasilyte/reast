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

## Rule 2. Split assign statements

When there is no multi-value context (no tuple assignment),
split parallel assignment into a series of simpler assignments.

```go
// Before:
x, y = a, b
x, y = y, x

// After:
x = a
y = b
var tmp = x // Exact tmp name can vary
x = y
y = tmp
```

The "comma, ok" assignments remain unchanged.

**Reason**: makes multi-value context dedicated to tuple assignment.

## Rule 3. Explicit type in var declarations

```go
// Before:
var x = 10
var y = new([]int)

// After:
var x int = 10
var y *[]int = new([]int)
```

**Reason**: makes `var` declarations more uniform.

## Rule 4. Explicit default initialization

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

## Rule 5. Split `var` declarations

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

**Reason**: along with rules below, makes multi-value context non-issue for `var` declaration.

## Rule 6. Replace `:=` assignments (short variable declaration)

This rule could be break down to these parts:
1. Replace simple `:=` statements with `var`.
2. Replace simple `:=` statements that re-declare names with `=`.
3. Split multi-assignment into `var` declaration plus `=`.

```go
func f1() (T1, T2)
func f2() T3
func f3() (T4, error)

// Before:
x, y := f1()

z := f2()

y, err := f3() // Note: y re-declared.

// After:
var x T1 = T1{}
var y T2 = T2{}
x, y = f1()

var z T3 = f2()

var err error = nil
y, err = f3()
```

"comma, ok" is affected, too:

```go
// Before:
x, ok := m[k1]
y, ok := m[k2]

// After:
var x T = T{}
var ok bool
x, ok = m[k1]
var y T = T{}
y, ok = m[k2]
```

**Reason**: the `:=` has much type-based and scope-based logic.
It can be replaced with `var` and `=` completely without any loss.

## Rule 7. Explicit value discard

Removes all `ast.StmtExpr` with `ast.AssignStmt`.

```go
// Before:
f1(x)
f2()
<- c
// After:
_ = f1(x)
_, _ = f2()
_ = <- c
```

**Reason**: merging of two functionally identical forms into one.

## Rule 8. Extract init statement

Moves init statement outside.
Introduces a new block to preserve proper lexical scoping context.

```go
// Before:
if err := f(); err != nil {
    // Body.
}

// After:
{
    var err error = f()
    if err != nil {
        // Body.
    }
}
```

This works for `if`, `switch`, `for` statements in a same way.

**Reason**: see [issue#1](https://github.com/Quasilyte/reast/issues/1).

## Rule 9. Inject `for` post statement

Combined with other rules, it makes only while-like `for` loop
form possible (`range` loops are a different thing).

```go
// Before:
for i := 0; i < len(xs); i++ {
    // Body.
}

// After:
{
    var i int = 0
    for i < len(xs) {
        // Body.
        i++
    }
}
```

**Reason**: see [issue#1](https://github.com/Quasilyte/reast/issues/1).

## Rule 10. Explicit `true` in `SwitchStmt`

With this, switch statement tag expression is never `nil`.

```go
// Before:
switch {
    // Cases clauses.
}

// After:
switch true {
    // Case clauses.
}
```

**Reason**: merging of two functionally identical forms into one.

## TODO

There are rules that make some constructions impossible to
express syntax errors.

For example, if `x, y := f()` is re-written into `var` and `=`,
it can no longer be used where **simple statement** is expected.
Most statements that allow (optional) initializer statement
fall into this category. 
It is a real issue that should be solved.

TODO list for rules:
* Comma, ok idiom. See [issue#2](https://github.com/Quasilyte/reast/issues/2).
* Range loops. See [issue#3](https://github.com/Quasilyte/reast/issues/3).
* Misc simplifications.
* Complete program translation examples.
