# Abstract Syntax Tree Semantics

WebAssembly code is represented as an abstract syntax tree
that has a basic division between statements and
expressions. Each function body consists of exactly one statement.
All expressions and operations are typed, with no implicit conversions or
overloading rules.

Verification of WebAssembly code requires only a single pass with constant-time
type checking and well-formedness checking.

Why not a stack-, register- or SSA-based bytecode?
* Trees allow a smaller binary encoding: [JSZap][], [Slim Binaries][].
* [Polyfill prototype][] shows simple and efficient translation to asm.js.

  [JSZap]: https://research.microsoft.com/en-us/projects/jszap/
  [Slim Binaries]: https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.108.1711
  [Polyfill prototype]: https://github.com/WebAssembly/polyfill-prototype-1

WebAssembly offers a set of operations that are language-independent but closely
match operations in many programming languages and are efficiently implementable
on all modern computers.

Some operations may *trap* under some conditions, as noted below. In the MVP,
trapping means that execution in the WebAssembly module is terminated and
abnormal termination is reported to the outside environment. In a JS
environment such as a browser, a trap results in throwing a JS exception.
If developer tools are active, attaching a debugger before the
termination would be sensible.

Callstack space is limited by unspecified and dynamically varying constraints.
If program callstack usage exceeds the available callstack space at any time,
a trap occurs.

## Local and Memory types

Individual storage locations in WebAssembly are typed, including global
variables, local variables, and parameters. The main storage of a wasm module, 
called the *Linear memory*, is a contiguous, byte-addressable range of memory
spanning from offset `0` to `memory_size`. The linear memory can be considered 
to be a untyped array of bytes which is stored separately from the global variables.
The linear memory is sandboxed; it does not alias the execution engine's internal
data structures, the execution stack, or other process memory.
All accesses to memory are annotated with a type. The legal types for
global variables and memory accesses are called *Memory types*.

  * `int8`: 8-bit integer
  * `int16`: 16-bit integer
  * `int32`: 32-bit integer
  * `int64`: 64-bit integer
  * `float32`: 32-bit floating point
  * `float64`: 64-bit floating point

The legal types for parameters and local variables, called *Local types*
are a subset of the Memory types:

  * `int32`: 32-bit integer
  * `int64`: 64-bit integer
  * `float32`: 32-bit floating point
  * `float64`: 64-bit floating point

All operations except loads and stores deal with local types. Loads convert
Memory types to Local types according to the following rules:

  * `int32.load_sx[int8]`: sign-extend to int32
  * `int32.load_sx[int16]`: sign-extend to int32
  * `int32.load_zx[int8]`: zero-extend to int32
  * `int32.load_zx[int16]`: zero-extend to int32
  * `int32.load[int32]`: (no conversion)
  * `int64.load_sx[int8]`: sign-extend to int64
  * `int64.load_sx[int16]`: sign-extend to int64
  * `int64.load_sx[int32]`: sign-extend to int64
  * `int64.load_zx[int8]`: zero-extend to int64
  * `int64.load_zx[int16]`: zero-extend to int64
  * `int64.load_zx[int32]`: zero-extend to int64
  * `int64.load[int64]`: (no conversion)
  * `float32.load[float32]`: (no conversion)
  * `float64.load[float64]`: (no conversion)

Note that the local types `int32` and `int64` don't technically have a sign; the
sign bit is interpreted differently by the operations below.

Similar to loads, stores convert Local types to Memory types according to the
following rules:

  * `int32.store[int8]`: wrap int32 to int8
  * `int32.store[int16]`: wrap int32 to int16
  * `int32.store[int32]`: (no conversion)
  * `int64.store[int8]`: wrap int64 to int8
  * `int64.store[int16]`: wrap int64 to int16
  * `int64.store[int32]`: wrap int64 to int32
  * `int64.store[int64]`: (no conversion)
  * `float32.store[float32]`: (no conversion)
  * `float64.store[float64]`: (no conversion)

Wrapping of integers simply discards any upper bits; i.e. wrapping does not
perform saturation, trap on overflow, etc.

## Addressing local variables

Each function has a fixed, pre-declared number of local variables which occupy a single
index space local to the function. Parameters are addressed as local variables. Local
variables do not have addresses and are not aliased in the globals or memory. Local
variables have *Local types* and are initialized to the appropriate zero value for their
type at the beginning of the function, except parameters which are initialized to the values
of the arguments passed to the function.

  * `get_local`: read the current value of a local variable
  * `set_local`: set the current value of a local variable

