---
layout: post
title: Julia Interoprating with HEP C++ libraries | GSoC @ CERN-HSF 
author: Yash Solanki
categories: Tech
permalink: /rootio-jl
---

## <u> Table of Contents </u>
1. [Synopsis](#-synopsis-)
2. [Baseline support](#-baseline-support-)
3. [C-style arrays](#-c-style-arrays-)
4. [Future scope](#-future-scope-)
5. [Conclusion, difficulties, and learning outcomes](#-conclusion-difficulties-and-learning-outcomes-)

## <u> Synopsis </u>

CERN's data-analysis software ROOT is widely used for high energy physics applications, and Julia is has gained popularity as a fast scientific computing language with a user friendly user syntex than C++ with a similar performance. Root is available via the [ROOT.jl](https://github.com/JuliaHEP/ROOT.jl) packages that allows Julia users to access the features of ROOT. However, prior to the introduction of [RootIO.jl](https://github.com/JuliaHEP/RootIO.jl), writing to ROOT files had a fairly complex syntax:

```julia
import Pkg
Pkg.add(Pkg.PackageSpec(url="https://github.com/JuliaHEP/ROOT.jl.git"))
using DataFrame, ROOT, RDatasets
function saveasroot(df, fname)
    f = ROOT.TFile!Open(fname, "RECREATE")
    tree = ROOT.TTree("tree", "example root tree created from dataframe")
    num_rows = nrow(df) 
    num_cols = ncol(df)      
    data_ptr_array = []
    branch_array = []
    for (col_name, col_data) in pairs(eachcol(df))
        data_pointer = col_data[1:1][]
        current_branch = Branch(tree, "$col_name", data_pointer[], num_rows, 99)
        push!(data_ptr_array, data_pointer[])
        push!(branch_array, current_branch)
    end
    for i in 1:num_rows
        current_row = df[i, :]
        for j in 1:num_cols
            data_ptr_array[j] = Ref(current_row[j])
            SetAddress(branch_array[j], data_ptr_array[j])
        end
        Fill(tree)
    end
    Write(tree)
    Close(f)
end
df = RDatasets.dataset("datasets", "quakes")
saveasroot(df, "quakes.root")
```
<center><u><b>Fig 1:An example of I/O of a dataframe in ROOT.jl</b></u></center>

In my Google summer of code project, we aim to create the [RootIO.jl](https://github.com/JuliaHEP/RootIO.jl) package which facilitates the I/O on Julia side to ROOT, allowing interoperation of C++ library in Julia.

The specifications for the functions were created by Pere and Phillipe, my mentors. The specification document can be found in the [specification file]({{ site.url }}/assets/specifications.pdf).

# <u> Baseline support </u> 

A baseline write interface was created as a starting point of the project. It relies on the functions provided by [ROOT.jl](https://github.com/JuliaHEP/ROOT.jl) and [CxxWrap](https://github.com/JuliaInterop/CxxWrap.jl). The methods provided in the baseline package allows the user to create trees with given specifications and I/O with given rows. It provides the methods `TTree(dir, name, title, type).`, `Fill(tree, row)` and `Write(dir, name, title, table)` for writing to the ttree. Further we can use a keyword-arguments constructor `TTree(file, name, title, col1_name = col1_type, col2_name = col2_type,...)` similar to the construct of DataFrames for constructing a new TTree.

Some examples of what user can do after the baseline package include:

1.Using structs to construct a TTree. The column data types and columns names are inferred from the fields of the struct:

```julia
import RootIO, ROOT
using CxxWrap
mutable struct Event
    x::Float32
    y::Float32
    z::Float32
    v::StdVector{Float32}
end
f = ROOT.TFile!Open("data.root", "RECREATE")
Event()  = Event(0., 0., 0., StdVector{Float32}())
tree = RootIO.TTree(f, "mytree", "mytreetitle", Event)
e = Event()
for i in 1:10
    e.x, e.y, e.z = rand(3)
    resize!.([e.v], 5)
    e.v .= rand(Float32, 5)
    RootIO.Fill(tree, e)
end
RootIO.Write(tree)
ROOT.Close(f)
```
<center><u><b>Fig 2: Writing to TTrees from struct</b></u></center>

2.Using keyword-arguments for constructing a TTree:

```julia
import RootIO, ROOT
using DataFrames
file = ROOT.TFile!Open("example.root", "RECREATE")
name = "example_tree"
title = "Example TTree"
data = (col_float=rand(Float64, 3), col_int=rand(Int32, 3))
tree = RootIO.TTree(file, name, title; data...)
RootIO.Write(tree)
ROOT.Close(file)
```
<center><u><b>Fig 3: Writing to TTrees using keyword-arguments</b></u></center>

3.Write function for writing a table directly to a TTree. The code from Fig 1. simplifies to:

```julia
import Pkg
Pkg.add(Pkg.PackageSpec(url="https://github.com/JuliaHEP/ROOT.jl.git"))
using DataFrame, ROOT
df = RDatasets.dataset("datasets", "quakes")
Write("mydir", "Quakes", "An Example TTree", df, "quakes.root")
```
<center><u><b>Fig 4: Writing to TTrees from DataFrame</b></u></center>

The follwing types is supported in the basline packages:

| **Type**                 | **Description**                                    |
|--------------------------|---------------------------------------------------|
| String                   | A character string                                |
| Int8                     | An 8-bit signed integer                           |
| UInt8                    | An 8-bit unsigned integer                         |
| Int16                    | A 16-bit signed integer                           |
| UInt16                   | A 16-bit unsigned integer                         |
| Int32                    | A 32-bit signed integer                           |
| UInt32                   | A 32-bit unsigned integer                         |
| Float32                  | A 32-bit floating-point number                    |
| Half32b                  | 32 bits in memory, 16 bits on disk                |
| Float64                  | A 64-bit floating-point number                    |
| Double32c                | 64 bits in memory, 32 bits on disk                |
| Int64                    | A long signed integer, stored as 64-bit           |
| UInt64                   | A long unsigned integer, stored as 64-bit         |
| Bool                     | A boolean                                         |
| StdVector{T}             | A vector of elements of any of the above types    |

<center><u><b>Fig 5: Supported data types</b></u></center>

# <u> C-style arrays </u>

The support to write C-style arrays with constant and variable sizes was also added through a separate Pull Request. The follwing syntax is used for creating the C-style arrays:

1.Creating fixed-size C-style array:

```julia
import RootIO, ROOT
file = ROOT.TFile!Open("example.root", "RECREATE")
name = "example_tree"
title = "Example TTree"
my_arr_fixed_length = 3
tree = RootIO.TTree(file, name, title; my_arr = (Int64, my_arr_fixed_length))
RootIO.Fill(tree, [[1,10,100]])
RootIO.Fill(tree, [[2,20]])
RootIO.Write(tree)
ROOT.Close(file)
```

<center><u><b>Fig 6: Fixed-size C-style arrays</b></u></center>

2.Creating a variable-size C-style array:

```julia
import RootIO, ROOT
file = ROOT.TFile!Open("example.root", "RECREATE")
name = "example_tree"
title = "Example TTree"
tree = RootIO.TTree(file, name, title; arr_size = Int64, my_arr = (Int64, :arr_size))
# the first parameter is the array size
RootIO.Fill(tree, [3, [1,10,100]])
RootIO.Fill(tree, [2, [2,20]])
RootIO.Write(tree)
ROOT.Close(file)
```
<center><u><b>Fig 7: Variable-size C-style arrays</b></u></center>

# <u> Future scope </u>

Currently, I'm working on adding compositions of structs and vector of structs. The C++ functions are not yet wrapped for supporting such compositions because creating sub-branches is not yet supported in the ROOT.JL library. I am learning to wrap the C++ ROOT functions too. After the introduction of these functions, we'll be able to write arbitrary compositions of structs. Follwing this, we aim to implement the I/O for RNTuple and custom TObjects. When the library gets completed, the end-user experience of I/O for Julia will become much more easier allowing people to focus more on analysis and less on I/O.

# <u> Conclusion, difficulties, and learning outcomes </u>

The major challenge was understanding how the code is actually working. Learning about memory management in Julia and using pointer to send data between Julia and C++ took was an important outcome. After working with the ROOT team for two consecutive years, I've learned how to wrap libraries and port the functionality between languages, Julia this year and Python last year, that allows a new community to use the code and the older community to benefit from newer languages.

I would like to thank my mentors **Phillipe Gras** and **Pere Mato**. They were very helpful, patient, and helped me learn more about Julia, ROOT and wrapping C++ libraries.

Signing off,

Yash Solanki