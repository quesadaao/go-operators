Disclaimer: **This is a hack!**

This project is an attempt to add operator overloading to the Go language. There
are very good reasons why Go doesn't support operator overloading as part of
the language and I fully support and agree with those reasons.

That being said, there are a few valid cases where it makes sense to have
operators defined for types other than the builtin types. One such valid
case, in my opinion, is for numeric types such as vectors or matrices. Of course,
Go was never designed for writing numeric heavy programs, so you could just
argue that Go is simply not the right tool for the job. Although I wouldn't
disagree with this, I still find Go an exceptionally enjoyable language and
recently I really wanted to write some vector-ish code.

go-operators allows you to add support for all Go operators to any custom
defined type, by adding appropriate methods on that type. The way this works is
that go-operators is a kind of preprocessor for your Go code which will parse
and rewrite your original source to map operators, on types which do not natively
support operators, to method calls on the operand(s) of the operator.

First, the original source is converted to an AST by using the builtin go/ast package.
Then, a patched version of go/types (available in go.tools) steps through the
AST and resolves the types of all the expressions in the AST. When an operator
is encountered which operators on non-numeric types, a method lookup is
performed on the first operand type. If the appropriate operator overloading
method can be found (with the correct operands type and return type), then the
AST node representing the operator is replaced with a method call. For binary
operators, if the first operand does not have an appropriate overloaded method,
the second operator is tested for a Pre- overload method. This way you can
overload both `v * 5.0` and `5.0 * v` (for example) on the type of `v`.

# Using go-operators
To use go-operators you will have to define special methods on your custom type.
These methods are prefixed with `Op_Multiply`, `Op_Add`, `Op_Subtract` etc.,
depending on the operator to overload. Any method which has such a prefix is
a potential candidate. Thus, if your type supports operators on various operand
types, then you can add methods such as `Op_MultiplyVec3`, `Op_MultiplyMat3`,
etc. with the appropriate argument types. The return type of the method can be
anything, but there must be extactly one return value. For overloaded binary
operators there must be exactly one argument while for overloaded unary operators
there must be exactly zero arguments.

The `go-operators` tool uses a set of conventions to parse and process files
for which to replace operators. Each file which needs to be processed should have
the `.op.go` suffix. Furthermore, these files should have the `operators` build
constraint (`// +build operators`). `go-operators` parses these files and
generates a corresponding `.go` file (stripping the `.op.go` suffix) in the
same directory, adding the `!operators` build constraint to avoid conflicts.
You run `go-operators` with the import paths of the packages you want to process.
These packages are obtained from `$GOPATH/src`.

# List of overloaded operators

| Operator | Method Prefix |
|----------|---------------|
| a + b    | Op_Add        |
| a - b    | Op_Subtract   |
| a * b    | Op_Multiply   |
| a / b    | Op_Divide     |
| a % b    | Op_Modulo     |
| a & b    | Op_BitAnd     |
| a &#124; b | Op_BitOr      |
| a << b   | Op_BitShiftLeft |
| a >> b   | Op_BitShiftRight |
| a &^ b   | Op_BitAndNot  |
| a ^ b    | Op_BitXor     |
| a && b   | Op_And        |
| a &#124;&#124; b   | Op_Or         |
| a < b    | Op_Less       |
| a > b    | Op_Greater    |
| a <= b   | Op_LessOrEqual |
| a >= b   | Op_GreaterOrEqual |
| -a       | Op_Minus      |
| +a       | Op_Add        |
| !a       | Op_Not        |

Binary operator methods should have one argument and unary operator methods should have zero arguments.
To support pre-operator methods (i.e. methods defined on `b` instead of `a`), then you can add the
same method but prefixed with `Op_Pre` instead of just `Op_`. Note that the prefixes for binary
and unary methods is the same and they are differentiated by their arguments. Therefore, to support
both unary minus and binary minus (for example), you could write:

```go
func (t T) Op_MinusBinary(o T) T {
  // ...
}

func (t T) Op_MinusUnary() T {
  // ...
}

```

# Not using go-operators
There is much to say for not using go-operators. I already mentioned that it's
a hack, right? Using go-operators relies on a preprocessing step which, although
fairly robust, is subject to bugs. The go/types project is still in development
and certain constructs are bound not to be correctly parsed (yet). Additionally,
using go-operators makes your project no longer go-gettable since a preprocessing
step needs to be performed before the code is actually usable.

# Example
```go
package main

import (
	"fmt"
)

type vec4 [4]float32

func (v vec4) Op_Multiply(o vec4) vec4 {
	return vec4{v[0] * o[0], v[1] * o[1], v[2] * o[2], v[3] * o[3]}
}

func (v vec4) Op_MultiplyScalar(o float32) vec4 {
	return vec4{v[0] * o, v[1] * o, v[2] * o, v[3] * o}
}

func (v vec4) Op_PreMultiplyScalar(o float32) vec4 {
	return v.Op_MultiplyScalar(o)
}

func (v vec4) Op_Add(o vec4) vec4 {
	return vec4{v[0] + o[0], v[1] + o[1], v[2] + o[2], v[3] + o[3]}
}

func (v vec4) Op_AddScalar(o float32) vec4 {
	return vec4{v[0] + o, v[1] + o, v[2] + o, v[3] + o}
}

func (v vec4) Op_PreAddScalar(o float32) vec4 {
	return v.Op_AddScalar(o)
}

func (v vec4) Op_Subtract(o vec4) vec4 {
	return vec4{v[0] - o[0], v[1] - o[1], v[2] - o[2], v[3] - o[3]}
}

func (v vec4) Op_SubtractScalar(o float32) vec4 {
	return vec4{v[0] - o, v[1] - o, v[2] - o, v[3] - o}
}

func (v vec4) Op_PreSubtractScalar(o float32) vec4 {
	return vec4{o - v[0], o - v[1], o - v[2], o - v[3]}
}

func ExampleOverload() {
	v1 := vec4{1, 2, 3, 4}
	v2 := vec4{5, 6, 7, 8}

	// Generates: ret := v1.Op_PreMultiplyScalar(2).Op_Multiply(v2).Op_Add(v1).Op_SubtractScalar(4)
	ret := 2*v1*v2 + v1 - 4

	fmt.Println(ret)
}
```