The details of index space for local variables and their types will be further clarified,
e.g. whether locals with type `int32` and `int64` must be contiguous and separate from 
others, etc.

## Control flow structures

WebAssembly offers basic structured control flow. All control flow structures
are statements.

  * `block`: a fixed-length sequence of statements
  * `if`: if statement
  * `do_while`: do while statement, basically a loop with a conditional branch
    (back to the top of the loop)
  * `forever`: infinite loop statement (like `while (1)`), basically an
    unconditional branch (back to the top of the loop)
  * `continue`: continue to start of nested loop
  * `break`: break to end from nested loop or block
  * `return`: return zero or more values from this function
  * `switch`: switch statement with fallthrough

Break and continue statements can only target blocks or loops in which they are
nested. This guarantees that all resulting control flow graphs are well-structured.

  * Simple and size-efficient binary encoding and compilation.
  * Any control flow—even irreducible—can be transformed into structured control
    flow with the
    [Relooper](https://github.com/kripken/emscripten/raw/master/docs/paper.pdf)
    [algorithm](http://dl.acm.org/citation.cfm?id=2048224&CFID=670868333&CFTOKEN=46181900),
    with guaranteed low code size overhead, and typically minimal throughput
    overhead (except for pathological cases of irreducible control
    flow). Alternative approaches can generate reducible control flow via node
    splitting, which can reduce throughput overhead, at the cost of increasing
    code size (potentially very significantly in pathological cases).
  * The
    [signature-restricted proper tail-call](PostMVP.md#signature-restricted-proper-tail-calls)
    feature would allow efficient compilation of arbitrary irreducible control
    flow.

## Floating Point Subnormal Handling

  * `subnormal_mode`: Like [`block`](AstSemantics.md#control-flow-structures),
     but with a mode attribute specifying special semantics for the handling of
     subnormal values in all lexically contained [floating point operations][]
     except `neg`, `abs`, and `copysign`, and [conversions][], except those in
     contained `subnormal_mode` blocks. The mode may be one of:
    * `standard`: standard semantics are followed
    * `dont_care`: Subnormal values may or may not be interpreted as zero of the
       same sign. Subnormal result values may or may not be replaced by any
       subnormal value of the same sign, or zero of the same sign.

  [floating point operations]: AstSemantics.md#floating-point-operations
  [conversions]: AstSemantics.md#datatype-conversions-truncations-reinterpretations-promotions-and-demotions

## Accessing Linear Memory

Programs address memory by using integers that are interpreted as unsigned byte indexes
starting at `0`.
Accesses to memory at indices larger than the size of memory are considered out-of-bounds,
and a module may optionally define that out-of-bounds includes small indices close to `0` (see [discussion] (https://github.com/WebAssembly/design/issues/204)).
Out-of-bounds access is considered a program error, and the semantics are discussed below.
Each memory access is annotated with a *Memory type* and the presumed alignment of the index.

  * `load_mem`: load a value from memory at a given index with given
    alignment
  * `store_mem`: store a given value to memory at a given index with given
    alignment

To enable more aggressive hoisting of bounds checks, memory accesses may also
include an offset:

  * `load_mem_with_offset`: load a value from memory at a given index plus a
    given immediate offset
  * `store_mem_with_offset`: store a given value to memory at a given index
    plus a given immediate offset

The addition of the offset and index is specified to use infinite precision such
that an out-of-bounds access never wraps around to an in-bounds access.  Bounds
checking before the final offset addition allows the offset addition to easily
be folded into the hardware load instruction *and* for groups of loads with the
same base and different offsets to easily share a single bounds check.

In the MVP, the indices are 32-bit unsigned integers. With
[64-bit integers](PostMVP.md#64-bit-integers) and
[>4GiB memory](FutureFeatures.md#heaps-bigger-than-4gib), these nodes would also
accept 64-bit unsigned integers.

In the MVP, linear memory is not shared between threads. When
[threads](PostMVP.md#threads) are added as a feature, the basic
`load_mem`/`store_mem` nodes will have the most relaxed semantics specified in
the memory model and new memory-access nodes will be added with atomic and
ordering guarantees.

Two important semantic cases are **misaligned** and **out-of-bounds** accesses:

### Alignment

If the incoming index argument to a memory access is misaligned with respect to
the alignment operand of the memory access, the memory access must still work
correctly (as if the alignment operand was `1`). However, on some platforms,
such misaligned accesses may incur a *massive* performance penalty (due to trap
handling). Thus, it is highly recommend that every WebAssembly producer provide
accurate alignment. Note: on platforms without unaligned accesses,
smaller-than-natural alignment may result in slower code generation (due to the
whole access being broken into smaller aligned accesses).

Either tooling or an explicit opt-in "debug mode" in the spec should allow
execution of a module in a mode that threw exceptions on misaligned access.
(This mode would incur some runtime cost for branching on most platforms which
is why it isn't the specified default.)

### Out of bounds

The ideal semantics is for out-of-bounds accesses to trap, but the 
implications are not yet fully clear.

There are several possible variations on this design being discussed and
experimented with. More measurement is required to understand the associated tradeoffs.

  * After an out-of-bounds access, the module can no longer execute code and any
    outstanding JS ArrayBuffers aliasing the linear memory are detached.
    * This would primarily allow hoisting bounds checks above effectful
      operations.
    * This can be viewed as a mild security measure under the assumption that
      while the sandbox is still ensuring safety, the module's internal state
      is incoherent and further execution could lead to Bad Things (e.g., XSS
      attacks).
  * To allow for potentially more-efficient memory sandboxing, the semantics could
    allow for a nondeterministic choice between one of the following when an
    out-of-bounds access occurred.
    * The ideal trap semantics.
    * Loads return an unspecified value.
    * Stores are either ignored or store to an unspecified location in the linear memory.
    * Either tooling or an explicit opt-in "debug mode" in the spec should allow
      execution of a module in a mode that threw exceptions on out-of-bounds
      access.

## Accessing globals

Global variables are storage locations outside the linear memory.
Every global has exactly one Memory type.
Accesses to global variables specify the index as an integer literal.

  * `load_global`: load the value of a given global variable
  * `store_global`: store a given value to a given global variable

The specification will add atomicity annotations in the future. Currently
all global accesses can be considered "non-atomic".

## Calls

Direct calls to a function specify the callee by index into a function table.

  * `call_direct`: call function directly

Each function has a signature in terms of local types, and calls must match the
function signature
exactly. [Imported functions](MVP.md#code-loading-and-imports) also have
signatures and are added to the same function table and are thus also callable
via `call_direct`.

Indirect calls may be made to a value of function-pointer type. A function-
pointer value may be obtained for a given function as specified by its index
in the function table.

  * `call_indirect`: call function indirectly
  * `addressof`: obtain a function pointer value for a given function

Function-pointer values are comparable for equality and the `addressof` operator
is monomorphic. Function-pointer values can be explicitly coerced to and from
integers (which, in particular, is necessary when loading/storing to memory
since memory only provides integer types). For security and safety reasons,
the integer value of a coerced function-pointer value is an abstract index and
does not reveal the actual machine code address of the target function.

In the MVP, function pointer values are local to a single module. The
[dynamic linking](FutureFeatures.md#dynamic-linking) feature is necessary for
two modules to pass function pointers back and forth.

Multiple return value calls will be possible, though possibly not in the
MVP. The details of multiple-return-value calls needs clarification. Calling a
function that returns multiple values will likely have to be a statement that
specifies multiple local variables to which to assign the corresponding return
values.

## Literals

Each Local type allows literal values directly in the AST. See the
[binary encoding section](BinaryEncoding.md#constant-pool).

## Expressions with control flow

Expression trees offer significant size reduction by avoiding the need for
`set_local`/`get_local` pairs in the common case of an expression with only one,
immediate use. The following primitives provide AST nodes that express control
flow and thus allow more opportunities to build bigger expression trees and
further reduce `set_local`/`get_local` usage (which constitute 30-40% of total
bytes in the
[polyfill prototype](https://github.com/WebAssembly/polyfill-prototype-1)).
Additionally, these primitives are useful building blocks for
WebAssembly-generators (including the JavaScript polyfill prototype).

  * `comma`: evaluate and ignore the result of the first operand, evaluate and
    return the second operand
  * `conditional`: basically ternary `?:` operator

New operations may be considered which allow measurably greater
expression-tree-building opportunities.

## 32-bit Integer operations

Most operations available on 32-bit integers are sign-independent. Signed
integers are always represented as two's complement and arithmetic that
overflows conforms to the standard wrap-around semantics. All comparison
operations yield 32-bit integer results with `1` representing `true` and `0`
representing `false`.

  * `int32.add`: signed-less addition
  * `int32.sub`: signed-less subtraction
  * `int32.mul`: signed-less multiplication (lower 32-bits)
  * `int32.sdiv`: signed division
  * `int32.udiv`: unsigned division
  * `int32.srem`: signed remainder
  * `int32.urem`: unsigned remainder
  * `int32.and`: signed-less logical and
  * `int32.ior`: signed-less inclusive or
  * `int32.xor`: signed-less exclusive or
  * `int32.shl`: signed-less shift left
  * `int32.shr`: signed-less logical shift right
  * `int32.sar`: signed-less arithmetic shift right
  * `int32.eq`: signed-less compare equal
  * `int32.slt`: signed less than
  * `int32.sle`: signed less than or equal
  * `int32.ult`: unsigned less than
  * `int32.ule`: unsigned less than or equal
  * `int32.clz`: count leading zeroes (defined for all values, including zero)
  * `int32.ctz`: count trailing zeroes (defined for all values, including zero)
  * `int32.popcnt`: count number of ones

Explicitly signed and unsigned operations trap whenever the result cannot be
represented in the result type. This includes division and remainder by zero,
and signed division overflow (`INT32_MIN / -1`). Signed remainder with a
non-zero denominator always returns the correct value, even when the
corresponding division would trap. Signed-less operations never trap.

Shifts interpret their shift count operand as an unsigned value. When the shift
count is at least the bitwidth of the shift, `shl` and `shr` produce `0`, and
`sar` produces `0` if the value being shifted is non-negative, and `-1`
otherwise.

Note that greater-than and greater-than-or-equal operations are not required,
since `a < b` is equivalent to `b > a` and `a <= b` is equivalent to `b >=
a`. Such equalities also hold for floating point comparisons, even considering
NaN.

## 64-bit integer operations

The same operations are available on 64-bit integers as the those available for
32-bit integers.

## Floating point operations

Floating point arithmetic follows the IEEE-754 standard, except that:
 - The sign bit and significand bit pattern of any NaN value returned from a
   floating point arithmetic operation other than `neg`, `abs`, and `copysign`
   are not specified. In particular, the "NaN propagation"
   section of IEEE-754 is not required. NaNs do propagate through arithmetic
   operations according to IEEE-754 rules, the difference here is that they do
   so without necessarily preserving the specific bit patterns of the original
   NaNs.
 - WebAssembly uses "non-stop" mode, and floating point exceptions are not
   otherwise observable. In particular, neither alternate floating point
   exception handling attributes nor the non-computational operations on status
   flags are supported. There is no observable difference between quiet and
   signalling NaN. However, positive infinity, negative infinity, and NaN are
   still always produced as result values to indicate overflow, invalid, and
   divide-by-zero conditions, as specified by IEEE-754.
 - WebAssembly uses the round-to-nearest ties-to-even rounding attribute, except
   where otherwise specified. Non-default directed rounding attributes are not
   supported.
 - Inside the lexical extent of a `subnormal_mode` block, operators may follow
   [alternate semantics](AstSemantics.md#floating-point-subnormal-handling).

In the future, these limitations may be lifted, enabling
[full IEEE-754 support](FutureFeatures.md#full-ieee-754-conformance).

Note that not all operations required by IEEE-754 are provided directly.
However, WebAssembly includes enough functionality to support reasonable library
implementations of the remaining required operations.

  * `float32.add`: addition
  * `float32.sub`: subtraction
  * `float32.mul`: multiplication
  * `float32.div`: division
  * `float32.abs`: absolute value
  * `float32.neg`: negation
  * `float32.copysign`: copysign
  * `float32.ceil`: ceiling operation
  * `float32.floor`: floor operation
  * `float32.trunc`: round to nearest integer towards zero
  * `float32.nearestint`: round to nearest integer, ties to even
  * `float32.eq`: compare equal
  * `float32.lt`: less than
  * `float32.le`: less than or equal
  * `float32.sqrt`: square root
  * `float32.min`: minimum (binary operator); if either operand is NaN, returns NaN
  * `float32.max`: maximum (binary operator); if either operand is NaN, returns NaN

  * `float64.add`: addition
  * `float64.sub`: subtraction
  * `float64.mul`: multiplication
  * `float64.div`: division
  * `float64.abs`: absolute value
  * `float64.neg`: negation
  * `float64.copysign`: copysign
  * `float64.ceil`: ceiling operation
  * `float64.floor`: floor operation
  * `float64.trunc`: round to nearest integer towards zero
  * `float64.nearestint`: round to nearest integer, ties to even
  * `float64.eq`: compare equal
  * `float64.lt`: less than
  * `float64.le`: less than or equal
  * `float64.sqrt`: square root
  * `float64.min`: minimum (binary operator); if either operand is NaN, returns NaN
  * `float64.max`: maximum (binary operator); if either operand is NaN, returns NaN

`min` and `max` operations treat `-0.0` as being effectively less than `0.0`.

## Datatype conversions, truncations, reinterpretations, promotions, and demotions

  * `int32.wrap[int64]`: wrap a 64-bit integer to a 32-bit integer
  * `int32.trunc_signed[float32]`: truncate a 32-bit float to a signed 32-bit integer
  * `int32.trunc_signed[float64]`: truncate a 64-bit float to a signed 32-bit integer
  * `int32.trunc_unsigned[float32]`: truncate a 32-bit float to an unsigned 32-bit integer
  * `int32.trunc_unsigned[float64]`: truncate a 64-bit float to an unsigned 32-bit integer
  * `int32.reinterpret[float32]`: reinterpret the bits of a 32-bit float as a 32-bit integer
  * `int64.extend_signed[int32]`: extend a signed 32-bit integer to a 64-bit integer
  * `int64.extend_unsigned[int32]`: extend an unsigned 32-bit integer to a 64-bit integer
  * `int64.trunc_signed[float32]`: truncate a 32-bit float to a signed 64-bit integer
  * `int64.trunc_signed[float64]`: truncate a 64-bit float to a signed 64-bit integer
  * `int64.trunc_unsigned[float32]`: truncate a 32-bit float to an unsigned 64-bit integer
  * `int64.trunc_unsigned[float64]`: truncate a 64-bit float to an unsigned 64-bit integer
  * `int64.reinterpret[float64]`: reinterpret the bits of a 64-bit float as a 64-bit integer
  * `float32.demote[float64]`: demote a 64-bit float to a 32-bit float
  * `float32.cvt_signed[int32]`: convert a signed 32-bit integer to a 32-bit float
  * `float32.cvt_signed[int64]`: convert a signed 64-bit integer to a 32-bit float
  * `float32.cvt_unsigned[int32]`: convert an unsigned 32-bit integer to a 32-bit float
  * `float32.cvt_unsigned[int64]`: convert an unsigned 64-bit integer to a 32-bit float
  * `float32.reinterpret[int32]`: reinterpret the bits of a 32-bit integer as a 32-bit float
  * `float64.promote[float32]`: promote a 32-bit float to a 64-bit float
  * `float64.cvt_signed[int32]`: convert a signed 32-bit integer to a 64-bit float
  * `float64.cvt_signed[int64]`: convert a signed 64-bit integer to a 64-bit float
  * `float64.cvt_unsigned[int32]`: convert an unsigned 32-bit integer to a 64-bit float
  * `float64.cvt_unsigned[int64]`: convert an unsigned 64-bit integer to a 64-bit float
  * `float64.reinterpret[int64]`: reinterpret the bits of a 64-bit integer as a 64-bit float

Wrapping and extension of integer values always succeed.
Promotion and demotion of floating point values always succeed.
Demotion of floating point values uses round-to-nearest ties-to-even rounding,
and may overflow to infinity or negative infinity as specified by IEEE-754.
If the operand of promotion or demotion is NaN, the sign bit and significand
of the result are not specified.

Reinterpretations always succeed.

Conversions from integer to floating point always succeed, and use
round-to-nearest ties-to-even rounding.

Truncation from floating point to integer where IEEE-754 would specify an
invalid operation exception (e.g. when the floating point value is NaN or
outside the range which rounds to an integer in range) traps.
