This is a collection of Erlang NIF functions for BLAS operations on vectors and matrices with CUDA. Both are natively implemented as [Thrust](http://thrust.github.io/) host/device vectors and special "buffer" classes are used to transfer them from Erlang to CUDA and back. 

Installation on Windows x64 
---------------------------

``` bash
git clone git://github.com/vascokk/NumEr.git
cd NumEr
```

All the commands from this point forward should be executed in a VC++ 10.0 command-line window

``` bash
set TARGET_ARCH=x64
```

Make sure you have the following bat file:

``` bash
C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\bin\amd64\vcvars64.bat
```

with this line inside: 

``` bash
call "C:\Program Files\Microsoft SDKs\Windows\v7.1\Bin\SetEnv.cmd" /x64
```

Execute the above bat file and compile:

``` bash
mkdir priv
rebar compile
rebar eunit suites=numer_helpers_tests
```

You should see:

``` bash
==> numer (eunit)
======================== EUnit ========================
module 'numer_helpers_tests'
  numer_helpers_tests: gemm_test...[0.421 s] ok
  numer_helpers_tests: gemm_2_test...[0.375 s] ok
  numer_helpers_tests: sum_by_cols_test...[0.375 s] ok
  numer_helpers_tests: gemv_test...[0.437 s] ok
  numer_helpers_tests: gemv_2_test...[0.421 s] ok
  numer_helpers_tests: gemv_3_test...[0.405 s] ok
  numer_helpers_tests: saxpy_test...[0.390 s] ok
  numer_helpers_tests: smm_test...[0.390 s] ok
  numer_helpers_tests: m2v_test...ok
  numer_helpers_tests: v2m_test...ok
  numer_helpers_tests: transpose_test...[0.390 s] ok
  numer_helpers_tests: sigmoid_test...[0.390 s] ok
  numer_helpers_tests: sigmoid_2_test...[0.453 s] ok
  numer_helpers_tests: tanh_test...[0.390 s] ok
  numer_helpers_tests: tanh_2_test...[0.406 s] ok
  numer_helpers_tests: log_test...[0.390 s] ok
  numer_helpers_tests: log_2_test...[0.390 s] ok
  numer_helpers_tests: ones_test...ok
  numer_helpers_tests: ones_2_test...ok
  numer_helpers_tests: zeros_test...ok
  numer_helpers_tests: zeros_2_test...ok
  [done in 6.349 s]
=======================================================
  All 21 tests passed.
```

Mac OS X
--------------

Strictly follow NVIDIA Mac OS X [Getting Started](http://docs.nvidia.com/cuda/cuda-getting-started-guide-for-mac-os-x/) and set env variables:

``` bash
export PATH=/Developer/NVIDIA/CUDA-5.0/bin:$PATH
export DYLD_LIBRARY_PATH=/Developer/NVIDIA/CUDA-5.0/lib:$DYLD_LIBRARY_PATH
```

Compile and run eunit:

``` bash
mkdir priv
./rebar compile
./rebar eunit suites=numer_helpers_tests
```


TODO: Linux

Operations with vectors and matrices
------------------------------

``` erlang
% this is a row-major matrix:
A = [[4.0,6.0,8.0,2.0],[5.0,7.0,9.0,3.0]].

%this is a vector:
X = [2.0,5.0,1.0,7.0].

% create a CUDA context and transfer to "buffers"
{ok, Ctx} = numer_nifs:new_context().
{ok, Buf_A} = numer_nifs:new_matrix_float_buffer(Ctx, A, ?ROW_MAJOR).
{ok, Buf_X} = numer_nifs:new_float_buffer(Ctx).
numer_nifs:write_buffer(Buf_X, X).
```
As you see one of the parameters in the matrix buffer is "?ROW_MAJOR". It is kinda borrowed from Boost library, but not yet fully implemented in NumEr. Currently only row-major matrices are supported. However, under the hood in the Thrust vectors the numbers are stored in column-major format. I chose to do it in this way, because the CUBLAS library is using column-major storage - being a derivative of the FORTRAN BLAS library.

There are several modules, which are wrappers for the NIF functions, like: numer\_blas.erl - for BLAS operations, numer\_buffer.erl - for operations with buffers (new, delete, read, write), etc.

Using numer\_buffer module, the above example will look like:

``` erlang
 {ok, Ctx} = numer_context:new().
 {ok, Buf_A} = numer_buffer:new(Ctx, matrix, float, row_major, A).
 {ok, Buf_X} = numer_buffer:new(Ctx, float).
 numer_buffer:write(Buf_X, X).
``` 

BLAS GEMV example:
------------------

``` erlang
%  GEMV: y <- α op ( A ) x + β y
gemv_test()->
    {ok, Ctx} = numer_context:new(),
    A = [[4.0,6.0,8.0,2.0],[5.0,7.0,9.0,3.0]],
    _m = 2, %rows A
    _n = 4, %columns A
    _alpha = 1.0,
    _beta = 0.0,
    X = [2.0,5.0,1.0,7.0],
    Y = [0.0, 0.0], 
    {ok, Buf_A} = numer_buffer:new(Ctx, matrix, float, row_major, A),
    {ok, Buf_X} = numer_buffer:new(Ctx, float),
    numer_buffer:write(Buf_X, X),
    {ok, Buf_Y} = numer_buffer:new(Ctx, float),
    numer_buffer:write(Buf_Y, Y),
    ok = numer_blas:gemv(Ctx, no_transpose , _m, _n, _alpha, Buf_A, Buf_X, _beta, Buf_Y),
    {ok, [60.0,75.0]} = numer_buffer:read(Buf_Y),
    ok = numer_buffer:destroy(Buf_A),
    ok = numer_buffer:destroy(Buf_X),
    ok = numer_buffer:destroy(Buf_Y),
    ok = numer_context:destroy(Ctx).
```

Using "helpers" module
----------------------

Since using buffer operations can make the code awkward to read, there is also a helper module - numer\_helpers.erl, wich can be used for prototyping the algorithms. WARNING - do not use this module in iterative algorithms (e.g. Machine Learning). Use it for prototyping or one-off calculations with big matrices/vectors. Here is how:

``` erlang
gemv_2_test()->
    A = [[4.0,6.0,8.0,2.0],[5.0,7.0,9.0,3.0]],
    _alpha = 1.0,
    _beta = 0.0,
    X = [2.0,5.0,1.0,7.0],
    Res = numer_helpers:gemv(no_transpose , _alpha, A, X, _beta, []),
    ?assertEqual([60.0,75.0], Res).
```

It is much more readable and useful for one-off calculations, but in the ML "training" stage (with hundreds of iterations) it will be unusable, due to the multiple buffer transfers. 

Machine Learning with Erlang & CUDA - "Logistic Regression"
------------------------------------------------------------

There is an implementation of the Logistic Regression (without regularization) learning function with Gradient Descent optimization. Take a look at learn_buf() and gradient_descent() in the numer\_logreg.erl module and run the eunit test:

Windows 7, GPU - Quadro FX 1800M 1GB:

``` bash
rebar eunit suites=numer_logreg_tests tests=learn_buf2_test

NOTICE: Using experimental option 'tests'
    Running test function(s):
      numer_logreg_tests:learn_buf2_test/0
======================== EUnit ========================
numer_logreg_tests: learn_buf2_test...test/numer_logreg_tests.erl:108:<0.187.0>:
 Learned:[-0.02788488380610943,0.010618738830089569,6.68175402097404e-4]
[3.557 s] ok
=======================================================
  Test passed.
```

In learn_buf2_test() all the "buffers" needed are created upfront and passed to the NIFs in order to avoid multiple buffer creations and transfers during the iterations. 

The same test with MacBook Pro, with GeForce 9400M 256 MB:

``` bash
NOTICE: Using experimental option 'tests'
    Running test function(s):
      numer_logreg_tests:learn_buf2_test/0
======================== EUnit ========================
numer_logreg_tests: learn_buf2_test...test/numer_logreg_tests.erl:108:<0.197.0>: 
 Learned:[-0.02788488380610943,0.010618738830089569,6.68175402097404e-4]
[0.430 s] ok
=======================================================
  Test passed.
```

The numer\_ml.erl module contains a C++ implementation (via single NIF function) of Logistic Regression, while the numer\_logreg.erl is using numer_blas.erl module. The first one I used to compare the speed between the "native" and "using NumEr modules" implementations. There is considerable difference between the two on Windows (using NumEr modules - 3.5 sec, using single NIF - under 1 sec)  and almost no difference on Mac (both - under 0.5 sec).

The project is still a work in progress and needs a lot of polishing and if anyone is willing to give a hand I'll be more than happy. Any suggestions to improve the framework are also very welcome.
