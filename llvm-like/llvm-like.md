# The Next Generation of Eeyore/Tigger IR

Eeyore v2 and Tigger v2.

## Eeyore v2

### Symbol Name

* **Named symbols**: `@name`, `name` can be any string that can be matched by `[_A-Za-z][_A-Za-z0-9]*`.
* **Temporary symbols**: `%name`, `name` can be any string that can be matched by `[0-9]|[1-9][0-9]+|[_A-Za-z][_A-Za-z0-9]*`.

### Data Types

```ebnf
Shape ::= "[" INT "]" {Shape};
```

* **32-bit signed integer**: default type.
* **Aggregate**:
  * `[2]`: 1-dimensional array of 32-bit signed integers which length is 2.
  * `[2][3]`: 2-dimensional array of 32-bit signed integers which shape is `[2][3]`.

Eeyore v2 is untyped, programmers should ensure that Eeyore v2 program do not have type issues, otherwise undefined behaviors will occur.

### Values

```ebnf
Value ::= SYMBOL | Literal;
Literal ::= INT | Aggregate | "zeroinit"
Aggregate ::= Shape "{" Literal {"," Literal} "}";
```

* **32-bit signed integer**: just like `1`, `233`, etc.
* **Symbol**: just like `@var`, `%0`, etc.
* **Aggregate**: just like `[3] {1, 2, 3}`, `[2][2] {{1, 2}, {3, 4}}`, etc., can only initialize or be assigned to aggregate types.
* **Zero initializer**: `zeroinit`, can initialize or be assigned to all types.

### Symbol Definition

```ebnf
SymbolDef ::= SYMBOL "=" (MemoryDeclaration | GlobalMemoryDeclaration | Load | TODO);
```

All symbols should only be defined once.

### Memory Declaration

```ebnf
MemoryDeclaration ::= "alloc" [Shape];
GlobalMemoryDeclaration ::= "global" "alloc" [Shape] "," Literal;
```

e.g.

```eeyore2
@i = alloc                                // int i
@arr = alloc [2][3]                       // int arr[2][3]
@arr2 = global alloc [2][5], zeroinit     // int arr2[2][5] = {}
@arr3 = global alloc [3], [3] {1, 2, 3}   // int arr3[3] = {1, 2, 3}
```

### Memory Access

```ebnf
Load ::= "load" Value;
Store ::= "store" Value, SYMBOL;
```

e.g.

```eeyore2
// x = i
%0 = load @i
store %0, @x
```

### Pointer Calculation

```ebnf
GetPointer ::= "getptr" SYMBOL, Value;
```

e.g.

```eeyore2
// a[2][3] = 5
%0 = getptr @a, 2
%1 = getptr %0, 3
store 5, %1
```

### Binary/Unary Operation

```ebnf
BinaryExpr ::= BINARY_OP Value, Value;
UnaryExpr ::= UNARY_OP Value;
```

Supported operations:

* **Binary**: `ne`, `eq`, `gt`, `lt`, `ge`, `le`, `land`, `lor`, `add`, `sub`, `mul`, `div`, `mod`.
* **Unary**: `neg`, `lnot`.
