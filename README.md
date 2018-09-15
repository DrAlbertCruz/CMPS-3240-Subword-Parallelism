# CMPS-3240-Subword-Parallelism

CMPS 3240 Computer Architecture: Subword parallelism in x86-64

# Introduction

## Requirements

1. The x86 labs assume you are using the ECE/CS departments ```odin.cs.csubak.edu``` server. ```sleipnir.cs.csubak.edu``` will not work because its processor is too old. If you use your own machine, see below for verifying that you have the required instruction set.  We will be using the AVX. It was released with Sandy Bridge processors (2009) and Bulldozer processors (2008). AMD and Intel processors have a compatible ISA. However, some Macbooks (Older than Mac OS X v10.6 Snow Leopard, 2009) use PowerPC processors and won't work for this lab.
3. POSIX (Linux or Mac). x86 intrinsic function calls and header names are different between Linux and Windows.

## Prerequisities

* Indexing and storing a 2-D array as a 1-D array
* Vector element-wise multiplication
* Matrix multiplication
* Subword parallelism
* x86 intrinsics

### Verify AVX instruction set

Skip this if you are on `odin.cs.csubak.edu` or if you already used `cat /proc/cpuinfo/` to verify the AVX instruction set on your processor. ```hello_avx.c``` will test your machine for the AVX instruction set. It initializes then subtracts two 256-bit AVX registers. The following:

```c
/* Initialize the two argument vectors */
__m256 evens = _mm256_set_ps(2.0, 4.0, 6.0, 8.0, 10.0, 12.0, 14.0, 16.0);
__m256 odds = _mm256_set_ps(1.0, 3.0, 5.0, 7.0, 9.0, 11.0, 13.0, 15.0);
```

Initializes two 256-bit AVX registers by partitioning them into 8 32-bit floating point values. Note 256/32 is 8. Also note that ```evens``` vector values are the ``odds`` vector plus one, so subtracting the two should result in a vector of ones. This code:

```c
/* Compute the difference between the two vectors */
__m256 result = _mm256_sub_ps(evens, odds);
```

Carries out the subtraction. Observe that interacting with x86 intrinsics for AVX instructions requires the use of special functions that often start with ```__m256``` for the data type and ```_mm256``` prefix for functions. Compile `hello_avx` and run it from the terminal like so:

```
make hello_avx
./hello.out
```

If you get:

```
Illegal instruction (core dumped)
```

your processor does not have AVX. There is nothing that can be done. Please use ```odin.cs.csubak.edu``` instead. If it works, you should get:

```
1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000
```

and you're good to go.

### Sidebar: `pd` and `ps`

The x86 intrinsics often have either a `pd` or a `ps` suffix indicating the operation is double precision or single precision respectively. This is not required, but if you wanted a simple exercise for yourself modify `hello_avx.c` to be double precision rather than single precision.

## Objectives

* Familiarize yourself with AVX instructions
* Create a program that uses AVX instructions to implement subword parallelism
* Observe run time improvement with DGMM
* Implement improvements with DAXPY

## Background

Sub-word parallelism improves execution time of many repetitive tasks. Consider a problem where we have to add two vectors together. Each element of the vector is word length. There are registers and operations that operate on quad or greater word length. We can place four words in this over-sized register, execute an over-sized addition operation and (assuming we withheld the carry operation at the appropriate points) a single instruction can carry out four addition operations at once.

With x86 processors, the instruction set that allows you to do this is AVX. It stands for advanced vector extensions. It is also known as Sandy Bridge New Extensions, so named for the Intel chip that was the first to feature it. These instructions were designed for graphics and multimedia applications that need high precision. It reduces multiple addition steps to a single instruction, and you can harvest the appropriate word/result from within an over-sized AVX register.

For a more in depth introduction re-read sections 3.7-3.8 in Patterson and Hennesey 5e. Once you fully understand these sections, read the following document:

* https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX

For a full list of AVX instructions see:

* https://software.intel.com/sites/landingpage/IntrinsicsGuide/

# Approach

This lab consists of two parts: (1) Run the avx_dgemm.c and avx_unopt.c code and see how much faster it is with AVX. (2) Apply what you've learned with part 1 to the DAXPY operation.

## Part 1 - Understand some AVX instructions with DGMM

DGMM stands for double precision generic matrix multiplication. I will not go into a matrix multiplication here, you should be familiar with it. Suffice to say when multiplying two matrixes together, there are many, *many* addition operations. ```unopt_dgmm.c``` contains C code to multiply two matrixes together. The matrixes are initialized dynamically on the heap using ```malloc```, and the size of the square matrixes is given as a command line argument. 

### Unoptimized code

Consider the following code, which was taken from the  Patterson and Hennesey textbook:

```c
void dgemm (int n, double* A, double* B, double* C ) {
   for ( int i = 0; i < n; i++ ) {
      for ( int j = 0; j < n; j++ ) {
         double cij = C[ i + j * n ];
         // cij += A[i][k] * B[k][j]
         for ( int k = 0; k < n; k++ )
            cij += A[ i + k * n ] * B[ k + j * n ];

         C[ i + j * n ] = cij;
      }
   }
}
```

