# 稀疏数组

```@meta
DocTestSetup = :(using SparseArrays, LinearAlgebra)
```

Julia 在 `SparseArrays` 标准库模块中提供了对稀疏向量和[稀疏矩阵](https://en.wikipedia.org/wiki/Sparse_matrix)的支持。与稠密数组相比，包含足够多零值的稀疏数组在以特殊的数据结构存储时可以节省大量的空间和运算时间。

## [压缩稀疏列(CSC)稀疏矩阵存储](@id man-csc)

在Julia中，稀疏矩阵是按照[压缩稀疏列(CSC)格式](https://en.wikipedia.org/wiki/Sparse_matrix#Compressed_sparse_column_.28CSC_or_CCS.29)存储的。Julia稀疏矩阵具有[`SparseMatrixCSC{Tv,Ti}`](@ref)类型，其中`Tv`是存储值的类型，`Ti`是存储列指针和行索引的整型类型。`SparseMatrixCSC`的内部表示如下所示：

```julia
struct SparseMatrixCSC{Tv,Ti<:Integer} <: AbstractSparseMatrix{Tv,Ti}
    m::Int                  # Number of rows
    n::Int                  # Number of columns
    colptr::Vector{Ti}      # Column i is in colptr[i]:(colptr[i+1]-1)
    rowval::Vector{Ti}      # Row indices of stored values
    nzval::Vector{Tv}       # Stored values, typically nonzeros
end
```

压缩稀疏列存储格式使得访问稀疏矩阵的列元素非常简单快速，而访问稀疏矩阵的行会非常缓慢。在CSC稀疏矩阵中执行类似插入新元素的操作也会非常慢。这是由于在稀疏矩阵中插入新元素时，在插入点之后的所有元素都要向后移动一位。

All operations on sparse matrices are carefully implemented to exploit the CSC data structure
for performance, and to avoid expensive operations.

If you have data in CSC format from a different application or library, and wish to import it
in Julia, make sure that you use 1-based indexing. The row indices in every column need to be
sorted. If your `SparseMatrixCSC` object contains unsorted row indices, one quick way to sort
them is by doing a double transpose.

In some applications, it is convenient to store explicit zero values in a `SparseMatrixCSC`. These
*are* accepted by functions in `Base` (but there is no guarantee that they will be preserved in
mutating operations). Such explicitly stored zeros are treated as structural nonzeros by many
routines. The [`nnz`](@ref) function returns the number of elements explicitly stored in the
sparse data structure, including structural nonzeros. In order to count the exact number of
numerical nonzeros, use [`count(!iszero, x)`](@ref), which inspects every stored element of a sparse
matrix. [`dropzeros`](@ref), and the in-place [`dropzeros!`](@ref), can be used to
remove stored zeros from the sparse matrix.

```jldoctest
julia> A = sparse([1, 2, 3], [1, 2, 3], [0, 2, 0])
3×3 SparseMatrixCSC{Int64,Int64} with 3 stored entries:
  [1, 1]  =  0
  [2, 2]  =  2
  [3, 3]  =  0

julia> dropzeros(A)
3×3 SparseMatrixCSC{Int64,Int64} with 1 stored entry:
  [2, 2]  =  2
```

## 稀疏向量储存

Sparse vectors are stored in a close analog to compressed sparse column format for sparse
matrices. In Julia, sparse vectors have the type [`SparseVector{Tv,Ti}`](@ref) where `Tv`
is the type of the stored values and `Ti` the integer type for the indices. The internal
representation is as follows:

```julia
struct SparseVector{Tv,Ti<:Integer} <: AbstractSparseVector{Tv,Ti}
    n::Int              # Length of the sparse vector
    nzind::Vector{Ti}   # Indices of stored values
    nzval::Vector{Tv}   # Stored values, typically nonzeros
end
```

As for [`SparseMatrixCSC`](@ref), the `SparseVector` type can also contain explicitly
stored zeros. (See [Sparse Matrix Storage](@ref man-csc).).

## 稀疏向量与矩阵构造函数

The simplest way to create a sparse array is to use a function equivalent to the [`zeros`](@ref)
function that Julia provides for working with dense arrays. To produce a
sparse array instead, you can use the same name with an `sp` prefix:

```jldoctest
julia> spzeros(3)
3-element SparseVector{Float64,Int64} with 0 stored entries
```

The [`sparse`](@ref) function is often a handy way to construct sparse arrays. For
example, to construct a sparse matrix we can input a vector `I` of row indices, a vector
`J` of column indices, and a vector `V` of stored values (this is also known as the
[COO (coordinate) format](https://en.wikipedia.org/wiki/Sparse_matrix#Coordinate_list_.28COO.29)).
`sparse(I,J,V)` then constructs a sparse matrix such that `S[I[k], J[k]] = V[k]`. The
equivalent sparse vector constructor is [`sparsevec`](@ref), which takes the (row) index
vector `I` and the vector `V` with the stored values and constructs a sparse vector `R`
such that `R[I[k]] = V[k]`.

```jldoctest sparse_function
julia> I = [1, 4, 3, 5]; J = [4, 7, 18, 9]; V = [1, 2, -5, 3];

julia> S = sparse(I,J,V)
5×18 SparseMatrixCSC{Int64,Int64} with 4 stored entries:
  [1 ,  4]  =  1
  [4 ,  7]  =  2
  [5 ,  9]  =  3
  [3 , 18]  =  -5

julia> R = sparsevec(I,V)
5-element SparseVector{Int64,Int64} with 4 stored entries:
  [1]  =  1
  [3]  =  -5
  [4]  =  2
  [5]  =  3
```

The inverse of the [`sparse`](@ref) and [`sparsevec`](@ref) functions is
[`findnz`](@ref), which retrieves the inputs used to create the sparse array.
[`findall(!iszero, x)`](@ref) returns the cartesian indices of non-zero entries in `x`
(including stored entries equal to zero).

```jldoctest sparse_function
julia> findnz(S)
([1, 4, 5, 3], [4, 7, 9, 18], [1, 2, 3, -5])

julia> findall(!iszero, S)
4-element Array{CartesianIndex{2},1}:
 CartesianIndex(1, 4)
 CartesianIndex(4, 7)
 CartesianIndex(5, 9)
 CartesianIndex(3, 18)

julia> findnz(R)
([1, 3, 4, 5], [1, -5, 2, 3])

julia> findall(!iszero, R)
4-element Array{Int64,1}:
 1
 3
 4
 5
```

Another way to create a sparse array is to convert a dense array into a sparse array using
the [`sparse`](@ref) function:

```jldoctest
julia> sparse(Matrix(1.0I, 5, 5))
5×5 SparseMatrixCSC{Float64,Int64} with 5 stored entries:
  [1, 1]  =  1.0
  [2, 2]  =  1.0
  [3, 3]  =  1.0
  [4, 4]  =  1.0
  [5, 5]  =  1.0

julia> sparse([1.0, 0.0, 1.0])
3-element SparseVector{Float64,Int64} with 2 stored entries:
  [1]  =  1.0
  [3]  =  1.0
```

You can go in the other direction using the [`Array`](@ref) constructor. The [`issparse`](@ref)
function can be used to query if a matrix is sparse.

```jldoctest
julia> issparse(spzeros(5))
true
```

## 稀疏矩阵的操作

Arithmetic operations on sparse matrices also work as they do on dense matrices. Indexing of,
assignment into, and concatenation of sparse matrices work in the same way as dense matrices.
Indexing operations, especially assignment, are expensive, when carried out one element at a time.
In many cases it may be better to convert the sparse matrix into `(I,J,V)` format using [`findnz`](@ref),
manipulate the values or the structure in the dense vectors `(I,J,V)`, and then reconstruct
the sparse matrix.

## Correspondence of dense and sparse methods

The following table gives a correspondence between built-in methods on sparse matrices and their
corresponding methods on dense matrix types. In general, methods that generate sparse matrices
differ from their dense counterparts in that the resulting matrix follows the same sparsity pattern
as a given sparse matrix `S`, or that the resulting sparse matrix has density `d`, i.e. each matrix
element has a probability `d` of being non-zero.

Details can be found in the [Sparse Vectors and Matrices](@ref stdlib-sparse-arrays)
section of the standard library reference.

| 构造函数                     | 密度                  | 说明                                                                                                                                                           |
|:-------------------------- |:---------------------- |:--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`spzeros(m,n)`](@ref)     | [`zeros(m,n)`](@ref)   | Creates a *m*-by-*n* matrix of zeros. ([`spzeros(m,n)`](@ref) is empty.)                                                                                              |
| [`sparse(I, n, n)`](@ref)  | [`Matrix(I,n,n)`](@ref)| Creates a *n*-by-*n* identity matrix.                                                                                                                                 |
| [`Array(S)`](@ref)         | [`sparse(A)`](@ref)    | Interconverts between dense and sparse formats.                                                                                                                       |
| [`sprand(m,n,d)`](@ref)    | [`rand(m,n)`](@ref)    | Creates a *m*-by-*n* random matrix (of density *d*) with iid non-zero elements distributed uniformly on the half-open interval ``[0, 1)``.                            |
| [`sprandn(m,n,d)`](@ref)   | [`randn(m,n)`](@ref)   | Creates a *m*-by-*n* random matrix (of density *d*) with iid non-zero elements distributed according to the standard normal (Gaussian) distribution.                  |
| [`sprandn(m,n,d,X)`](@ref) | [`randn(m,n,X)`](@ref) | Creates a *m*-by-*n* random matrix (of density *d*) with iid non-zero elements distributed according to the *X* distribution. (Requires the `Distributions` package.) |

# [Sparse Arrays](@id stdlib-sparse-arrays)

```@docs
SparseArrays.SparseVector
SparseArrays.SparseMatrixCSC
SparseArrays.sparse
SparseArrays.sparsevec
SparseArrays.issparse
SparseArrays.nnz
SparseArrays.findnz
SparseArrays.spzeros
SparseArrays.spdiagm
SparseArrays.blockdiag
SparseArrays.sprand
SparseArrays.sprandn
SparseArrays.nonzeros
SparseArrays.rowvals
SparseArrays.nzrange
SparseArrays.dropzeros!
SparseArrays.dropzeros
SparseArrays.permute
permute!{Tv, Ti, Tp <: Integer, Tq <: Integer}(::SparseMatrixCSC{Tv,Ti}, ::SparseMatrixCSC{Tv,Ti}, ::AbstractArray{Tp,1}, ::AbstractArray{Tq,1})
```

```@meta
DocTestSetup = nothing
```
