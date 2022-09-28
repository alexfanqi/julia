# riscv64
Currently, this fork is compiling okay and seems to work fine. Most of the test suite pass with some minor failures,
e.g. the NaN vs -NaN issue on riscv, missing Artifacts, and inaccurate results with range.jl. 

riscv needs the new llvm JITLINK, as the legacy RTDyld is in maintanence mode. `_PIC` and medium codemodel is necessary to avoid issue of resolving external symbols (ref: <https://discourse.julialang.org/t/relocation-target-is-out-of-range-using-jitlink/79568>)

I am currently working alone on riscv port. I will submit this in a pr shortly. If anyone else is intereted, I would be very pleased to work together to make the riscv support mature like arm.
## How to build
My build script is provided in [doc/src/devdocs/build/build-julia-riscv.sh](https://github.com/alexfanqi/julia/tree/master/doc/src/devdocs/build/build-julia-riscv.sh)  
I build on Gentoo with dependencies pulled from offical julia.ebuild. It should be no problem to use bundled libs or build on other distros. Building with either clang or gcc will work. Although any setup is okay, below is always required at minimum:
1. llvm 14 with these patches: [R_RISCV_SUB6](https://reviews.llvm.org/D120001), [Generate correct ELF EFlags](https://reviews.llvm.org/D121183), [Fix state persistence bugs](https://reviews.llvm.org/D125905), for the last two to be applied, you'd potentially need [D119244](https://reviews.llvm.org/D119244), [D119250](https://reviews.llvm.org/D119250) and [D121149](https://reviews.llvm.org/D121149). llvm 15 doesn't need any of these patches, but I had some issues compiling Julia with it. 
2. after stdlib is downloaded, under `usr/share/julia/stdlib/v1.9/SparseArrays/src/solvers`, copy `lib/aarch64-linux-gnu.jl` as `lib/riscv64-linux-gnu.jl` and modify `LibSuiteSparse.jl` accordingly.
3. if you use system libopenblas, you might need to create an alias of your system libopenblas64.so by `ln -s /usr/lib64/libopenblas64.so /usr/lib64/libopenblas64_.so`. This is needed because of `stdlib/OpenBLAS_jll/src/OpenBLAS_jll.jl:20  
4. libopenblas also has issue with BLASINT64 support (INTERFACE=64). ref: https://github.com/xianyi/OpenBLAS/issues/3859
The libopenblas64.so compiled has a mixture of BLASINT64 for c functions and BLASINT32 for fortran functions,
which will cause most functions in LinearAlgebra.LAPACK to fail
A quick fix is append `-fdefault-integer-8` to FFLAG.
5. recent enough openlibm with [riscv support](https://github.com/JuliaMath/openlibm/commit/428e7af2148894e287801a0539b7a1756f451e88)
6. recent enough libopenblas with riscv support and compiled for 64 bits (`INTERFACE64=1 LIBPREFIX=libopenblas64`) if using system libopenblas
7. <strike>libblastrampoline needs [riscv64 asm stub](https://github.com/JuliaLinearAlgebra/libblastrampoline/commit/205b3e81cd730918528dfb2f700fec7ea5b9f0d2)</strike> already included in julia main branch

## What's missing
1. <strike>abi_riscv: I have little idea how to implement this correctly with current abi abstraction code. For example, riscv's procedure call convention requires struct containing two floats or one float and one integer should be passed by register.</strike> implmented, references how rust does it. passes all ccall test.
2. Float16 has issue, Float32/64 seem fine. <- should be fixed now
```
julia> Float16(4)
Float16(0.0)

julia> Float16(4.0)
Float16(4.0)

julia> Base.sitofp(Float16, 4)
Float16(4.0)
```
3. cpu detection and additional extensions switch (currently assume +gc)
4. RVV, zfh, crypto