This code carries out the matrix multiplication of matrixes A and B, and stores the result in C. The cost of the matrix multiplication is  O(n^3). If you compile it, and run/time it for yourself:

```
make unopt_dgmm
time ./unopt_dgmm.out 1024
```
you should get something along the lines of:

```
Running matrix multiplication of size 1024 x 1024
real	0m5.912s
user	0m5.900s
sys	0m0.008s
```

Even on the beefy odin server it takes almost six seconds to multiply two 1024 x 1024 sized matrixes.

### AVX instructions

Now consider ```avx_dgmm.c```. The first thing to notice is:

```c
#include <immintrin.h>
```

This header file includes the compiler intrinsics for the AVX instruction set. The text calls for ```#include <x86intrin.h>``` because it is written for Windows. If on Linux, use the above. Consider the following code:

```c
void dgemm_avx (int n, double* A, double* B, double* C ) {
   for ( int i = 0; i < n; i+=4) {
      for ( int j = 0; j < n; j++ ) {
         //double cij = C[ i + j * n ];
          //replaced with
         __m256d cij = _mm256_loadu_pd(C + i + j * n);

         // Below, carries out cij += A[i][k] * B[k][j]
         for ( int k = 0; k < n; k++) {
             //cij += A[ i + k * n ] * B[ k + j * n ];
             //replaced with
             cij = _mm256_add_pd(
                 cij,
                _mm256_mul_pd(
                    _mm256_loadu_pd(A + i + k * n),
                    _mm256_broadcast_sd(B + k + j * n)
                )
            );
         }
         //C[ i + j * n ] = cij;
         //replaced with
        _mm256_storeu_pd(&C[ i + j * n ], cij);         
      }
   }
}
```

```__m256d cij``` creates a variable ```cij``` and associates it with a 256-bit double precision floating point number. ```_mm256_loadu_pd(.)``` dereferences ```C[ i + j * n ]``` and stores the result in ```cij```. ```_mm256_add_pd(cij,.)``` takes the place of ```cij +=...``` in the unoptimized code. ```_mm256_loadu_pd(.)``` loads four successive values, whereas ```_mm256_broadcast_sd(.)``` repeats the same value into many positions of the oversized register. Admittedly, this is hard to read, so you may want to just write out a four by four multiplication by hand to ensure that all the terms are there after fully expanding everything. Note that on windows, you would use ```_mm256_load_pd(.)``` in place of ```_mm256_loadu_pd(.)```. The same goes for ```store```. Go ahead and compile and time the code:

```
make avx_dgmm
time ./avx_dgmm.out 1024
```

I got the following times:

```
Running matrix multiplication of size 1024 x 1024
real	0m2.834s
user	0m2.812s
sys	0m0.020s
```

So there is nearly a two-times improvement when using AVX instructions.

## Part 2 - Implement DAXPY with and without AVX

In part 2 we will use a different operation, double-precision constant times a vector plus a vector (DAXPY):

D = a * X + Y

Where D, X and Y are vectors (not matrixes this time) of the same size, and a is a scalar. The C code looks like so:

```c
void daxpy( int m, double A, double* X, double* Y, double* result ) {
    for ( int i = 0; i < m; i++ ) {
        result[ i ] = A * X[ i ] + Y[ i ];
    }
}
```

Implement an unoptimized version of DAXPY first, then make sure it works. Some tips:

* When using `malloc` you need to allocate only `N` space rather than `N*N` space
* Note that DAXPY has three input arguments. The scalar fractional number, X and Y.

Now use AVX intrinsics to speed it up. Some tips:

* The profile for your function should look like:
```c
void daxpy( int m, const double* A, double* X, double* Y, double* result );
```
note that the `const double* A`. x86 intrinsics can broadcast a double into packed mm positions but you must pass it the pointer not the value itself.
* You can do this in one function call and a `_mm256_storeu_pd`. It should look like:
```
result <- add( multiply( scalar, load from X ), load from Y )
store( result )
```
check `avx_dgmm.c` for how to implement this.
* `_mm256_broadcast_sd( const double* value )` is the function to broadcast the same scalar double precision value into four positions in an mm register. Note that you must pass it the pointer to the value, not the value itself.
* The vectors need to be very large for you to see differences in timings. I recommend at least 2^27 = 134217728. 

You should get a modest improvement. Some results that I got:

```shell
Albert@Alberts-MacBook-Pro:~/CMPS-3240-Subword-Parallelism$ time ./unopt_daxpy.out 100000000
Running vector addition of size 100000000 x 1
real	0m2.661s
user	0m1.962s
sys	0m0.692s
Albert@Alberts-MacBook-Pro:~/CMPS-3240-Subword-Parallelism$ time ./avx_daxpy.out 100000000
Running vector addition of size 100000000 x 1
real	0m2.624s
user	0m1.920s
sys	0m0.697s
```

# Discussion

Include responses to the following questions in your lab report:

1. If you're using 256-bit sized AVX registers to hold 64-bit sized floating point numbers, what will happen to the DGMM code if N is not a multiple of 4.
2. What factor improvement did you achieve? Does this make sense? E.g., if using 256-bit sized AVX registers to hold 64-bit sized floating point numbers one would think that there should be a four fold improvement. What factors are preventing this?

# References

* `hello.avx` was taken from https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX
