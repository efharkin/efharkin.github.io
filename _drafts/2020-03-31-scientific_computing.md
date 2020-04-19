---
title: Scientific computing beyond Python and C++
author: Emerson Harkin
---

## Why Python and C++?

Why are Python and C++ so dominant in scientific computing?

### C++

It's fast and has been around for a long time. Many operating systems,
compilers, and high-performance libraries are written in C or C++.

Compared with other high-performance languages (eg C and Fortran), C++ offers
modern language features to make development easier. These include object
orientation, type generics, smart pointers, and, most recently, something
resembling
[duck-typing](https://en.cppreference.com/w/cpp/language/constraints). However,
many of these features often go unused in scientific code.

#### Drawbacks

A strong type system and visibility rules make large codebases easier to
maintain, but also make prototyping cumbersome.

The biggest drawback of C++ is its lack of memory safety. This is at the root of
many scary things, including

- lack of array bounds checking
- data races
- `delete` and `delete[]` are not interchangeable
- use after free
- null pointers

all of which can lead to undefined behaviour, leaving your program in an
invalid state.

### Python

Pseudocode-like syntax, duck-typing, and interpreter make prototyping and
debugging very quick and easy. This has enabled an amazing ecosystem of
libraries and tools to spring up around Python.

#### Drawbacks

Undefined behaviour, data races, and memory leaks are alien concepts because
Python does not offer enough low-level control to create these sorts of
problems (with exceptions, see [reference
cycles](https://stackoverflow.com/questions/9910774/what-is-a-reference-cycle-in-python)).
Unfortunately, this means execution is very slow.

Duck typing makes large codebases hard to maintain and runtime errors hard to
avoid. [Type hinting](https://docs.python.org/3/library/typing.html) goes some
way to solving this problem.

## How to eat your cake and have it too

Ideally we want fast prototyping *and* fast runtimes.

### Julia

Julia promises this by combining simple syntax, an interpreter, and a
just-in-time (JIT) compiler.

Other cool features:
- [Array broadcasting](https://docs.julialang.org/en/v1/manual/functions/#man-vectorized-1) is a basic language feature
- Optional static typing
- Calling Python libraries is [trivial](https://github.com/JuliaPy/PyCall.jl)

Syntax is a mix of Python, C, and Matlab (one-based indexing, `^` for
exponentiation).
```julia
function mandelbrot(a)
    z = 0
    for i=1:50
        z = z^2 + a
    end
    return z
end

for y=1.0:-0.05:-1.0
    for x=-2.0:0.0315:0.5
        abs(mandelbrot(complex(x, y))) < 2 ? print("*") : print(" ")
    end
    println()
end
```

#### Drawbacks

- Young and not widely adopted
    - Help is not always easy to find
- Doesn't know what it wants to be
    - Interpreted and compiled
    - Strongly typed but everything is generic by default
    - Object oriented (sort of) without classes

### Numba: JIT compilation for plain Python

[Numba](http://numba.pydata.org) promises to at least try to accelerate plain
Python code simply by wrapping functions with `nb.jit`. Integrates seamlessly
with numpy.
```python
def elementwise_product(a, b):
    output = np.empty_like(a)
    for i in range(len(a)):
        output[i] = a[i] * b[i]
    return output

fast_elementwise_product = nb.jit(elementwise_product)

%timeit elementwise_product(a, b)
>>> 57.1 µs ± 39.2 ns per loop (mean ± std. dev. of 7 runs, 10000 loops each)

%timeit fast_elementwise_product(a, b)
>>> 773 ns ± 1.41 ns per loop (mean ± std. dev. of 7 runs, 1000000 loops each)

%timeit np.elementwise_product(a, b)
>>> 555 ns ± 1.72 ns per loop (mean ± std. dev. of 7 runs, 1000000 loops each)
```

But not guaranteed to do very much.
```python
def slow_elementwise_product(a, b):
    output = []
   for ai, bi in zip(a, b):
        output.append(ai * bi)
    return output

fast_slow_elementwise_product = nb.jit(slow_elementwise_product)

%timeit slow_elementwise_product(a, b)
>>> 42.6 µs ± 161 ns per loop (mean ± std. dev. of 7 runs, 10000 loops each)

%timeit fast_slow_elementwise_product(a, b)
>>> 7.34 µs ± 25.4 ns per loop (mean ± std. dev. of 7 runs, 100000 loops each)
```
There are some [simple
guidelines](http://numba.pydata.org/numba-doc/latest/user/5minguide.html#will-numba-work-for-my-code)
to follow to get the best out of Numba.

#### Drawbacks

- GPU support isn't free
- Speedup not guaranteed, especially when using OO style

--------------------------------------------------------------------------------

## JAX

|              | Hardware support | Autograd | Paradigm    | Sugar             |
| :----------: | :--------------: | :------: | :---------: | :---------------: |
| Numba        | CPU, GPU for $$$ | No       | Imperative* | None              |
| Tensorflow   | CPU + GPU        | Yes      | Declarative | DL ops and utils  |
| JAX          | CPU + GPU        | Yes      | Functional  | None--rough edges |

### Main feature

Autodifferentiation, JIT compilation, and plain Python under the same roof.

### Autodifferentiation

Neither numerical nor symbolic.

JAX views a program as a series of composed operations.
```
Program = op_N(...op_2(op_1)...)
output = Program(input) = op_N(...op_2(op_1(input))...)
```
JAX provides a `grad` function that knows the derivative of most ops. This is
trivial when the ops are mathematical functions, but `grad` also works for
control flow.

To see how, consider this implementation of the ReLU function.
```python
def relu(x):
    if x >= 0:
        output = x
    else:
        output = 0
    return x
```

Depending on the value of `x`, the operation of this function is either `op(x)
= x` or `op(x) = 0`, both of which have trivial derivatives.

```python
from jax import grad

relu_grad = grad(relu)

relu_grad(1.)
>>> DeviceArray(1., dtype=float32)

relu_grad(-1.)
>>> DeviceArray(0., dtype=float32)
```

#### Forward and reverse-mode differentiation

JAX implements two algorithms for tracing the derivatives of arbitrary
functions.

Forward-mode differentiation:
```
partial_output = input
partial_gradient = 1

for op_i in Program
    partial_gradient = partial_gradient * grad(op_i, partial_output)  // Chain rule
    partial_output = op_i(partial_output)
done

final_gradient = partial_gradient
final_output = partial_output
```
In forward-mode differentiation, both the gradients and output are computed in
a single pass through `Program`.

Reverse-mode differentiation:
```
partial_output = input

// Forward pass
for op_i in Program
    partial_output = op_i(partial_output)
done

// Backward pass
partial_gradient = 1
for op_i in op_N..op_1
    partial_output = inv(op_i)(partial_output)
    partial_gradient = partial_gradient * grad(op_i, partial_output)  // Chain rule
done

final_gradient = partial_gradient
final_output = partial_output
```
Here I've sketched out forward- and reverse-mode procedures to take the
gradient of a function with a scalar input and scalar output. In this
simplified case, reverse mode differentiation is more computationally expensive
than forward-mode because it requires traversing the program twice. However, in
situations where the dimensionality of the output is lower than the
dimensionality of the input, reverse mode is usually cheaper.

#### More accurate than finite differences

Example:
```python
def f1(x):
  return 1e-20 * x + 1

def f2(x):
  return 1.00000000000000001e20 * (x - 1)

def foo(x):
  return f2(f1(x))
```

The exact derivative of `foo(5.0)` is `1.000...1`. However, the derivative
evaluated numerically is zero, and the derivative obtained by JAX
autodifferentiation is one.
```python
## Numerical
(foo(5.1) - foo(4.9)) / 0.2
>>> 0.0

## JAX
foo_grad = grad(foo)
foo_grad(5.0)
>>> DeviceArray(1., dtype=float32)
```
### Practical considerations

Combining JIT compilation and autodifferentiation into one utility introduces
tradeoffs.

- Generic representations of Python code reduce need for recompiling
- Specifics are needed to trace gradients
- "Composition of functions" autodiff model requires side-effect-free (ie functional) programs
    - Changing elements of an array after initialization is not allowed!

```python
from jax import numpy as np

def elementwise_product(a, b):
    output = np.empty_like(a)
    for i in range(len(a)):
        output[i] = a[i] * b[i]  # Mutating output[i] is not allowed!
    return output

elementwise_product(a, b)
>>> TypeError: '<class 'jax.interpreters.xla.DeviceArray'>' object does not support item assignment. JAX arrays are immutable; perhaps you want jax.ops.index_update or jax.ops.index_add instead?
```

JAX provides functions (`index_update`, `index_add`) for mutation-like
operations, but these sometimes involve making copies of the underlying array.
Could be expensive if, for example, each element of a large array is being
initialized manually or updated repeatedly. Additionally, repeated updates to
the same location in the array can create race conditions, a form of undefined
behaviour!
([Reference.](https://jax.readthedocs.io/en/latest/_autosummary/jax.ops.index_update.html))

#### Running with scissors

See if you can find the problem here.
```python
from jax import numpy as np

...100 lines of code...

arr = np.ones(10)

...100 more lines of code...

print(arr[10])
>>> DeviceArray(1., dtype=float32)
```

[Answer.](https://jax.readthedocs.io/en/latest/notebooks/Common_Gotchas_in_JAX.html#🔪-Out-of-Bounds-Indexing)

## A note about Rust

Designed by Mozilla to incrementally replace C; meaning that interfacing with
C/C++ should be easy, but not necessarily seamless.

Key features:
- About as fast as C/C++
- Amazing toolchain: `cargo`
    - Easier dependency-management than Python
    - Generate documentation automatically
- Memory-safe
    - Array bounds-checking at runtime
    - Automatic memory management without garbage collection
        - Object lifetimes are found at compile time
- All variables are immutable by default

- Young language but growing number of libraries
    - Numpy-like linear algebra with `ndarray`
    - Object serialization/deserialization with `serde`
        - Including pickled Python objects
    - Write HDF5 files with `hdf5`
    - Forward autodifferentiation with `autodiff`

```rust
use ndarray::Array1;

pub fn elementwise_product(a: Array1<f32>, b: Array1<f32>) -> Array1<f32> {
    let output = &a * &b;
    return output;
}

let num_elements = 10;
let a = Array1::zeros(num_elements);
let b = Array1::zeros(num_elements);

let result = elementwise_product(a, b);
```
### Drawbacks

Steep learning curve because of strong guarantees of program correctness and
unusual approaches to object-orientation and error handling.

#### Guaranteed correctness

Rust programs are guaranteed to be memory safe because of its unique [ownership
system](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html).
In Rust, a variable must be "owned" by a given scope in order to be read or
mutated within that scope. If a variable is owned mutably by any scope (meaning
that changes to the variable are allowed within that scope), no other scopes
are allowed to own the same variable. Variables can be owned immutably by any
number of scopes (meaning that it can be read but not written) as long as it is
not owned mutably by any scope. In multithreaded programs, these rules
guarantee that data races cannot occur. Because ownership is checked at
compile-time, programs that compile without errors are much more likely to be
correct than corresponding C or C++ programs in which any variable can
potentially be read or modified at any time via raw pointers.

Note that Rust provides
[options](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html) to override
the ownership system if necessary.

A nice side-effect of Rust's ownership system is that the lifetime of all
objects in the program can be worked out at compile time. This means that
memory allocation and deallocation can be managed automatically (no need for
`new` and `delete` as in C++) without the overhead of a garbage collector.

#### Rust-style object orientation

In C-style languages, object orientation revolves around classes, which are
structs (containers of data) attached methods that can be used to derive
subclasses. Subclasses inherit the public interface (names and call signatures
of public methods and attributes) of the parent class as well as some parts
of its implementation (protected methods and attributes).

In Rust, object orientation revolves around interfaces called `Trait`s, which
specify the names and call signatures of a collection of methods.

Example:
```rust
trait Conductance {
    fn get_current(&self, voltage: f32) -> f32;
}

struct StaticConductance {
    conductance: f32,
    reversal: f32
}

impl Conductance for StaticConductance {
    fn get_current(&self, voltage: f32) -> f32 {
        let driving_force = voltage - self.reversal;
        return self.conductance * driving_force;
    }
}

struct InactivatingConductance {
    max_conductance: f32,
    reversal: f32,
    activation_gate: f32,
    inactivation_gate: f32
}

impl Conductance for InactivatingConductance {
    fn get_current(&self, voltage: f32) -> f32 {
        let driving_force = voltage - self.reversal;
        let gating_state = self.activation_gate * self.inactivation_gate;
        return self.max_conductance * gating_state * driving_force;
    }
}
```

Here, both `StaticConductance` and `InactivatingConductance` implement the
`Conductance` interface. In C++, the same thing would be accomplished by
creating a `Conductance` class and having `StaticConductance` and
`InactivatingConductance` inherit from it. Rust's approach has the advantage
that new `Trait`s can be implemented on types for which the source code isn't
available, or shouldn't be modified.

#### Error handling in Rust

In Rust, functions can't raise errors or throw exceptions. Instead, functions
that might not exit cleanly return a `Result`, which might be a valid result
(`Ok` type) or an error (`Err` type). Callers are then required to explicitly
handle the possibility of an invalid result. (The laziest way to do this is to
simply crash the program if the function returns `Err`.)

