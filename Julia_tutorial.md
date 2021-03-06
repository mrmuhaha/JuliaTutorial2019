Julia tutorial. Install Julia and then follow the instructions below.
A markdown version of this tutorial may be generated by
```
using Pkg
Pkg.activate(".");
Pkg.instantiate()
include("make.jl")
```

# Introduction to Julia for FEM programmers 1

## Quick preview

The Julia command line introduces a programming environment.

```julia
using Pkg
Pkg.develop(PackageSpec(url="https://github.com/PetrKryslUCSD/FinEtoolsHeatDiff.jl"))
Pkg.activate("FinEtoolsHeatDiff")
Pkg.instantiate()
```

The functionality is divided into modules. Here we engage some packages
written for finite element analysis.

```julia
using FinEtools
using FinEtools.MeshExportModule
using FinEtoolsHeatDiff
using BenchmarkTools
```

The basic building block  is a function.

```julia
function Poisson_on_triangle_mesh()
    tstart = time()
    A = 1.0 # dimension of the domain (length of the side of the square)
    thermal_conductivity =  [i==j ? 1.0 : 0.0 for i=1:2, j=1:2];
    Q = -6.0; # internal heat generation rate
    tempf(x, y) =(1.0 + x^2 + 2.0 * y^2);#the exact distribution of temperature
    tempf(x) = tempf.(view(x, :, 1), view(x, :, 2))
    N = 1000;# number of subdivisions along the sides of the square domain
    tolerance = 1.0/N/100.0

    fens, fes = T3block(A, A, N, N)

    geom = NodalField(fens.xyz)
    Temp = NodalField(zeros(size(fens.xyz,1), 1))

    l1 = selectnode(fens; box=[0. 0. 0. A], inflate = tolerance)
    l2 = selectnode(fens; box=[A A 0. A], inflate = tolerance)
    l3 = selectnode(fens; box=[0. A 0. 0.], inflate = tolerance)
    l4 = selectnode(fens; box=[0. A A A], inflate = tolerance)
    List = vcat(l1, l2, l3, l4)
    setebc!(Temp, List, true, 1, tempf(geom.values[List,:])[:])
    applyebc!(Temp)
    numberdofs!(Temp)

    material = MatHeatDiff(thermal_conductivity)
    femm = FEMMHeatDiff(IntegDomain(fes, TriRule(1)), material)

    K = conductivity(femm, geom, Temp)
    F2 = nzebcloadsconductivity(femm, geom, Temp);

    fi = ForceIntensity(FFlt[Q]);
    F1 = distribloads(FEMMBase(IntegDomain(fes, TriRule(1))), geom, Temp, fi, 3);

    U = K\(F1+F2)
    scattersysvec!(Temp, U[:])

    Error = 0.0
    for k in 1:size(fens.xyz,1)
        Error = Error + abs.(Temp.values[k,1] - tempf(fens.xyz[k,:]...))
    end
    Error = Error / size(fens.xyz,1)

    println("The simulation took = $(time() - tstart) seconds")

    if false
        File =  "a.vtk"
        MeshExportModule.vtkexportmesh(File, fens, fes; scalars=[("Temperature", Temp.values)])
        @async run(`"paraview.exe" $File`)
    end

    return Error
end
Poisson_on_triangle_mesh()
```

Depending on the computer (CPU, RAM) the function may run in around 10
seconds. So we can process 2 million triangles, 1 million degrees of freedom,
heat conduction problem in roughly 10 seconds. Clearly the code must run near
FORTRAN or C-language speed.

Now we switch to the environment for the remainder of the tutorial.

```julia
using Pkg
Pkg.activate(".")
Pkg.instantiate()
```

