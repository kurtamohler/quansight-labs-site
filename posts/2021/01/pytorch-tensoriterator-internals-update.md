<!--
.. title: PyTorch TensorIterator Internals (TensorIteratorConfig Update)
.. slug: pytorch-tensoriterator-internals-update
.. date: 2021-01-27 00:40:00 UTC-06:00
.. author: Kurt Mohler
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

For contributors to the PyTorch codebase, one of the most commonly encountered
C++ classes is
[`TensorIterator`](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/TensorIterator.h).
`TensorIterator` offers a standardized way to iterate over elements of
a tensor, automatically parallelizing operations, while abstracting device and 
data type details.

In April 2020, Sameer wrote a blog article discussing [PyTorch TensorIterator Internals](../../2020/04/pytorch-tensoriterator-internals.html). Recently,
however, the interface has changed significantly. This post describes how to
use the current interface as of January 2021. Much of the information from the
previous article is directly copied here, but with updated API calls and some
extra details.

All of the code examples below can be compiled and run in [this GitHub
repo](https://github.com/kurtamohler/pytorch-TensorIterator-examples).

# Basics of TensorIterator and TensorIteratorConfig

In order to create a `TensorIterator`, a `TensorIteratorConfig` must be created
first. `TensorIteratorConfig` specifies the input and output tensors that will
be iterated over, whether all tensors are expected to share the same data type
and device, and a handful of other settings. After setting up the
configuration, we can call `TensorIteratorConfig::build()` to obtain
a `TensorIterator` that has the specified settings. `TensorIterator` is
immutable, so once it is created, its configuration cannot be changed.

In the following example, a tensor named `out` is configured as the output
tensor and `a` and `b` are the input tensors. Calling `build` creates the 
`TensorIterator` object from the specified configuration.

``` cpp
at::TensorIteratorConfig iter_config;
iter_config
  .add_output(out)
  .add_input(a)
  .add_input(b);

auto iter = iter_config.build();
```

# Performing iterations

Iterations using `TensorIterator` can be classified as point-wise iterations or
reduction iterations. This plays a fundamental role in how iterations using
`TensorIterator` are parallelized - point-wise iterations can be freely
parallelized along any dimension and grain size while reduction operations have
to be either parallelized along dimensions that you're not iterating over or by
performing bisect and reduce operations along the dimension being iterated.
Parallelization can also happen using vectorized operations. Parallelization
with vectorized operations can also be implemented.

## Iteration details

The simplest iteration operation can be performed using the
[`for_each`](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/TensorIterator.cpp#L677)
function. This function has two overloads: one takes a function object which
iterates over a single dimension (`loop_t`); the other takes a function object
which iterates over two dimensions simultaneously (`loop2d_t`). Find their
definitions
[here](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/TensorIterator.h#L246).
The simplest way of using `for_each` is to pass it a lambda of type `loop_t`
(or `loop2d_t`).

In the example below, the `char** data` argument of the `copy_loop` function
(which is an instance of the `loop_t` lambda) contains a `char*` pointer for
each of the tensors, in the order that they are specified in the
`TensorIteratorConfig`. To make the implementation agnostic of any particular
data type, the pointer is typecast to `char`, so we can access it as an array
of bytes.

The second argument is `const int64_t* strides`, which is an array containing
the strides of each tensor in the dimension that you're iterating over. We can
add this stride to the pointer received in order to reach the next element in
the tensor. The last argument is `int64_t n` which is the size of the dimension
being iterated over.

`for_each` implicitly parallelizes the operation by executing `copy_loop` in
parallel if the number of iterations is more than the value of
`internal::GRAIN_SIZE`, which is a value that is determined as the 'right
amount' of data to iterate over in order to gain a significant speedup using
multi-threaded execution. If you want to explicitly specify that your operation
_must_ run in serial, then use the `serial_for_each` loop.

``` cpp
at::TensorIteratorConfig iter_config;
iter_config
  .add_output(out)
  .add_input(a)

  // call if output was already allocated
  .resize_outputs(false)

  // call if inputs/outputs have different types
  .check_all_same_dtype(false);

auto iter = iter_config.build();

// Copies data from input into output
auto copy_loop = [](char** data, const int64_t* strides, int64_t n) {
  auto* out_data = data[0];
  auto* in_data = data[1];

  for (int i = 0; i < n; i++) {
    // assume float data type for this example
    *reinterpret_cast<float*>(out_data) += *reinterpret_cast<float*>(in_data);
    out_data += strides[0];
    in_data += strides[1];
  }
};

iter.for_each(copy_loop);
```

### Using kernels for iterations

Frequently we want to create a kernel that applies a simple point-wise function
onto entire tensors. `TensorIterator` provides various such generic kernels
that can be used for iterating over the elements of a tensor without having to
worry about the stride, data type of the operands or details of the
parallelism.

For example, say we want to build a function that performs the point-wise
addition of two tensors and stores the result in a third tensor. We can use the
`cpu_kernel` function. Note that in this example we assume a tensor of `float`,
but you can use one of the `AT_DISPATCH_ALL_TYPES*` macros to support multiple
data types.

``` cpp
at::TensorIteratorConfig iter_config;
iter_config
  .add_output(c)
  .add_input(a)
  .add_input(b);

auto iter = iter_config.build();

// Element-wise add
at::native::cpu_kernel(iter, [] (float a, float b) -> float {
  return a + b;
});
```

Writing the kernel in this way ensures that the value returned by the lambda
passed to `cpu_kernel` will populate the corresponding position in the target
output tensor.

### Setting tensor iteration dimensions

The value of the sizes and strides will determine which dimension of the tensor
you will iterate over. `TensorIterator` performs optimizations to make sure
that at least most of the iterations happen on contiguous data to take
advantage of hierarchical cache-based memory architectures (think dimension
coalescing and reordering for maximum data locality).

A multi-dimensional tensor has a stride value for each dimension. So the stride
that `TensorIterator` needs to use will be different depending on which
dimension you want to iterate over. `TensorIterator` directly computes the
strides that get passed into the loop by itself within the `build()` function.
How exactly it computes the dimension to iterate over is something that should
be properly understood in order to use `TensorIterator` effectively.

When performing a reduction operation (see the `sum_out` code in
[ReduceOps.cpp](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/native/ReduceOps.cpp#L433)),
`TensorIterator` will figure out the dimensions that will be reduced depending
on the shape of the input and output tensor, which determines how the input
will be broadcast over the output. If you're performing a simple pointwise
operation between two tensors (like a `addcmul` from
[PointwiseOps.cpp](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/native/PointwiseOps.cpp#L31))
the iteration will happen over the entire tensor, without providing a choice of
the dimension. This allows `TensorIterator` to freely parallelize the
computation, without guaranteeing the order of execution, since it does not
matter anyway.

For something like a cumulative sum operation, where you want be able to choose
the dimension to reduce but iterate over multiple non-reduced dimensions
(possibly in parallel), you must be careful to take into account two different
strides--one for the dimension being reduced and one for all other dimensions.
Take a look at the following example of a somewhat simplified version of the
[cumsum
kernel](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/native/cpu/ReduceOpsKernel.cpp#L47).

For a 1-D input,
[torch.cumsum](https://pytorch.org/docs/stable/generated/torch.cumsum.html?highlight=cumsum#torch.cumsum)
calculates the sum of all elements from the beginning of the vector up to and
including each position in the input. A 2-D input is treated as a list of
vectors, and the cumulative sum is calculated for each vector. Higher
dimensional inputs follow the same logic--everything is just a list of 1-D
vectors.  So to implement a cumulative sum, we must take into account two
different strides: the stride between elements in a vector (`result_dim_stride`
and `self_dim_stride` in the example below) and the stride between each vector
(`strides[0]` and `strides[1]` in the example below).

```cpp
// A cumulative sum's output is the same size as the input
at::Tensor result = at::empty_like(self);

at::TensorIteratorConfig iter_config;
auto iter = iter_config
  .check_all_same_dtype(false)
  .resize_outputs(false)
  .declare_static_shape(self.sizes(), /*squash_dim=*/dim)
  .add_output(result)
  .add_input(self)
  .build();

// Size of dimension to calculate the cumulative sum across
int64_t self_dim_size = at::native::ensure_nonempty_size(self, dim);

// These strides indicate number of memory-contiguous elements, not bytes,
// between each successive element in dimension `dim`.
auto result_dim_stride = at::native::ensure_nonempty_stride(result, dim);
auto self_dim_stride = at::native::ensure_nonempty_stride(self, dim);

auto loop = [&](char** data, const int64_t* strides, int64_t n) {
  // There are `n` individual vectors that span across dimension `dim`, so
  // `n` is equal to the number of elements in `self` divided by the size of
  // dimension `dim`.

  // These are the byte strides that separate each vector that spans across
  // dimension `dim`
  auto* result_data_bytes = data[0];
  const auto* self_data_bytes = data[1];

  for (int64_t i = 0; i < n; ++i) {
    // Calculate cumulative sum for each element of the vector
    auto cumulative_sum = (at::acc_type<float, false>) 0;
    for (int64_t i = 0; i < self_dim_size; ++i) {
      const auto* self_data = reinterpret_cast<const float*>(self_data_bytes);
      auto* result_data = reinterpret_cast<float*>(result_data_bytes);
      cumulative_sum += self_data[i * self_dim_stride];
      result_data[i * result_dim_stride] = (float)cumulative_sum;
    }

    // Go to the next vector
    result_data_bytes += strides[0];
    self_data_bytes += strides[1];
  }
};

iter.for_each(loop);
```

# Conclusion

This post was a very short introduction to what `TensorIterator` is actually
capable of. If you want to learn more about how it works and what goes into
things like collapsing the tensor size for optimizing memory access, a good
place to start would be the `build()` function in
[TensorIterator.cpp](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/TensorIterator.cpp#L1255).
Also have a look at [this wiki
page](https://github.com/pytorch/pytorch/wiki/How-to-use-TensorIterator) from
the PyTorch team on using `TensorIterator.`