To paraphrase [Chris
Foster](https://discourse.julialang.org/t/elevator-pitch/29457/14?u=petrkryslucsd),
here is the elevator pitch for

- Matlab programmers: “It looks and feels like Matlab for array programming
  and linear algebra. But all your loops can be as fast as C, and it allows you
  to build large programs in a structured way. You can use and install it
  anywhere for free without running a license server!”

- C++ programmers: “Julia functions are as fast as template functions but
  have nice syntax like normal overloaded functions. What’s more, they can do
  dynamic dispatch like virtual functions. In Julia you can interactively
  explore your data and experiment with new algorithms without worrying about
  about recompiling and reloading the data.”

- Python programmers: “Julia code is as succinct and easy to use as Python.
  There’s a wide range of numerical libraries which are easy to install with the
  builtin package system. Because Juila is fast, they are usually written in
  Julia itself so you don’t need to know C to understand and modify the
  libraries you’re using.”

The Julia programming language (https://julialang.org/): What is it?
- a flexible dynamic language,
- multiple dispatch paradigm feels natural to scientific programmers,
- strong support for scientific and numerical computing,
- with performance comparable to traditional statically-typed languages.
- excellent code reuse.

I tend to think of the Julia model as a kind of flexible and
agile compile/link/run cycle. Compared to C++ the build
(compile plus link) is nearly invisible (except for the occasional
time delay). Configuration also tends to be much less onerous.

A simple example shows that the running Julia compiles  different versions of
the function `f` depending on the argument.

```julia
f(x) = (3 * x)
@code_native f(3)
@code_native f(0 + 3im)
@code_native f(3.0im)
```

For details on the compilation model see
[AOT or JIT?](https://juliacomputing.com/blog/2016/02/09/static-julia.html)

**Julia is a modern programming language that benefits from several decades of progress in language design.**

A naive implementation of the operation `Y <- A*X + Y` can be programmed in
Julia to run at essentially the same speed as BLAS!

```julia
myaxpy!(a, x, y) = begin
    @assert length(x) == length(y)
    @inbounds for i in eachindex(x)
        y[i] = a * x[i] + y[i]
    end
    return y
end

N = 10_000_000
x = rand(N)
y = rand(N)
a = 1.9
using BenchmarkTools
using LinearAlgebra
@btime @. $y = $a * $x + $y
@btime $y = BLAS.axpy!($a, $x, $y)
@btime $y = myaxpy!($a, $x, $y)
```

Arbitrary-precision numerics.

```julia
using LinearAlgebra

N = 150
A = rand(N, N) + 10000I
det(A)

using ArbNumerics
det(ArbFloat.(A))
```

## Lots of resources

[The Julia website](https://julialang.org/learning/) has a
number of	excellent resources, including video tutorials:
[julialang.org/learning](https://julialang.org/learning/)

## A taste of the language

Cubic Bezier basis functions.

```julia
_b0(t) = (1 - t) ^ 3
_b1(t) = t * (1 - t) * (1 - t) * 3
_b2(t) = (1 - t) * t * t * 3
_b3(t) = (t ^ 3)
```

Documentation writing is easy.

```julia
"""
    bezier(t, p0::T, p1::T, p2::T, p3::T)  where {T}

Return the result of evaluating the Bezier cubic curve function, `t`
from 0 to 1, starting at p0, finishing at p3, intermediate points `p1`,
`p2`.
"""
bezier(t, p0::T, p1::T, p2::T, p3::T)  where {T} =
      p0 * _b0(t) + p1 * _b1(t) + p2 * _b2(t) + p3 * _b3(t)
```

Try `?bezier`.

Different versions of function arguments will result in a collection of
methods defined for the function.

```julia
p0, p1, p2, p3 = [1.0 0.0], [1.0 1.0], [0.0 1.0], [0.0 0.0]
a = vcat(p0, p1, p2, p3)
bezier(t, a) = bezier(t, a[1,:]', a[2,:]', a[3,:]', a[4,:]')
bezier(0.5, a)
bezier(0.5, p0, p1, p2, p3)
methods(bezier)
```

Plotting is supplied with packages. Here we use the package `Winston` to plot a
Bezier curve.

```julia
using Winston
t = 0.0: 0.01: 1.0
a = vcat(p0, p1, p2, p3)
bx = t -> bezier(t, a)[1]
by = t -> bezier(t, a)[2]
p = Winston.plot(bx.(t), by.(t), "r-")
display(p)
```

Manipulation of Julia code is possible with macros. Macros run at compile
time.

```julia
@code_warntype _b0(Float32(0.0))
```

## A few distinguishing features

- Garbage-collected memory management,
- Compiled into LLVM code (easy interfacing with C, C++, or Fortran libraries),
- Flexible and powerful language, yet with quite obvious syntax,
- Designed for numerical computing: All the needed numerical types, and
arrays are powerful and first-class.

## REPL

Read-Evaluate-Print Loop. It is an interactive environment in which Julia
executes code typed in or supplied in files or modules loaded from packages.
On the surface, quite similar to the way Matlab or Python operate.

## Atom

Atom is a good environment for working with Julia. Alternatives are Sublime
Text, Visual Studio Code, vim, ...

## Basics: Variables

Variable names are bindings to values.

```julia
Γ = 90
δ = sind(Γ)
```

## Numbers

Notice the convention: type names are spelled with a capital letter.

```julia
typeof(10_000)
typeof(0.0)
```

Types may be equipped with a parameter: another type

```julia
typeof(0.0 + 1.0im)
Float16(4.)
```

Usual conventions in numerical computing are followed.

```julia
Inf - Inf

prevfloat(Inf)
```

Specialized floating-point types are not uncommon.

```julia
BigFloat(2.0^66) / 3
```

Complex numbers supported from the get-go.

```julia
a = -13.0 + 0.15im
```

## Matrices (arrays)

Arrays are first-class objects.

```julia
A = [1.0 3.0; -3.0 1.0]
B = [-2.0 0.0; -3.0 1.0]
using LinearAlgebra
C = A * B
det(C)
D = A \ B
```

## Operators

```julia
zero(Float64)
one(0)

0 == zero(Int16)
```

## Conversions

```julia
convert(Int, 5.0)
Int(5.1)
Int(round(5.1))
```

## Functions

An untyped function: the compiler will attempt  to figure out the types  when
the function gets called.

```julia
function f(x, y)
    return x + y
end

f(5, 3.0)
f(2, 3)
```

Unicode characters may be used the name functions too.

```julia
Σ(x, y) = x + y
Σ(1, 2)
```

Julia function arguments follow a convention sometimes called
"pass-by-sharing", which means that values are not copied when they are passed
to functions.

Function arguments act as new variable bindings (new locations that
can refer to values), but the values they refer to are identical to the passed
values.

```julia
g(x) = (x = 3)
x = 0
g(x)
x
```

Defining "functions" with the same name produces *methods* for the function.
Methods are differentiated based on the types of their arguments. We have
mentioned methods above, and we will talk more about  methods later.

## Operators are functions

```julia
3 + 5
+(3, 5)
op = *
op(4, 5, 6)
```

## Tuples

List of values, separated by commas. It may be thought of in terms of an
argument list, or the output from a function.

```julia
(2, 3)
("hello", 42)
```

This function returns a tuple.

```julia
h(x) = "the value is", x
c = h(1)
c[1]
c[2]
c1, c2 = h(1)
c1
```

This one does too:

```julia
function foo(x)
    return x, 2*x, 3*x
end
foo(4)
```

Named tuples may also come in handy.

```julia
x = (a=9, b=1+1)
x.b
x[1]
```

## Optional and keyword arguments

Function with an optional argument.

```julia
f(x, y=2) = y*x
f(3)
```

Function with a keyword argument.

```julia
f(x; y) = x / y
f(1)
f(1; y=0)
```

Function with only keyword arguments.

```julia
f(; x, y) = y, x, y
f(y = 1, x = 2)
```

# Introduction to Julia for FEM programmers 2

## Multi-dimensional Arrays

Arrays in Julia are fully supported by the language, plus there are additional
packages for working with arrays in novel ways: small arrays allocated on the
stack, strided arrays, distributed arrays.

### Concatenation

```julia
[1, 2, 3]

[1 3 5 7]

[1 2; 3 4]

vcat(3, 4, [5, 6])
```

### Typed array initializers

```julia
Int8[[1 2] [3 4]]
```

### Comprehensions and generators

```julia
x = [j^2 for j in 1:5]
```

The following generator expression sums a series without allocating memory:

```julia
sum(j^2 for j in 1:5)
```

### Indexing

```julia
x = [2^j+i+j for i in 1:3, j in 1:4]
@show x
```

Arrays are stored column by column.

```julia
x[2]
```

Indexing is also implemented as functions!

```julia
getindex(x, 3)
getindex(x, 3, 2)

x[3] = -1
@show x
setindex!(x, -9, 1, 3)
@show x
```

Multidimensional arrays are supported:

```julia
A = reshape(collect(1:16), (2, 2, 2, 2))
A[2, 1, 1, 2]
```

Linear indexing:

```julia
A[:]

x = reshape(1:16, 4, 4)
x[2:end, 1:2]

x = [1 2 3; 4 5 6; 7 8 9]
x[[1, 3], [1]] .= 0
@show x
```

### Iteration

Linear indexing

```julia
A = rand(2, 3)
for a in A
    print(a, "\n")
end
```

Cartesian indexes

```julia
@show B = view(A, 1:2, 1:2:3);
for i in eachindex(B)
    print(i, " ", B[i], "\n")
end
B[2, 2] == B[CartesianIndex(2, 2)]
```

### Array and Vectorized Operators and Functions

```julia
B = reshape(1:4, (2, 2))
B.^2
sin.(B)
sin.(B).^2 + cos.(B).^2
```

### Broadcasting

```julia
string.(1:3, ". ", ["First", "Second", "Third"])

a = [1.0, 3.0]
b = [2.0 4.0]
a .* b
```

# Introduction to Julia for FEM programmers 3

## Types

Julia's type system is dynamic, but allows to indicate that certain values are
of specific types (static typing).

`x` can be of any type in this definition of a function

```julia
f(x) = 5*x
```

`x` can only be an integer number with 64-bit...

```julia
F(x::Int64) = 5*x
```

...and hence this will fail.

```julia
F(3.3)
```

### General observations about types in Julia

- Only values have types, variables are just names.

```julia
a = 3
typeof(a)
a = 310.0
typeof(a)
```

This doesn't mean that `a` changed type, it just points to a value of a
different type.

- All values have types, and all types are on equal footing (first class).

```julia
typeof(5), typeof(5//3)
```

- Abstract types are declared as such and no values of these types can exist
(an abstract type cannot define "fields", and hence cannot hold any data).

```julia
abstract type T1 end
T1()
```

- However, functions can refer to arguments of abstract types.

```julia
function f1(a::T, b) where {T<:T1}
    return a.i * b
end
In this case, the assumption is that a subtype of `T1` will have a field `i`.
```

- Type that is not abstract is concrete.

```julia
isconcretetype(typeof(1_000))
isconcretetype(Number)
```

- Abstract types can be subtyped. This type is concrete, and hence the value
of this type can exist. Note the field `i`.

```julia
struct CT1 <: T1; i; end
a = CT1(4)
```

We can pass this value to the function `f1` defined above.

```julia
f1(a, 2)
```

This is how we find out about the type tree.

```julia
supertype(typeof(a))
```

- No type can have a concrete type for its supertype. In other words, concrete
types cannot be subtyped.

```julia
struct CT2 <: CT1 end
```

- Types can be parameterized. The type parameter here is `T`.

```julia
struct CT1P{T} <: T1 where {T}
    i::T # field
end
b = CT1P{Float64}(1.0)
supertype(typeof(b))
typeof(b.i)
c = CT1P{Int64}(13)
typeof(c.i)
```

- All types are subtypes of `Any`.
Here the argument `z` can be anything, since its type is not specified and
therefore it defaults to `Any`.

```julia
f(z) = zero(z)
```

### "Is an instance of" operator, type assertion

```julia
(1+2)::Int

c::CT1P
c::T1
```

### Listing type hierarchy

```julia
function listsupertypes(t)
	tlist = [t]
	while t != (t = supertype(t))
		push!(tlist, t)
	end
	tlist = reverse(tlist)
	print(popfirst!(tlist), "\n")
	while !isempty(tlist)
		print("is a supertype of ", popfirst!(tlist), "\n")
	end
end

listsupertypes(Float16)
```

Credit of Carsten Bauer

```julia
function show_subtypetree(T, level=1, indent=4)
   level == 1 && println(T)
   for s in subtypes(T)
     println(join(fill(" ", level * indent)) * string(s))
     show_subtypetree(s, level+1, indent)
   end
end
show_subtypetree(Number)
```

There is also a "subtype" operator.

```julia
typeof(1)<:Int
typeof(1)<:Number
typeof(1)<:Any
Float16 <: Real
```

# Introduction to Julia for FEM programmers 4

## Types: continued

### Primitive types

The value representation consists entirely of a block of bits

```julia
primitive type MyFloat256 <: AbstractFloat 256 end
```

### Composite types

Example of a type that represents a set of nodes, each represented by a location
stored in the array `xyz` which is a field of the "object".

```julia
struct FENodeSet
	xyz::Array{Float32, 2}
end
```

The type name is reflected in the constructor:

```julia
fens = FENodeSet([ 0.785864  0.211725
					0.987322  0.434764
					0.501885  0.388429])
```

Access using the dot notation

```julia
fens.xyz
```

The above type definition is for an immutable type: the fields cannot be
changed.

```julia
fens.xyz = rand(3, 2)
```

In this way we can make the fields of the type mutable.

```julia
mutable struct MutableFENodeSet
	xyz::Array{Float32, 2}
end

fens = MutableFENodeSet([ 0.785864  0.211725
					 0.987322  0.434764
					 0.501885  0.388429])
fens.xyz = rand(3, 2)
```

An object with an immutable type may be copied freely by the compiler since it
is impossible to distinguish between the original object and a copy. Small
immutable object are typically passed on the stack; immutable object rather
live on the heap.

# Introduction to Julia for FEM programmers 5

## Types: continued

### Parametric types (generic programming)

Example of a type that represents a set of nodes, each represented by location
stored in the array `xyz` which is a field of the "object". Different variants
of this type can be generated by specifying the type of the floating-point
numbers of the coordinates.

```julia
struct ParFENodeSet{T}
	xyz::Array{T, 2}
end
```

The type name is reflected in the constructor:

```julia
fens1 = ParFENodeSet([0.785864  0.211725
					0.987322  0.434764
					0.501885  0.388429])

fens2 = ParFENodeSet(Float16[0.785864  0.211725
					0.987322  0.434764
					0.501885  0.388429])

function listsupertypes(t)
	tlist = [t]
	while t != (t = supertype(t))
		push!(tlist, t)
	end
	tlist = reverse(tlist)
	print(popfirst!(tlist), "\n")
	while !isempty(tlist)
		print("is a supertype of ", popfirst!(tlist), "\n")
	end
end

listsupertypes(ParFENodeSet{Float16})

fens3 = ParFENodeSet(Number[0.785864  0.211725
					0.987322  0.434764
					0.501885  0.388429])
listsupertypes(ParFENodeSet{Number})
```

Composite types with subtype parameters are not subtypes.

```julia
Float16 <: Number
ParFENodeSet{Float16} <: ParFENodeSet{Number}
```

This is known as *type invariance*.

Abstract types as type parameters cause sometimes inefficiencies.
`ParFENodeSet{Number}` must be able to store any number, integers of various
numbers of bits, floating-point numbers... Compiler cannot allocate space.

To write a function to operate on any variant of `FENodeSet{T}`, we should
write

```julia
function smallestx(fens::ParFENodeSet{<:Number})
	minimum(fens.xyz[:, 1])
end

smallestx(fens1)
smallestx(fens2)
smallestx(fens3)
```

When the function `smallestx` is called with different types, it gets compiled
for each new type anew (an appropriate method of `minimum` will be called).

# Introduction to Julia for FEM programmers 6

## Methods

When a function is defined, its arguments may be to some degree constrained by
types, but when its called, the types of the arguments may be known more
precisely. For instance, the addition operator:

```julia
methods(+)
```

The word "addition" has a semantic content meaning add together two things. The
implementation of adding together two things is different for different types
of things.

The concept of methods being associated with functions rather than objects
seems quite natural in scientific computing (such as the numerical methods of
the finite difference or finite element type).

*Example:*
Consider the  check whether or not a vector of floating-point numbers is
real, in the sense that no entry of that factor has a nonzero imaginary
component.

For a vector of complex numbers, we check all entries (or more precisely check
with short-circuiting to stop the search as soon we find one component with
nonzero imaginary part):

```julia
isrealvector(v::Array{Complex{T}, 1}) where {T} = !any((-).(v, conj(v)) .!= zero(Complex{T}))
```

The above is fine for vectors consisting of complex numbers. For vectors that
consists only of real numbers, we know that they are real and no actual
computation is necessary.

```julia
isrealvector(v::Array{T, 1}) where {T<:Real} = true
v = rand(4)
isrealvector(v)
v = rand(4) + 0im .* rand(4)
isrealvector(v)
v = rand(4) + 1im .* rand(4)
isrealvector(v)
```

So in this way we have a function with two methods, taking advantage of
short-circuiting as much as possible for performance.

### Multiple dispatch

```julia
a2(x::AbstractFloat, y::Int) = 2x + 2y
a2(5.13, 7)
a2(7, 5.13)
```

Now we add another method which works of arguments of type `Any`: this covers
all possible inputs. Of course, only inputs for which the operation `2x + 2y`
is defined make sense.

```julia
a2(x, y) = 2x + 2y
a2(7, 5.13)

methods(a2)
```

What if we wanted to construct a string expressing the operation? Define
another method.

```julia
a2(x::String, y::String) = "2*$x + 2*$y"
a2("7", "5.13")
```

Multiple dispatch on the types of values is a **central feature** of the Julia
language.

### Parameterized methods

```julia
aa2(x::T1, y::T2) where {T1, T2} = 2x + 2y
methods(aa2)
aa2(5.13, 7)
aa2(7, 5.13)
```

# Introduction to Julia for FEM programmers 7

## Methods: continued

### Callable object (functors)

```julia
struct NormalEvaluator{F}
    evaluationfunction::F
end

function (n::NormalEvaluator)(x)
    return n.evaluationfunction(x)
end


function spherenormal(x)
    xn = sum(x.^2)
    return x / sqrt(xn)
end

e = NormalEvaluator(spherenormal)

e([1.0, 1.0, -1.0])
e([0.0, 1.0, -1.0])
```

# Introduction to Julia for FEM programmers 8

## Interfaces

### Iteration

```julia
struct ElemConnSet
    connectivity::Array{Int64, 2}
end

Base.iterate(S::ElemConnSet, state=1) = state > size(S.connectivity, 1) ?
    nothing :
    (S.connectivity[state, :], state+1)

ecs = ElemConnSet(rand(UInt16, 5, 2) .+ 1)

for c in ecs
    @show c
end
```

### Abstract arrays

Define methods `size`, `getindex`, `setindex!`. Plus perhaps some optional
methods.

# Introduction to Julia for FEM programmers 9

## Modules

Modules are a good organizational principle: related concepts, and also
potentially data, are grouped together and protected from interference by
external code. Modules are imported with the `using` and `import` keywords.

### Example

```julia
module FizzingWhizzbees

function makeone()
    "Here is your Fizzing Whizzbee"
end

end

using .FizzingWhizzbees
FizzingWhizzbees.makeone()

using .FizzingWhizzbees: makeone

makeone()
```

### Standard modules

- Basic facilities of the language: `Base`, `Core`.
- Top-level, default when running: `Main`.

### Standard library

- Package manager,
- Unit testing,
- Linear algebra,
- Random numbers,
- Statistics,
- Sparse arrays,
- Distributed computing,
- and many more.

To use the standard library, the standard library package needs to be
installed, using the package manager.

```julia
using Pkg
Pkg.add("LinearAlgebra")
```

To use standard library modules, the code needs to be brought into scope by `using` or `import`.

A few examples are shown below:

#### Test-driven Development (TDD) is supported by Unit testing.

```julia
using Test

myapproximateequal(x, y, tol) = abs(x - y) <= tol

x = 1.13
y = 1.135
tol = 0.01
@test myapproximateequal(x, y, tol) == true
@test myapproximateequal(x, y, tol / 10) !== true
```

#### linear algebra

```julia
using LinearAlgebra

x = rand(3)
x / norm(x)


using LinearAlgebra: norm

x = rand(3)
x / norm(x)
dot(x, x)
```

# Introduction to Julia for FEM programmers 10

## Meta-programming

Julia can treat code is data: transform and generate code. Julia code is a
data structure which can be accessed from the language itself.

Examples of macros:

```julia
@show (1 + 1)

x = 0
@assert x == 1

a = fill(0.0, 100)
@inbounds for i in 1:length(a)
        a[i]
end

@time let
        s = 0;
        for i in 1:1_000_000
                s = s + cos(s)*rem(s, 7)
        end
end

macroexpand(Main, :(@show 2^3))
macroexpand(Main, quote
@time let
        s = 0;
        for i in 1:1_000_000
                s = s + cos(s)*rem(s, 7)
        end
end
end)
```

Macros execute when code is parsed, and therefore, macros allow the programmer
to generate and include fragments of customized code before the full program
is run.

# Introduction to Julia for FEM programmers 11

## Interoperability with the C language

In file mycos.c:
#include "math.h"

// gcc -shared -fPIC -o mycos.so mycos.c -lm

double
mycos(double x) { return cos(x); }

```julia
using Libdl

mycoslib = dlopen("./mycos.so")

x = pi / 4
c = ccall(dlsym(mycoslib, :mycos), Cdouble, (Cdouble,), x)
@show c, cos(x)
```

Bits arrays (arrays which consist of entries of the "bits" type, such as
floating-point or integer numbers) are packed in memory so that they can be
*passed directly* to a C function.

# Introduction to Julia for FEM programmers 12

## Linear algebra

BLAS and LAPACK are supported. In addition, there are pure-Julia
implementations of linear algebra which are a vision of what the [future
linear-algebra library may look
like](https://github.com/JuliaLinearAlgebra/GenericLinearAlgebra.jl).

### Avoid data copying and temporaries

```julia
M, N, P = 100, 2000, 1500
A = rand(ComplexF64, M, N)
B = rand(ComplexF64, N, P)
C = fill(0.0 + 0.0im, M, P)
mul!(C, A, B)
using BenchmarkTools
@btime mul!($C, $A, $B)
@btime $C =  $A * $B
```

### Views

When constructing sub matrices it is possible to avoid copying of the matrix
entries by using views.

```julia
A = rand(1000, 2000)
using BenchmarkTools
@btime Asub = A[2:2:end, 1:2:end]
@btime Asub = view(A, 2:2:lastindex(A, 1), 1:2:lastindex(A, 2))
```

### Broadcast and fusion

All functions can be applied to collections (arrays) with the "dot" notation.
The compiler can fuse nested operations so that no temporaries are created.

```julia
A = [1.0, 2.0, 3.0]
A.^2
B = [1.0, 2.0, 3.0]
C = similar(B)
@. C = sin(A^2) + cos(3*B)
@btime @. $C = sin($A^2) + cos(3*$B)
```

With the "dot" notation, views are also employed to carry out assignment in
place.

```julia
Y = rand(50)
X = fill(0.0, length(Y) + 1)
X[2:end] .= sin.(Y)
```

# Introduction to Julia for FEM programmers 13

## Sparse matrix algebra (SparseArrays)

Sparse matrices in Julia are represented in the compressed-Column format
(CSC).

```julia
using SparseArrays

A = sparse([1, 2, 3, 3, 4, 3, 5], [1, 2, 2, 4, 4, 3, 5], Float32[3, 4, 2, 4, -3, 5, 1], 5, 5)

using UnicodePlots
Pl = UnicodePlots.spy(A)
display(Pl)
```

Solvers include direct factorizations, with high-performance solvers such as
UMFPACK. Other solvers are available with additional packages (PETSC, Pardiso)

```julia
using LinearAlgebra

@show A \ rand(5)

LU = lu(A)


Pl = UnicodePlots.spy(LU.L)
display(Pl)
Pl = UnicodePlots.spy(LU.U)
display(Pl)
```

There are iterative solvers available, for instance
[Krylov methods](https://github.com/JuliaInv/KrylovMethods.jl).

Eigenvalue solvers include `eigs`from [Arpack](https://github.com/JuliaLinearAlgebra/Arpack.jl).

# Introduction to Julia for FEM programmers 14

## Plotting

Plotting is not built in into the language, it is provided by external
packages. Statistical plotting, for instance, has Gadfly, Plots, VegaLite; 3D
interactive graphics with OpenGL: Makie. Regular 2D plotting PGFPlotsx.

Sample of Gadfly plotting.

```julia
using Gadfly
xvalues = rand(100)
yvalues = rand(100)
p = Gadfly.plot(x=xvalues, y=yvalues, Geom.point, Geom.line)
display(p)

p = Gadfly.plot(x=1:10, y=2.0.^rand(10), Scale.y_sqrt, Geom.point, Geom.smooth, Guide.xlabel("Stimulus"), Guide.ylabel("Response"), Guide.title("Dog training"))
display(p)
```

Sample of PGFPlotsX  publication-quality plotting for journals.

```julia
using PGFPlotsX
@pgf p = Axis({
        xlabel = "Cost",
        ylabel = "Error",
    },
    PGFPlotsX.Plot({
            color = "red",
            mark  = "x"
        },
        Coordinates(
            [
                (2, -2.8559703),
                (3, -3.5301677),
                (4, -4.3050655),
                (5, -5.1413136),
                (6, -6.0322865),
                (7, -6.9675052),
                (8, -7.9377747),
            ]
        ),
    ),
)
display(p)
```

# Introduction to Julia for FEM programmers 15

## Input, output

Interfaces to databases are available (SQL, for instance). Data frames are
quite developed and integrated into [statistical-exploration
packages](https://www.youtube.com/watch?v=OFPNph-WxLM). Scientific data
formats include HDF5, VTK graphics files and more.

# Introduction to Julia for FEM programmers 16

## Parallel computing

A native master-worker system based on remote procedure calls, MPI, [threads](https://julialang.org/blog/2019/07/multithreading).

Accelerators (graphics-card processing): ArrayFire.jl, GPUArrays.jl, ...

Example: clamping an array. Serial versus threaded version.

```julia
clamp(a) = begin
	for i in 1:length(a)
		a[i] > 0.5 && (a[i] = 0.5)
	end
end
N = 100_000_000
a = rand(N);
using BenchmarkTools
@btime clamp($a)

tclamp(a) = begin
	Threads.@threads for i in 1:length(a)
		a[i] > 0.5 && (a[i] = 0.5)
	end
end
@btime tclamp($a)
```

Example: compute free-vibration solutions for a number of
substructures in parallel.

```julia
using SparseArrays
using LinearAlgebra
M = 4
N = 2_000
symmtx(n) = begin
	m = sprand(Float64, N, N, 0.001)
	m + m' + 10.0I
end
Ks = [symmtx(N) for i in 1:M]
Ms = [symmtx(N) for i in 1:M]

using Distributed
using Arpack: eigs
rmprocs(workers())
addprocs(2)
@everywhere begin
	using Pkg
	Pkg.activate(".")
	using Distributed
	using Arpack: eigs
end
using BenchmarkTools
@btime for i in 1:length($Ks)
	sol = eigs($Ks[i], $Ms[i]; nev=6, which =  :SM)
end

@btime sol = pmap(i -> eigs($Ks[i], $Ms[i]; nev=6, which =  :SM), 1:length($Ks));
```

# Introduction to Julia for FEM programmers 17

## Optimization of Julia code

```
Julia source
|
+- Parsed
   |
   +- Macro expansion (`@macroexpand`)
      |
      +- Lowering  (`@code_lowered`)
         |
         +- Type inference
            |
            +- Inlining
               |
               +- Generated function expansion  (`@code_typed`)
                  |
                  +- Code generation   (`@code_llvm`)
                     |
                     +- Native code  (`@code_native`)
                        |
                        +- Run
```

*This page was generated using [Literate.jl](https://github.com/fredrikekre/Literate.jl).*

